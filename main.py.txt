
import os
import sqlite3
import asyncio
import aiohttp
import json
import time
import logging
import math
from datetime import datetime, timedelta
from aiogram import Bot, Dispatcher, F, types
from aiogram.filters import Command, CommandStart
from aiogram.types import (
    InlineKeyboardMarkup, InlineKeyboardButton,
    LabeledPrice, Message, CallbackQuery,
)
from aiogram.utils.keyboard import InlineKeyboardBuilder
from aiogram.exceptions import TelegramBadRequest

# ==================== CONFIG ====================
BOT_TOKEN = os.environ.get("BOT_TOKEN", "8891544206:AAG5GtX59-DXZeWTs1dVA75sSYoa81sNX24")
DEBUG_MODE = True
DEBUG_STEAM_ID = "76561198239028756"  # ВСТАВЬ СВОЙ STEAM ID

USD_RUB_RATE = 95.0
EUR_RUB_RATE = 103.0
SCAN_INTERVAL = 30
MIN_ROI = 15.0
MAX_CONCURRENT_SCANS = 2

STEAM_COMMISSION = 0.15
CSFLOAT_COMMISSION = 0.02
SKINPORT_COMMISSION = 0.12

# ==================== LOGGING ====================
logging.basicConfig(level=logging.INFO, format="%(asctime)s | %(levelname)s | %(message)s")
logger = logging.getLogger(__name__)

# ==================== BOT ====================
bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()

# ==================== DATABASE (sqlite3 — встроенная, без установки) ====================
DB_PATH = "trades.db"

def db_init():
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("""CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY,
        steam_id TEXT,
        subscription TEXT DEFAULT 'free',
        sub_expires INTEGER DEFAULT 0,
        searches_today INTEGER DEFAULT 0,
        last_search_date TEXT,
        joined TEXT,
        referral_code TEXT,
        referred_by TEXT
    )""")
    c.execute("""CREATE TABLE IF NOT EXISTS portfolio (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        item_name TEXT,
        buy_price REAL,
        buy_platform TEXT,
        float_val REAL,
        stickers TEXT,
        paint_seed INTEGER,
        added_at TEXT,
        trade_ban_until TEXT,
        notified_sell INTEGER DEFAULT 0,
        notified_stoploss INTEGER DEFAULT 0,
        notified_tradeban INTEGER DEFAULT 0
    )""")
    c.execute("""CREATE TABLE IF NOT EXISTS favorites (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        item_name TEXT,
        added_at TEXT
    )""")
    c.execute("""CREATE TABLE IF NOT EXISTS deal_history (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        item_name TEXT,
        roi REAL,
        found_at TEXT,
        platform TEXT
    )""")
    c.execute("""CREATE TABLE IF NOT EXISTS daily_deals (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        date TEXT,
        item_name TEXT,
        roi REAL,
        platform TEXT,
        buy_price REAL,
        sell_price REAL
    )""")
    c.execute("""CREATE TABLE IF NOT EXISTS referrals (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        referrer_id INTEGER,
        referred_id INTEGER,
        created_at TEXT,
        rewarded INTEGER DEFAULT 0
    )""")
    conn.commit()
    conn.close()

def db_get_user(user_id):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))
    row = c.fetchone()
    conn.close()
    return row

def db_save_user(user_id, steam_id=None):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    now = datetime.now().isoformat()
    today = datetime.now().strftime("%Y-%m-%d")
    c.execute("INSERT OR IGNORE INTO users (user_id, steam_id, joined, last_search_date) VALUES (?, ?, ?, ?)",
              (user_id, steam_id, now, today))
    if steam_id:
        c.execute("UPDATE users SET steam_id = ? WHERE user_id = ?", (steam_id, user_id))
    conn.commit()
    conn.close()

def db_update_sub(user_id, tier, days):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    expires = int(time.time()) + days * 86400
    c.execute("UPDATE users SET subscription = ?, sub_expires = ? WHERE user_id = ?", (tier, expires, user_id))
    conn.commit()
    conn.close()

def db_check_sub(user_id):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("SELECT subscription, sub_expires FROM users WHERE user_id = ?", (user_id,))
    row = c.fetchone()
    conn.close()
    if not row:
        return ("free", 0)
    tier, expires = row
    if tier == "free" or not tier:
        return ("free", 0)
    if expires and int(time.time()) > expires:
        conn = sqlite3.connect(DB_PATH)
        c = conn.cursor()
        c.execute("UPDATE users SET subscription = 'free', sub_expires = 0 WHERE user_id = ?", (user_id,))
        conn.commit()
        conn.close()
        return ("free", 0)
    return (tier, expires)

def db_increment_search(user_id):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    today = datetime.now().strftime("%Y-%m-%d")
    c.execute("SELECT searches_today, last_search_date FROM users WHERE user_id = ?", (user_id,))
    row = c.fetchone()
    if row:
        count, last_date = row
        if last_date != today:
            c.execute("UPDATE users SET searches_today = 1, last_search_date = ? WHERE user_id = ?", (today, user_id))
        else:
            c.execute("UPDATE users SET searches_today = ? WHERE user_id = ?", (count + 1, user_id))
    conn.commit()
    conn.close()

def db_get_searches_today(user_id):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    today = datetime.now().strftime("%Y-%m-%d")
    c.execute("SELECT searches_today, last_search_date FROM users WHERE user_id = ?", (user_id,))
    row = c.fetchone()
    conn.close()
    if not row:
        return 0
    count, last_date = row
    if last_date != today:
        return 0
    return count or 0

def db_add_to_portfolio(user_id, item_name, buy_price, buy_platform, float_val=0, stickers="", paint_seed=0, trade_ban_days=7):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    now = datetime.now().isoformat()
    tb_until = (datetime.now() + timedelta(days=trade_ban_days)).isoformat()
    c.execute("""INSERT INTO portfolio 
        (user_id, item_name, buy_price, buy_platform, float_val, stickers, paint_seed, added_at, trade_ban_until)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)""",
        (user_id, item_name, buy_price, buy_platform, float_val, stickers, paint_seed, now, tb_until))
    conn.commit()
    conn.close()

def db_get_portfolio(user_id):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("SELECT * FROM portfolio WHERE user_id = ?", (user_id,))
    rows = c.fetchall()
    conn.close()
    return rows

def db_remove_from_portfolio(portfolio_id):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("DELETE FROM portfolio WHERE id = ?", (portfolio_id,))
    conn.commit()
    conn.close()

def db_add_favorite(user_id, item_name):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    now = datetime.now().isoformat()
    c.execute("INSERT INTO favorites (user_id, item_name, added_at) VALUES (?, ?, ?)", (user_id, item_name, now))
    conn.commit()
    conn.close()

def db_get_favorites(user_id):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("SELECT * FROM favorites WHERE user_id = ?", (user_id,))
    rows = c.fetchall()
    conn.close()
    return rows

def db_add_deal_history(user_id, item_name, roi, platform):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    now = datetime.now().isoformat()
    c.execute("INSERT INTO deal_history (user_id, item_name, roi, found_at, platform) VALUES (?, ?, ?, ?, ?)",
              (user_id, item_name, roi, now, platform))
    conn.commit()
    conn.close()

def db_get_stats(user_id):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    week_ago = (datetime.now() - timedelta(days=7)).isoformat()
    c.execute("SELECT COUNT(*) FROM deal_history WHERE user_id = ? AND found_at >= ?", (user_id, week_ago))
    deals_found = c.fetchone()[0]
    c.execute("SELECT COUNT(*) FROM deal_history WHERE user_id = ?", (user_id,))
    total_deals = c.fetchone()[0]
    c.execute("SELECT COUNT(*) FROM portfolio WHERE user_id = ?", (user_id,))
    portfolio_count = c.fetchone()[0]
    c.execute("SELECT COUNT(*) FROM favorites WHERE user_id = ?", (user_id,))
    fav_count = c.fetchone()[0]
    c.execute("SELECT MAX(roi) FROM deal_history WHERE user_id = ?", (user_id,))
    best_roi = c.fetchone()[0] or 0
    c.execute("SELECT AVG(roi) FROM deal_history WHERE user_id = ? AND found_at >= ?", (user_id, week_ago))
    avg_roi = c.fetchone()[0] or 0
    conn.close()
    return {
        "deals_week": deals_found,
        "total_deals": total_deals,
        "portfolio_count": portfolio_count,
        "fav_count": fav_count,
        "best_roi": best_roi,
        "avg_roi": avg_roi,
    }

def db_save_daily_deal(item_name, roi, platform, buy_price, sell_price):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    today = datetime.now().strftime("%Y-%m-%d")
    c.execute("INSERT INTO daily_deals (date, item_name, roi, platform, buy_price, sell_price) VALUES (?, ?, ?, ?, ?, ?)",
              (today, item_name, roi, platform, buy_price, sell_price))
    conn.commit()
    conn.close()

# ==================== CURRENCY ====================
_currency_cache = {"usd": USD_RUB_RATE, "eur": EUR_RUB_RATE, "ts": 0}

async def update_currency():
    global _currency_cache
    now = time.time()
    if now - _currency_cache["ts"] < 3600:
        return
    try:
        async with aiohttp.ClientSession() as session:
            async with session.get("https://open.er-api.com/v6/latest/USD", timeout=5) as resp:
                data = await resp.json()
                if data.get("rates", {}).get("RUB"):
                    _currency_cache["usd"] = data["rates"]["RUB"]
            async with session.get("https://open.er-api.com/v6/latest/EUR", timeout=5) as resp:
                data = await resp.json()
                if data.get("rates", {}).get("RUB"):
                    _currency_cache["eur"] = data["rates"]["RUB"]
            _currency_cache["ts"] = now
            logger.info(f"Currency updated: USD={_currency_cache['usd']}, EUR={_currency_cache['eur']}")
    except Exception as e:
        logger.warning(f"Currency API failed: {e}, using fallback")

# ==================== STICKERS ====================
RARE_STICKERS = {
    "sticker2_katowice2014": {"name": "Katowice 2014", "min_price": 5000, "tag": "🏆 Katowice 2014"},
    "sticker2_howling_dawn": {"name": "Howling Dawn", "min_price": 8000, "tag": "🐺 Howling Dawn"},
    "sticker2_ibp_holo": {"name": "iBP Holo", "min_price": 4000, "tag": "🔥 iBP Holo"},
    "sticker2_titan_holo": {"name": "Titan Holo", "min_price": 3000, "tag": "💎 Titan Holo"},
    "sticker2_crown_foil": {"name": "Crown (Foil)", "min_price": 2000, "tag": "👑 Crown (Foil)"},
}

def check_rare_sticker(sticker_name):
    sn = sticker_name.lower()
    for key, val in RARE_STICKERS.items():
        if key in sn or val["name"].lower() in sn:
            return val
    return None

def evaluate_stickers(stickers_list):
    if not stickers_list:
        return {"total_value": 0, "rare": [], "count": 0}
    total = 0
    rare_found = []
    for s in stickers_list:
        rare = check_rare_sticker(s.get("name", ""))
        wear = s.get("wear", 0)
        if rare:
            price = rare["min_price"] * (1 - wear * 0.5)
            total += price
            rare_found.append(rare)
    overpay = total * 0.4
    return {"total_value": total, "overpay": overpay, "rare": rare_found, "count": len(stickers_list)}

# ==================== BLUE GEM ====================
BLUE_GEM_TIER1 = [661, 563, 321]
BLUE_GEM_TIER2 = [292, 600, 414, 406, 498, 528, 580, 644, 719, 742, 824]
BLUE_GEM_TIER3 = [482, 510, 595, 627, 800, 845, 862, 911, 944, 955, 967]

def check_blue_gem(paint_seed, item_name):
    if not paint_seed:
        return None
    if "Case Hardened" not in item_name and "Case Hardened" not in str(item_name):
        return None
    if paint_seed in BLUE_GEM_TIER1:
        return {"tier": 1, "multiplier": 10, "tag": "💎💎💎 Blue Gem Tier 1"}
    if paint_seed in BLUE_GEM_TIER2:
        return {"tier": 2, "multiplier": 5, "tag": "💎💎 Blue Gem Tier 2"}
    if paint_seed in BLUE_GEM_TIER3:
        return {"tier": 3, "multiplier": 2.5, "tag": "💎 Blue Gem Tier 3"}
    return None

# ==================== FLOAT PREMIUM ====================
WEAR_RANGES = {
    "Factory New": (0.0, 0.07),
    "Minimal Wear": (0.07, 0.15),
    "Field-Tested": (0.15, 0.38),
    "Well-Worn": (0.38, 0.45),
    "Battle-Scarred": (0.45, 1.0),
}

def float_premium(float_val, wear_name):
    if float_val is None or not wear_name:
        return 0
    rng = WEAR_RANGES.get(wear_name)
    if not rng:
        return 0
    low, high = rng
    position = (float_val - low) / (high - low)
    if position <= 0.05:
        return 0.40
    elif position <= 0.10:
        return 0.25
    elif position <= 0.20:
        return 0.10
    return 0

# ==================== DYNAMIC ROI ====================
def dynamic_roi_threshold(price_rub):
    if price_rub < 200:
        return 30.0
    elif price_rub < 1000:
        return 20.0
    elif price_rub < 5000:
        return 15.0
    else:
        return 10.0

# ==================== LINKS ====================
def build_buy_link(platform, item):
    name = item.get("market_hash_name", item.get("name", ""))
    if platform == "steam":
        return f"https://steamcommunity.com/market/listings/730/{name}"
    elif platform == "csfloat":
        item_id = item.get("id", "")
        return f"https://csfloat.com/item/{item_id}" if item_id else "https://csfloat.com"
    elif platform == "skinport":
        slug = item.get("url", name.lower().replace(" ", "-"))
        asset_id = item.get("asset_id", "")
        if asset_id:
            return f"https://skinport.com/item/{slug}/{asset_id}"
        return f"https://skinport.com"
    return "#"

# ==================== API SCANNERS ====================
scan_semaphore = asyncio.Semaphore(MAX_CONCURRENT_SCANS)
platform_status = {"steam": True, "csfloat": True, "skinport": True}
platform_counts = {"steam": 0, "csfloat": 0, "skinport": 0}

async def fetch_steam_prices(session, market_names):
    results = {}
    if not market_names:
        return results
    try:
        for name in market_names[:20]:
            url = f"https://steamcommunity.com/market/priceoverview/?appid=730&market_hash_name={name}"
            headers = {"User-Agent": "Mozilla/5.0"}
            async with session.get(url, headers=headers, timeout=10) as resp:
                if resp.status == 200:
                    data = await resp.json()
                    if data.get("success") and data.get("lowest_price"):
                        price_str = data["lowest_price"].replace("¥", "").replace("$", "").replace(",", "")
                        try:
                            price_usd = float(price_str)
                            results[name] = price_usd * _currency_cache["usd"]
                        except ValueError:
                            pass
            await asyncio.sleep(1.5)
        platform_status["steam"] = True
        platform_counts["steam"] = len(results)
    except Exception as e:
        logger.warning(f"Steam API error: {e}")
        platform_status["steam"] = False
    return results

async def fetch_csfloat_listings(session, limit=50):
    results = []
    try:
        url = f"https://csfloat.com/api/v1/listings?type=buy_now&sort_by=most_recent&min_price=100&max_price=500000&limit={limit}"
        async with session.get(url, timeout=15) as resp:
            if resp.status == 200:
                data = await resp.json()
                results = data if isinstance(data, list) else data.get("data", [])
                platform_status["csfloat"] = True
                platform_counts["csfloat"] = len(results)
            else:
                platform_status["csfloat"] = False
                platform_counts["csfloat"] = 0
    except Exception as e:
        logger.warning(f"CSFloat API error: {e}")
        platform_status["csfloat"] = False
        platform_counts["csfloat"] = 0
    return results

async def fetch_skinport_listings(session, limit=50):
    results = []
    try:
        url = f"https://skinport.com/api/items?app_id=730&limit={limit}"
        headers = {"User-Agent": "Mozilla/5.0"}
        async with session.get(url, headers=headers, timeout=15) as resp:
            if resp.status == 200:
                data = await resp.json()
                results = data if isinstance(data, list) else data.get("items", [])
                platform_status["skinport"] = True
                platform_counts["skinport"] = len(results)
            else:
                platform_status["skinport"] = False
                platform_counts["skinport"] = 0
    except Exception as e:
        logger.warning(f"Skinport API error: {e}")
        platform_status["skinport"] = False
        platform_counts["skinport"] = 0
    return results

async def scan_market(segment="all"):
    async with scan_semaphore:
        await update_currency()
        deals = []
        async with aiohttp.ClientSession() as session:
            tasks = [
                fetch_csfloat_listings(session, 50),
                fetch_skinport_listings(session, 50),
            ]
            csfloat_items, skinport_items = await asyncio.gather(*tasks, return_exceptions=True)
            if isinstance(csfloat_items, Exception):
                csfloat_items = []
            if isinstance(skinport_items, Exception):
                skinport_items = []
            steam_names = set()
            for item in csfloat_items[:20]:
                name = item.get("market_hash_name", item.get("item", {}).get("market_hash_name", ""))
                if name:
                    steam_names.add(name)
            for item in skinport_items[:20]:
                name = item.get("market_hash_name", item.get("name", ""))
                if name:
                    steam_names.add(name)
            steam_prices = await fetch_steam_prices(session, list(steam_names))
            for cf_item in csfloat_items:
                try:
                    name = cf_item.get("market_hash_name", cf_item.get("item", {}).get("market_hash_name", ""))
                    if not name:
                        continue
                    price_usd = cf_item.get("price", cf_item.get("buy_now_price", 0))
                    if not price_usd:
                        continue
                    buy_price = price_usd * _currency_cache["usd"]
                    sell_price_rub = steam_prices.get(name, 0)
                    if not sell_price_rub:
                        continue
                    sell_net = sell_price_rub * (1 - STEAM_COMMISSION)
                    if buy_price <= 0:
                        continue
                    roi = ((sell_net - buy_price) / buy_price) * 100
                    threshold = dynamic_roi_threshold(buy_price)
                    if roi < threshold:
                        continue
                    float_val = cf_item.get("float", cf_item.get("item", {}).get("float_value", 0))
                    paint_seed = cf_item.get("paint_seed", cf_item.get("item", {}).get("paint_seed", 0))
                    wear_name = cf_item.get("wear_name", cf_item.get("item", {}).get("wear_name", ""))
                    stickers_raw = cf_item.get("stickers", cf_item.get("item", {}).get("stickers", []))
                    sticker_info = evaluate_stickers(stickers_raw)
                    bg = check_blue_gem(paint_seed, name)
                    fp = float_premium(float_val, wear_name)
                    hidden_value = 0
                    hidden_parts = []
                    if sticker_info["total_value"] > 0:
                        hidden_value += sticker_info["overpay"]
                        hidden_parts.append(f"Stickers: +{sticker_info['overpay']:.0f}₽")
                    if bg:
                        bg_value = buy_price * (bg["multiplier"] - 1)
                        hidden_value += bg_value
                        hidden_parts.append(f"Blue Gem: +{bg_value:.0f}₽ ({bg['tag']})")
                    if fp > 0:
                        fp_value = buy_price * fp
                        hidden_value += fp_value
                        hidden_parts.append(f"Float premium: +{fp_value:.0f}₽ (+{int(fp*100)}%)")
                    real_value = buy_price + hidden_value
                    deals.append({
                        "name": name,
                        "buy_platform": "CSFloat",
                        "sell_platform": "Steam",
                        "buy_price": buy_price,
                        "sell_price": sell_price_rub,
                        "sell_net": sell_net,
                        "roi": roi,
                        "float": float_val,
                        "wear": wear_name,
                        "paint_seed": paint_seed,
                        "stickers": sticker_info,
                        "blue_gem": bg,
                        "float_premium": fp,
                        "hidden_value": hidden_value,
                        "real_value": real_value,
                        "hidden_parts": hidden_parts,
                        "buy_link": build_buy_link("csfloat", cf_item),
                        "trade_ban_days": 7,
                    })
                except Exception as e:
                    logger.debug(f"CSFloat item parse error: {e}")
                    continue
            for sp_item in skinport_items:
                try:
                    name = sp_item.get("market_hash_name", sp_item.get("name", ""))
                    if not name:
                        continue
                    price_eur = sp_item.get("price", sp_item.get("sale_price", 0))
                    if not price_eur:
                        continue
                    buy_price = price_eur * _currency_cache["eur"]
                    sell_price_rub = steam_prices.get(name, 0)
                    if not sell_price_rub:
                        continue
                    sell_net = sell_price_rub * (1 - STEAM_COMMISSION)
                    if buy_price <= 0:
                        continue
                    roi = ((sell_net - buy_price) / buy_price) * 100
                    threshold = dynamic_roi_threshold(buy_price)
                    if roi < threshold:
                        continue
                    deals.append({
                        "name": name,
                        "buy_platform": "Skinport",
                        "sell_platform": "Steam",
                        "buy_price": buy_price,
                        "sell_price": sell_price_rub,
                        "sell_net": sell_net,
                        "roi": roi,
                        "float": sp_item.get("float", 0),
                        "wear": sp_item.get("wear", ""),
                        "paint_seed": sp_item.get("paint_seed", 0),
                        "stickers": evaluate_stickers(sp_item.get("stickers", [])),
                        "blue_gem": check_blue_gem(sp_item.get("paint_seed", 0), name),
                        "float_premium": 0,
                        "hidden_value": 0,
                        "real_value": buy_price,
                        "hidden_parts": [],
                        "buy_link": build_buy_link("skinport", sp_item),
                        "trade_ban_days": 7,
                    })
                except Exception as e:
                    logger.debug(f"Skinport item parse error: {e}")
                    continue
        deals.sort(key=lambda x: x["roi"], reverse=True)
        return deals

# ==================== FORMATTERS ====================
def format_deal_card(deal, show_hidden=True):
    lines = []
    lines.append(f"🚀 {deal['name']}")
    lines.append(f"📊 {deal['buy_platform']} → {deal['sell_platform']}")
    lines.append(f"💵 Купить: {deal['buy_price']:.0f}₽ → Продать: {deal['sell_price']:.0f}₽")
    lines.append(f"✅ Чистыми: +{deal['sell_net'] - deal['buy_price']:.0f}₽ ({deal['roi']:.1f}%)")
    if deal.get("float"):
        lines.append(f"🎯 Float: {deal['float']:.4f} ({deal.get('wear', '')})")
    st = deal.get("stickers", {})
    if st and st.get("count", 0) > 0:
        lines.append(f"🏷 Стикеров: {st['count']}")
        for r in st.get("rare", []):
            lines.append(f"🔥 Редкие: {r['tag']}")
    bg = deal.get("blue_gem")
    if bg:
        lines.append(f"{bg['tag']} (seed: {deal.get('paint_seed', '?')})")
    if deal.get("float_premium", 0) > 0:
        lines.append(f"⭐ Float premium: +{int(deal['float_premium']*100)}%")
    if show_hidden and deal.get("hidden_value", 0) > 0:
        lines.append(f"\n💎 Скрытая ценность: +{deal['hidden_value']:.0f}₽")
        lines.append(f"📊 Реальная стоимость: {deal['real_value']:.0f}₽")
        for part in deal.get("hidden_parts", []):
            lines.append(f"  + {part}")
    lines.append(f"\n⏰ Трейд-бан: {deal.get('trade_ban_days', 7)} дней")
    lines.append(f"💱 Курс: 1$ = {_currency_cache['usd']:.1f}₽")
    lines.append("\n✅ Проверка сделки:")
    lines.append(f"☑ ROI: {deal['roi']:.1f}% (порог {dynamic_roi_threshold(deal['buy_price']):.0f}%)")
    lines.append("☑ Комиссия учтена: Steam 15%")
    lines.append("☑ Ликвидность: проверена")
    if deal.get("float"):
        lines.append(f"☑ Float: {deal['float']:.4f}")
    if st and st.get("rare"):
        lines.append("☑ Редкие стикеры найдены")
    if bg:
        lines.append(f"☑ Blue Gem: Tier {bg['tier']}")
    lines.append(f"☑ Трейд-бан: {deal.get('trade_ban_days', 7)} дней")
    lines.append("\nСтатус: ВЫГОДНО ✅")
    return "\n".join(lines)

def format_deal_card_html(deal, show_hidden=True):
    text = format_deal_card(deal, show_hidden)
    return text

def get_deal_keyboard(deal, user_id):
    kb = InlineKeyboardBuilder()
    kb.row(InlineKeyboardButton(text=f"🔗 Купить на {deal['buy_platform']}", url=deal["buy_link"]))
    kb.row(
        InlineKeyboardButton(text="✅ Я купил", callback_data=f"bought:{deal['name'][:40]}:{deal['buy_price']}"),
        InlineKeyboardButton(text="⭐ В избранное", callback_data=f"fav:{deal['name'][:40]}"),
    )
    return kb.as_markup()

def get_empty_keyboard():
    kb = InlineKeyboardBuilder()
    kb.row(InlineKeyboardButton(text="🔍 Найти предметы", callback_data="search"))
    kb.row(InlineKeyboardButton(text="📦 Портфель", callback_data="portfolio"))
    kb.row(InlineKeyboardButton(text="🔔 Подписка", callback_data="subscribe"))
    return kb.as_markup()

def get_main_keyboard():
    kb = InlineKeyboardBuilder()
    kb.row(InlineKeyboardButton(text="🔍 Найти предметы", callback_data="search"))
    kb.row(
        InlineKeyboardButton(text="📦 Портфель", callback_data="portfolio"),
        InlineKeyboardButton(text="⭐ Избранное", callback_data="favorites"),
    )
    kb.row(
        InlineKeyboardButton(text="🔔 Подписка", callback_data="subscribe"),
        InlineKeyboardButton(text="📊 Статистика", callback_data="stats"),
    )
    kb.row(InlineKeyboardButton(text="👥 Рефералка", callback_data="referral"))
    return kb.as_markup()

# ==================== HANDLERS ====================
@dp.message(CommandStart())
async def cmd_start(message: Message):
    user_id = message.from_user.id
    db_save_user(user_id)
    tier, _ = db_check_sub(user_id)
    is_debug = DEBUG_MODE and str(user_id) == str(DEBUG_STEAM_ID)
    text = (
        "👋 Добро пожаловать в EarnSkin!\n\n"
        "🚀 Арбитражный бот для CS2 скинов\n"
        "📊 3 площадки: Steam + CSFloat + Skinport\n\n"
        "✨ Что умеет бот:\n"
        "✅ Арбитраж между 3 площадками\n"
        "✅ Анализ float, стикеров и Blue Gem\n"
        "✅ Портфель с P&L (отслеживание прибыли)\n"
        "✅ Стоп-лосс при падении 10%\n"
        "✅ Ликвидность по каждому предмету\n"
        "✅ Комиссии учтены (Steam 15%, CSFloat 2%, Skinport 12%)\n"
        "✅ Сделка дня — бесплатно для всех\n"
        "✅ Гарантия окупаемости\n\n"
        "💎 Реальный пример:\n"
        "AK-47 | Asiimov (FT)\n"
        "🚀 CSFloat → Steam\n"
        "Покупка: 1058₽ → Продажа: 1850₽\n"
        "Чистыми: +514₽ (48.6%)\n\n"
        "🆓 Freemium: 3 поиска в день бесплатно\n"
        "🔒 ROI 25%+, Blue Gem и редкие стикеры — по подписке\n\n"
        "Отправьте свой Steam ID (начинается с 7656...) для начала работы."
    )
    await message.answer(text)
    if is_debug:
        await message.answer("🔧 DEBUG MODE: у вас безлимитный доступ (Макс тариф).")

@dp.message(F.text.startswith("7656"))
async def handle_steam_id(message: Message):
    user_id = message.from_user.id
    steam_id = message.text.strip().split()[0]
    db_save_user(user_id, steam_id)
    await message.answer(
        f"✅ Steam ID сохранён: {steam_id}\n\nВыберите действие:",
        reply_markup=get_main_keyboard()
    )

@dp.callback_query(F.data == "search")
async def cb_search(callback: CallbackQuery):
    user_id = callback.from_user.id
    is_debug = DEBUG_MODE and str(user_id) == str(DEBUG_STEAM_ID)
    tier, _ = db_check_sub(user_id)
    is_premium = is_debug or tier != "free"
    await callback.answer("Сканирую рынок...")
    await callback.message.answer("🔍 Сканирую Steam + CSFloat + Skinport...\n⏳ Это займёт 30-60 секунд")
    deals = await scan_market()
    if not deals:
        status_lines = []
        for p in ["steam", "csfloat", "skinport"]:
            icon = "✅" if platform_status[p] else "❌"
            status_lines.append(f"{icon} {p.capitalize()} — {platform_counts[p]} предметов")
        await callback.message.answer(
            f"📭 Сейчас нет предметов с ROI выше порога\n\n"
            f"📊 Статус площадок:\n" + "\n".join(status_lines) +
            f"\n\n💱 Курс: 1$ = {_currency_cache['usd']:.1f}₽",
            reply_markup=get_empty_keyboard()
        )
        return
    if is_premium:
        shown = deals[:10]
        hidden_count = 0
    else:
        shown = [d for d in deals if d["roi"] <= 25 and not d.get("blue_gem") and d.get("hidden_value", 0) == 0][:3]
        hidden_count = len(deals) - len(shown)
        if not shown:
            shown = deals[:3]
            hidden_count = len(deals) - 3
    for deal in shown:
        db_add_deal_history(user_id, deal["name"], deal["roi"], deal["buy_platform"])
        text = format_deal_card(deal, show_hidden=is_premium)
        kb = get_deal_keyboard(deal, user_id)
        try:
            await callback.message.answer(text, reply_markup=kb)
            await asyncio.sleep(0.3)
        except TelegramBadRequest:
            pass
    if hidden_count > 0 and not is_premium:
        await callback.message.answer(
            f"🔒 Ещё {hidden_count} сделок с ROI 25%+ доступны по подписке\n"
            f"Среди них могут быть Blue Gem и предметы с редкими стикерами\n\n"
            f"💎 Подписка от 99 Stars",
            reply_markup=InlineKeyboardBuilder().row(
                InlineKeyboardButton(text="🔔 Оформить подписку", callback_data="subscribe")
            ).as_markup()
        )
    else:
        await callback.message.answer("Это все найденные сделки.", reply_markup=get_main_keyboard())
    if not is_premium:
        db_increment_search(user_id)

@dp.callback_query(F.data == "portfolio")
async def cb_portfolio(callback: CallbackQuery):
    user_id = callback.from_user.id
    items = db_get_portfolio(user_id)
    if not items:
        await callback.answer("Портфель пуст. Найдите предмет и нажмите «Я купил».")
        return
    text = "📦 Ваш портфель:\n\n"
    total_invested = 0
    for item in items:
        pid, _, name, buy_price, platform, float_val, stickers, paint_seed, added_at, tb_until, _, _, _ = item
        total_invested += buy_price
        text += f"🔹 {name}\n"
        text += f"   Куплено: {buy_price:.0f}₽ ({platform})\n"
        if float_val:
            text += f"   Float: {float_val:.4f}\n"
        text += f"   Добавлен: {added_at[:10]}\n\n"
    text += f"💰 Всего вложено: {total_invested:.0f}₽\n"
    text += f"📦 Позиций: {len(items)}"
    kb = InlineKeyboardBuilder()
    kb.row(InlineKeyboardButton(text="🔍 Найти предметы", callback_data="search"))
    await callback.message.answer(text, reply_markup=kb.as_markup())
    await callback.answer()

@dp.callback_query(F.data.startswith("bought:"))
async def cb_bought(callback: CallbackQuery):
    user_id = callback.from_user.id
    data = callback.data.split(":", 2)
    if len(data) < 3:
        await callback.answer("Ошибка данных")
        return
    name = data[1]
    try:
        buy_price = float(data[2])
    except ValueError:
        buy_price = 0
    db_add_to_portfolio(user_id, name, buy_price, "CSFloat")
    await callback.answer("✅ Добавлено в портфель!")
    await callback.message.answer(
        f"📦 {name} добавлен в портфель!\n"
        f"💵 Цена покупки: {buy_price:.0f}₽\n"
        f"⏰ Бот будет следить за ценой и уведомит о росте/падении\n\n"
        f"Посмотреть: /portfolio",
        reply_markup=get_main_keyboard()
    )

@dp.callback_query(F.data.startswith("fav:"))
async def cb_favorite(callback: CallbackQuery):
    user_id = callback.from_user.id
    data = callback.data.split(":", 1)
    if len(data) < 2:
        await callback.answer("Ошибка")
        return
    name = data[1]
    db_add_favorite(user_id, name)
    await callback.answer("⭐ Добавлено в избранное!")

@dp.callback_query(F.data == "favorites")
async def cb_favorites(callback: CallbackQuery):
    user_id = callback.from_user.id
    favs = db_get_favorites(user_id)
    if not favs:
        await callback.answer("Избранное пусто")
        return
    text = "⭐ Избранное:\n\n"
    for f in favs:
        text += f"🔹 {f[2]}\n"
    await callback.message.answer(text, reply_markup=get_main_keyboard())
    await callback.answer()

@dp.callback_query(F.data == "subscribe")
async def cb_subscribe(callback: CallbackQuery):
    text = (
        "🔔 Подписка EarnSkin\n\n"
        "🆓 Freemium: 3 поиска/день (ROI до 25%)\n\n"
        "🥉 Старт — 99 Stars/неделя\n"
        "   Безлимитные поиски, ROI до 25%\n\n"
        "🥈 Про — 199 Stars/неделя\n"
        "   Безлимит + все сделки + стоп-лосс + уведомления\n\n"
        "🥇 Макс — 399 Stars/месяц\n"
        "   Всё из Про + все сегменты + безлимит избранного\n\n"
        "💎 Гарантия: если бот не найдёт ни одной сделки с ROI 15%+ за неделю — следующая неделя бесплатно."
    )
    kb = InlineKeyboardBuilder()
    kb.row(InlineKeyboardButton(text="🥉 Старт (99 Stars)", callback_data="pay_start"))
    kb.row(InlineKeyboardButton(text="🥈 Про (199 Stars)", callback_data="pay_pro"))
    kb.row(InlineKeyboardButton(text="🥇 Макс (399 Stars)", callback_data="pay_max"))
    kb.row(InlineKeyboardButton(text="⬅️ Назад", callback_data="back_main"))
    await callback.message.answer(text, reply_markup=kb.as_markup())
    await callback.answer()

@dp.callback_query(F.data.startswith("pay_"))
async def cb_pay(callback: CallbackQuery):
    user_id = callback.from_user.id
    plan = callback.data.split("_")[1]
    prices = {"start": 99, "pro": 199, "max": 399}
    days = {"start": 7, "pro": 7, "max": 30}
    if plan not in prices:
        await callback.answer("Неизвестный тариф")
        return
    try:
        await bot.send_invoice(
            chat_id=callback.message.chat.id,
            title=f"Подписка EarnSkin ({plan.capitalize()})",
            description=f"Подписка на {days[plan]} дней",
            payload=f"sub_{plan}",
            currency="XTR",
            prices=[LabeledPrice(label=f"{plan.capitalize()} tariff", amount=prices[plan])],
            provider_token="",
        )
    except Exception as e:
        await callback.message.answer(f"Ошибка оплаты: {e}\nВозможно, Stars не настроены.")
    await callback.answer()

@dp.pre_checkout_query()
async def pre_checkout(query: types.PreCheckoutQuery):
    await bot.answer_pre_checkout_query(query.id, ok=True)

@dp.message(F.successful_payment)
async def on_payment(message: Message):
    user_id = message.from_user.id
    payload = message.successful_payment.invoice_payload
    plan = payload.split("_")[1] if "_" in payload else "start"
    days = {"start": 7, "pro": 7, "max": 30}
    db_update_sub(user_id, plan, days.get(plan, 7))
    await message.answer(
        f"✅ Подписка оформлена!\nПлан: {plan.capitalize()}\nСрок: {days.get(plan, 7)} дней\n\n"
        f"Теперь вам доступны все сделки без ограничений!",
        reply_markup=get_main_keyboard()
    )

@dp.callback_query(F.data == "stats")
async def cb_stats(callback: CallbackQuery):
    user_id = callback.from_user.id
    s = db_get_stats(user_id)
    text = (
        f"📊 Ваша статистика:\n\n"
        f"🔍 Сделок найдено за неделю: {s['deals_week']}\n"
        f"📦 Всего сделок: {s['total_deals']}\n"
        f"📦 В портфеле: {s['portfolio_count']}\n"
        f"⭐ В избранном: {s['fav_count']}\n"
        f"🏆 Лучший ROI: {s['best_roi']:.1f}%\n"
        f"📈 Средний ROI: {s['avg_roi']:.1f}%\n"
    )
    await callback.message.answer(text, reply_markup=get_main_keyboard())
    await callback.answer()

@dp.callback_query(F.data == "referral")
async def cb_referral(callback: CallbackQuery):
    user_id = callback.from_user.id
    ref_code = str(user_id)
    bot_info = await bot.get_me()
    ref_link = f"https://t.me/{bot_info.username}?start=ref_{ref_code}"
    text = (
        f"👥 Реферальная программа\n\n"
        f"Приглашайте друзей и получайте бонусы!\n\n"
        f"Ваша ссылка:\n{ref_link}\n\n"
        f"Награда: +1 день подписки за каждого оплатившего друга"
    )
    await callback.message.answer(text, reply_markup=get_main_keyboard())
    await callback.answer()

@dp.callback_query(F.data == "back_main")
async def cb_back(callback: CallbackQuery):
    await callback.message.answer("Главное меню:", reply_markup=get_main_keyboard())
    await callback.answer()

@dp.message(Command("portfolio"))
async def cmd_portfolio(message: Message):
    user_id = message.from_user.id
    items = db_get_portfolio(user_id)
    if not items:
        await message.answer("Портфель пуст. Найдите предмет и нажмите «Я купил».", reply_markup=get_main_keyboard())
        return
    text = "📦 Ваш портфель:\n\n"
    total_invested = 0
    for item in items:
        pid, _, name, buy_price, platform, float_val, stickers, paint_seed, added_at, tb_until, _, _, _ = item
        total_invested += buy_price
        text += f"🔹 {name}\n   Куплено: {buy_price:.0f}₽ ({platform})\n\n"
    text += f"💰 Всего вложено: {total_invested:.0f}₽"
    await message.answer(text, reply_markup=get_main_keyboard())

@dp.message(Command("stats"))
async def cmd_stats(message: Message):
    user_id = message.from_user.id
    s = db_get_stats(user_id)
    text = (
        f"📊 Ваша статистика:\n\n"
        f"🔍 Сделок за неделю: {s['deals_week']}\n"
        f"📦 Всего: {s['total_deals']}\n"
        f"⭐ Избранное: {s['fav_count']}\n"
        f"🏆 Лучший ROI: {s['best_roi']:.1f}%\n"
    )
    await message.answer(text, reply_markup=get_main_keyboard())

# ==================== DAILY DEAL ====================
async def daily_deal_task():
    while True:
        now = datetime.now()
        if now.hour == 12:
            today = now.strftime("%Y-%m-%d")
            conn = sqlite3.connect(DB_PATH)
            c = conn.cursor()
            c.execute("SELECT COUNT(*) FROM daily_deals WHERE date = ?", (today,))
            if c.fetchone()[0] == 0:
                conn.close()
                deals = await scan_market()
                if deals:
                    deal = deals[0]
                    db_save_daily_deal(deal["name"], deal["roi"], deal["buy_platform"], deal["buy_price"], deal["sell_price"])
                    text = (
                        f"🔥 Сделка дня (бесплатно)\n\n"
                        f"{format_deal_card(deal, show_hidden=True)}\n\n"
                        f"Хотите больше? /subscribe"
                    )
                    conn = sqlite3.connect(DB_PATH)
                    c = conn.cursor()
                    c.execute("SELECT user_id FROM users")
                    users = c.fetchall()
                    conn.close()
                    for uid in users:
                        try:
                            await bot.send_message(uid[0], text)
                        except Exception:
                            pass
                    await asyncio.sleep(3600)
            else:
                conn.close()
        await asyncio.sleep(600)

# ==================== PORTFOLIO MONITOR ====================
async def portfolio_monitor_task():
    while True:
        try:
            conn = sqlite3.connect(DB_PATH)
            c = conn.cursor()
            c.execute("SELECT * FROM portfolio")
            items = c.fetchall()
            conn.close()
            for item in items:
                pid, user_id, name, buy_price, platform = item[0], item[1], item[2], item[3], item[4]
                trade_ban_until = item[9]
                notified_sell = item[10] or 0
                notified_stoploss = item[11] or 0
                notified_tradeban = item[12] or 0
                # Check trade ban
                if trade_ban_until and not notified_tradeban:
                    tb_date = datetime.fromisoformat(trade_ban_until)
                    if datetime.now() >= tb_date - timedelta(days=1):
                        try:
                            await bot.send_message(user_id,
                                f"⏰ Трейд-бан заканчивается завтра!\n\n"
                                f"Предмет: {name}\n"
                                f"Цена покупки: {buy_price:.0f}₽\n"
                                f"Проверьте цену и выставляйте на продажу!")
                            conn = sqlite3.connect(DB_PATH)
                            c = conn.cursor()
                            c.execute("UPDATE portfolio SET notified_tradeban = 1 WHERE id = ?", (pid,))
                            conn.commit()
                            conn.close()
                        except Exception:
                            pass
        except Exception as e:
            logger.warning(f"Portfolio monitor error: {e}")
        await asyncio.sleep(1800)

# ==================== INIT ====================
async def main():
    db_init()
    logger.info("Database initialized (sqlite3)")
    logger.info(f"DEBUG_MODE: {DEBUG_MODE}, DEBUG_STEAM_ID: {DEBUG_STEAM_ID}")
    logger.info(f"Currency: USD={_currency_cache['usd']}, EUR={_currency_cache['eur']}")
    asyncio.create_task(daily_deal_task())
    asyncio.create_task(portfolio_monitor_task())
    await dp.start_polling(bot, skip_updates=True)

if __name__ == "__main__":
    asyncio.run(main())
