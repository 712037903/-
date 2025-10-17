# bot_shop_full_with_vouchers_and_transfers.py
# متطلبات: pip install pytelegrambotapi
import telebot
from telebot import types
import sqlite3
import time
import random
from datetime import datetime

# ----------------- تكوين أساسي - عدل هذه القيم -----------------
TOKEN = "8493424289:AAHjbAqigIm23LyeT9h-J03S6vLFRv6rXN8"
DEVELOPER_ID = 8411461248  # رقمك
DEVELOPER_USERNAME = "V_7_a1"
REQUIRED_CHANNEL = "@sjsjwijn"
DAILY_GIFT_AMOUNT = 50
WHEEL_COST = 50
REFERRAL_BONUS = 100
DB_FILE = "bot_data.db"
# ---------------------------------------------------------------

bot = telebot.TeleBot(TOKEN)

# ----------------- قاعدة بيانات SQLite -------------------
conn = sqlite3.connect(DB_FILE, check_same_thread=False)
c = conn.cursor()

c.execute('''
CREATE TABLE IF NOT EXISTS users (
    user_id INTEGER PRIMARY KEY,
    username TEXT,
    balance INTEGER DEFAULT 0,
    invites INTEGER DEFAULT 0,
    last_daily INTEGER DEFAULT 0,
    wheel_uses INTEGER DEFAULT 0,
    bought_count INTEGER DEFAULT 0,
    joined_at INTEGER DEFAULT 0
)
''')

c.execute('''
CREATE TABLE IF NOT EXISTS referrals (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    referrer_id INTEGER,
    referred_id INTEGER,
    at INTEGER
)
''')

# جدول الأكواد (الفواوشر)
c.execute('''
CREATE TABLE IF NOT EXISTS vouchers (
    code TEXT PRIMARY KEY,
    uses_left INTEGER,
    amount INTEGER,
    created_by INTEGER,
    created_at INTEGER
)
''')

# جدول التحويلات (links)
c.execute('''
CREATE TABLE IF NOT EXISTS transfers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    sender_id INTEGER,
    amount INTEGER,
    fee INTEGER,
    total_deducted INTEGER,
    claimed INTEGER DEFAULT 0,
    claimed_by INTEGER,
    created_at INTEGER
)
''')

conn.commit()
# ---------------------------------------------------------------

# ----------------- دوال مساعدة -----------------
def get_user(uid):
    c.execute("SELECT user_id, username, balance, invites, last_daily, wheel_uses, bought_count, joined_at FROM users WHERE user_id=?", (uid,))
    row = c.fetchone()
    if row:
        return {
            "user_id": row[0],
            "username": row[1],
            "balance": row[2],
            "invites": row[3],
            "last_daily": row[4],
            "wheel_uses": row[5],
            "bought_count": row[6],
            "joined_at": row[7]
        }
    return None

def ensure_user(uid, username=None):
    u = get_user(uid)
    if u:
        if username and u['username'] != username:
            c.execute("UPDATE users SET username=? WHERE user_id=?", (username, uid))
            conn.commit()
        return get_user(uid)
    now = int(time.time())
    c.execute("INSERT INTO users (user_id, username, joined_at) VALUES (?,?,?)", (uid, username, now))
    conn.commit()
    return get_user(uid)

def add_balance(uid, amount):
    c.execute("UPDATE users SET balance = balance + ? WHERE user_id=?", (amount, uid))
    conn.commit()

def set_balance(uid, amount):
    c.execute("UPDATE users SET balance = ? WHERE user_id=?", (amount, uid))
    conn.commit()

def sub_balance(uid, amount):
    c.execute("UPDATE users SET balance = balance - ? WHERE user_id=?", (amount, uid))
    conn.commit()

def set_last_daily(uid, ts):
    c.execute("UPDATE users SET last_daily=? WHERE user_id=?", (ts, uid))
    conn.commit()

def inc_invites(uid, n=1):
    c.execute("UPDATE users SET invites = invites + ? WHERE user_id=?", (n, uid))
    conn.commit()

def record_referral(referrer_id, referred_id):
    now = int(time.time())
    c.execute("INSERT INTO referrals (referrer_id, referred_id, at) VALUES (?,?,?)", (referrer_id, referred_id, now))
    conn.commit()

def top_by_invites(limit=10):
    c.execute("SELECT user_id, username, invites FROM users ORDER BY invites DESC LIMIT ?", (limit,))
    return c.fetchall()

def top_by_balance(limit=10):
    c.execute("SELECT user_id, username, balance FROM users ORDER BY balance DESC LIMIT ?", (limit,))
    return c.fetchall()

# تأكد من وجود حساب المطور وأعطه 100000 لو رصيد أقل من هذا
ensure_user(DEVELOPER_ID, DEVELOPER_USERNAME)
dev = get_user(DEVELOPER_ID)
if dev and dev['balance'] < 100000:
    set_balance(DEVELOPER_ID, 100000)

# ----------------- تحقق الاشتراك في القناة -----------------
def is_subscribed(user_id):
    try:
        member = bot.get_chat_member(REQUIRED_CHANNEL, user_id)
        status = member.status
        return status not in ['left', 'kicked']
    except Exception:
        return False

# ----------------- لوحة رئيسية -- تظهر زر خاص للمطور -----------------
def main_keyboard(user_id=None):
    kb = types.ReplyKeyboardMarkup(row_width=2, resize_keyboard=True)
    kb.add(types.KeyboardButton("هدية اليوم 🎁"), types.KeyboardButton("نقاطي 🎯"))
    kb.add(types.KeyboardButton("دعوة الأصدقاء 📣"), types.KeyboardButton("سوق العروض 🛒"))
    kb.add(types.KeyboardButton("العجلة 🎡"), types.KeyboardButton("الألعاب 🎮"))
    kb.add(types.KeyboardButton("المتصدرين 🏆"), types.KeyboardButton("المهام اليومية 📝"))
    kb.add(types.KeyboardButton("المساعدة ℹ️"), types.KeyboardButton("تحويل 💸"))
    # زر إدارة/لوحة المطور يظهر فقط للمطور (أو تختاره)
    if user_id == DEVELOPER_ID:
        kb.add(types.KeyboardButton("لوحة المطور ⚙️"))
    return kb

# ----------------- حالة مؤقتة للمحادثات (جلسة بسيطة) -----------------
pending_voucher = {}  # developer_id -> step & temp data
pending_transfer = {}  # user_id -> step & data

# ----------------- /start (مُحدّث ليدعم claim links وreferrals) -----------------
@bot.message_handler(commands=['start'])
def start_handler(m):
    args = m.text.split()
    ensure_user(m.from_user.id, m.from_user.username)

    # تحقق اشتراك القناة
    if not is_subscribed(m.from_user.id):
        text = ("أهلاً! قبل استخدام البوت لازم تكون مشترك في القناة:\n"
                f"{REQUIRED_CHANNEL}\n\n"
                "بعد ما تشترك ارسل /start مرة ثانية.")
        bot.reply_to(m, text)
        return

    # إذا جاي من رابط claim transfer مثل: start=claim_transfer_12
    if len(args) > 1:
        param = args[1]
        if param.startswith("claim_transfer_"):
            try:
                tr_id = int(param.split("_")[-1])
                c.execute("SELECT id, sender_id, amount, fee, claimed FROM transfers WHERE id=?", (tr_id,))
                row = c.fetchone()
                if not row:
                    bot.reply_to(m, "هذا الرابط غير صالح أو انتهت صلاحيته.")
                else:
                    tid, sender_id, amount, fee, claimed = row
                    if claimed:
                        bot.reply_to(m, "هذا التحويل تم استلامه بالفعل.")
                    else:
                        # اعطِ المبلغ لصاحب اليوزر الذي فتح الرابط
                        add_balance(m.from_user.id, amount)
                        c.execute("UPDATE transfers SET claimed=1, claimed_by=? WHERE id=?", (m.from_user.id, tid))
                        conn.commit()
                        bot.reply_to(m, f"✅ تم استلام {amount} ريال. رصيدك الآن: {get_user(m.from_user.id)['balance']} ريال")
                        # إنذار للمرسل والمطور
                        try:
                            bot.send_message(sender_id, f"تم استلام المبلغ ({amount} ريال) من قبل @{m.from_user.username or m.from_user.first_name}")
                        except:
                            pass
                        try:
                            bot.send_message(DEVELOPER_ID, f"Transfer #{tid} claimed by {m.from_user.id} (@{m.from_user.username})")
                        except:
                            pass
                return
            except Exception:
                pass
        else:
            # محاولة referral إن كان الرقم معطى
            try:
                ref_id = int(param)
                if ref_id != m.from_user.id:
                    ref_user = get_user(ref_id)
                    if ref_user:
                        record_referral(ref_id, m.from_user.id)
                        inc_invites(ref_id, 1)
                        add_balance(ref_id, REFERRAL_BONUS)
                        bot.send_message(ref_id, f"لقد حصلت على {REFERRAL_BONUS} ريال كمكافأة لدعوة {m.from_user.first_name} (@{m.from_user.username or 'noname'})")
            except:
                pass

    kb = main_keyboard(m.from_user.id)
    text = (f"مرحبا {m.from_user.first_name} 👋\n"
            "هذي لوحة التحكّم. اختَر أي خيار من الأزرار.\n\n"
            "ملاحظة: كل عملية شراء ستُرسل تلقائياً للمطور ليتم تنفيذها.")
    bot.send_message(m.chat.id, text, reply_markup=kb)

# ----------------- زر: هدية اليوم -----------------
@bot.message_handler(func=lambda m: m.text == "هدية اليوم 🎁")
def daily_gift(m):
    user = ensure_user(m.from_user.id, m.from_user.username)
    if not is_subscribed(m.from_user.id):
        bot.reply_to(m, f"لازم تكون مشترك في القناة أولاً: {REQUIRED_CHANNEL}")
        return
    now = int(time.time())
    last = user['last_daily'] or 0
    if now - last >= 24*3600:
        add_balance(m.from_user.id, DAILY_GIFT_AMOUNT)
        set_last_daily(m.from_user.id, now)
        bot.reply_to(m, f"✅ تم إضافة {DAILY_GIFT_AMOUNT} ريال إلى رصيدك 🎉\nرصيدك الآن: {get_user(m.from_user.id)['balance']} ريال")
    else:
        remain = 24*3600 - (now - last)
        h = remain // 3600
        mnt = (remain % 3600) // 60
        bot.reply_to(m, f"سبق أخذك الهدية، جرب بعد {h} ساعة و {mnt} دقيقة.")

# ----------------- زر: نقاطي -----------------
@bot.message_handler(func=lambda m: m.text == "نقاطي 🎯")
def my_points(m):
    user = ensure_user(m.from_user.id, m.from_user.username)
    text = (f"📊 معلومات حسابك:\n"
            f"- اليوزر: @{user['username'] if user['username'] else 'لا يوجد'}\n"
            f"- الايدي: `{user['user_id']}`\n"
            f"- رصيدك: {user['balance']} ريال\n"
            f"- عدد الدعوات: {user['invites']}\n"
            f"- مرات استخدام العجلة: {user['wheel_uses']}\n"
            f"- مرات الشراء: {user['bought_count']}\n\n"
            f"رابط دعوتك: https://t.me/{bot.get_me().username}?start={user['user_id']}\n")
    bot.reply_to(m, text, parse_mode="Markdown")

# ----------------- زر: دعوة الأصدقاء -----------------
@bot.message_handler(func=lambda m: m.text == "دعوة الأصدقاء 📣")
def invites(m):
    user = ensure_user(m.from_user.id, m.from_user.username)
    text = ("ادعُ أصدقائك عبر الرابط التالي، وكل شخص يدخل منه يعطيك +100 ريال:\n\n"
            f"https://t.me/{bot.get_me().username}?start={user['user_id']}\n\n"
            "مهم: كل إحالة تُسجَّل مرة واحدة للشخص المدعو.")
    bot.reply_to(m, text)

# ----------------- زر: العجلة -----------------
@bot.message_handler(func=lambda m: m.text == "العجلة 🎡")
def wheel(m):
    user = ensure_user(m.from_user.id, m.from_user.username)
    if user['balance'] < WHEEL_COST:
        bot.reply_to(m, f"رصيدك غير كافي. تحتاج {WHEEL_COST} ريال لتجربة العجلة.\nرصيدك: {user['balance']} ريال")
        return
    kb = types.InlineKeyboardMarkup()
    kb.add(types.InlineKeyboardButton("أدفع 50 وجرب العجلة 🎡", callback_data="wheel_spin"))
    bot.reply_to(m, "تأكيد الدفع لتدوير العجلة:", reply_markup=kb)

@bot.callback_query_handler(func=lambda call: call.data == "wheel_spin")
def wheel_spin(call):
    user = ensure_user(call.from_user.id, call.from_user.username)
    if user['balance'] < WHEEL_COST:
        bot.answer_callback_query(call.id, "رصيد غير كافي", show_alert=True)
        return
    sub_balance(call.from_user.id, WHEEL_COST)
    reward = random.randint(1, 100)
    add_balance(call.from_user.id, reward)
    c.execute("UPDATE users SET wheel_uses = wheel_uses + 1 WHERE user_id=?", (call.from_user.id,))
    conn.commit()
    bot.edit_message_text(f"🎡 قمت بدفع {WHEEL_COST} ريال\n🎉 ربحت: {reward} ريال!\nرصيدك الآن: {get_user(call.from_user.id)['balance']} ريال", call.message.chat.id, call.message.message_id)
    bot.answer_callback_query(call.id, "تم تدوير العجلة")

# ----------------- سوق العروض (كما قبل) -----------------
SHOP_OFFERS = {
    "instagram": [
        {"title": "1000 متابع حقيقي (إنستا)", "price": 2700, "desc": "1000 متابع حقيقي/تسليم تدريجي"},
        {"title": "500 متابع حقيقي (إنستا)", "price": 1500, "desc": "500 متابع حقيقي"}
    ],
    "tiktok": [
        {"title": "1000 متابع تيك توك", "price": 3000, "desc": "متابعين حقيقين"},
    ],
    "youtube": [
        {"title": "1000 مشترك يوتيوب", "price": 3200, "desc": "مشتركين حقيقيين"},
    ],
    "whatsapp": [
        {"title": "خدمة رسائل واتساب", "price": 500, "desc": "حملة رسائل قصيرة"},
    ]
}

@bot.message_handler(func=lambda m: m.text == "سوق العروض 🛒")
def market(m):
    kb = types.InlineKeyboardMarkup(row_width=2)
    kb.add(types.InlineKeyboardButton("إنستغرام 📸", callback_data="market_instagram"),
           types.InlineKeyboardButton("تيك توك 🎵", callback_data="market_tiktok"))
    kb.add(types.InlineKeyboardButton("يوتيوب ▶️", callback_data="market_youtube"),
           types.InlineKeyboardButton("واتساب 💬", callback_data="market_whatsapp"))
    bot.reply_to(m, "اختر القسم:", reply_markup=kb)

def show_offers(chat_id, section_key):
    offers = SHOP_OFFERS.get(section_key, [])
    kb = types.InlineKeyboardMarkup()
    for i, o in enumerate(offers):
        kb.add(types.InlineKeyboardButton(f"{o['title']} — {o['price']} ريال", callback_data=f"buy::{section_key}::{i}"))
    kb.add(types.InlineKeyboardButton("رجوع للقائمة الرئيسية 🔙", callback_data="back_main"))
    bot.send_message(chat_id, f"عروض {section_key}:", reply_markup=kb)

@bot.callback_query_handler(func=lambda call: call.data.startswith("market_"))
def market_section(call):
    key = call.data.split("_",1)[1]
    show_offers(call.message.chat.id, key)
    bot.answer_callback_query(call.id)

@bot.callback_query_handler(func=lambda call: call.data.startswith("buy::"))
def buy_offer(call):
    parts = call.data.split("::")
    section = parts[1]
    idx = int(parts[2])
    offer = SHOP_OFFERS[section][idx]
    user = ensure_user(call.from_user.id, call.from_user.username)
    price = offer['price']
    if user['balance'] < price:
        bot.answer_callback_query(call.id, "رصيدك غير كافي لشراء العرض", show_alert=True)
        return
    sub_balance(call.from_user.id, price)
    c.execute("UPDATE users SET bought_count = bought_count + 1 WHERE user_id=?", (call.from_user.id,))
    conn.commit()
    buyer_info = f"📦 طلب جديد من @{user['username'] or 'NoUser'}\nID: {user['user_id']}\nالعرض: {offer['title']}\nالسعر: {price} ريال\nالوقت: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}"
    try:
        bot.send_message(DEVELOPER_ID, buyer_info)
    except:
        pass
    bot.edit_message_text(f"✅ تم شراء العرض: {offer['title']}\nتم خصم {price} ريال من رصيدك.\nسيتم التواصل معك حالاً من المطور @{DEVELOPER_USERNAME}", call.message.chat.id, call.message.message_id)
    bot.answer_callback_query(call.id, "تم الشراء")

# ----------------- الألعاب -----------------
GAMES_OFFERS = [
    {"title":"PUBG - 660 UC", "price": 500, "desc":"شحن شدات PUBG"},
    {"title":"Free Fire - 300 Diam", "price": 350, "desc":"شحن الماس"},
    {"title":"Clash of Clans - Gems 500", "price": 400, "desc":"شحن جوهرة كلاش"}
]

@bot.message_handler(func=lambda m: m.text == "الألعاب 🎮")
def games_section(m):
    kb = types.InlineKeyboardMarkup()
    for i, g in enumerate(GAMES_OFFERS):
        kb.add(types.InlineKeyboardButton(f"{g['title']} — {g['price']} ريال", callback_data=f"buy_game::{i}"))
    kb.add(types.InlineKeyboardButton("رجوع 🔙", callback_data="back_main"))
    bot.reply_to(m, "اختر باقة الألعاب للشحن:", reply_markup=kb)

@bot.callback_query_handler(func=lambda call: call.data.startswith("buy_game::"))
def buy_game(call):
    idx = int(call.data.split("::")[1])
    offer = GAMES_OFFERS[idx]
    user = ensure_user(call.from_user.id, call.from_user.username)
    if user['balance'] < offer['price']:
        bot.answer_callback_query(call.id, "رصيدك غير كافي", show_alert=True)
        return
    sub_balance(call.from_user.id, offer['price'])
    c.execute("UPDATE users SET bought_count = bought_count + 1 WHERE user_id=?", (call.from_user.id,))
    conn.commit()
    buyer_info = f"🎮 طلب شحن لعبة من @{user['username'] or 'NoUser'}\nID: {user['user_id']}\nالباقة: {offer['title']}\nالسعر: {offer['price']} ريال\nالوقت: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}"
    try:
        bot.send_message(DEVELOPER_ID, buyer_info)
    except:
        pass
    bot.edit_message_text(f"تم شراء {offer['title']} بنجاح ✅\nسيقوم المطور @{DEVELOPER_USERNAME} بالتواصل معك.", call.message.chat.id, call.message.message_id)
    bot.answer_callback_query(call.id)

# ----------------- المتصدرين -----------------
@bot.message_handler(func=lambda m: m.text == "المتصدرين 🏆")
def leaders(m):
    top_inv = top_by_invites(10)
    top_bal = top_by_balance(10)
    text = "🏆 المتصدرين حسب الدعوات:\n"
    for i, r in enumerate(top_inv, 1):
        text += f"{i}. @{r[1] or 'NoUser'} — {r[2]} دعوات\n"
    text += "\n💰 المتصدرين حسب الرصيد:\n"
    for i, r in enumerate(top_bal, 1):
        text += f"{i}. @{r[1] or 'NoUser'} — {r[2]} ريال\n"
    bot.reply_to(m, text)

# ----------------- المهام اليومية -----------------
DAILY_TASKS = [
    {"id":"task_invite_2", "title":"ادعِ 2 من أصحابك", "reward":200},
    {"id":"task_join_channel", "title":"اشترك في القناة الرسمية", "reward":100}
]

@bot.message_handler(func=lambda m: m.text == "المهام اليومية 📝")
def tasks_menu(m):
    kb = types.InlineKeyboardMarkup()
    for t in DAILY_TASKS:
        kb.add(types.InlineKeyboardButton(f"{t['title']} — مكافأة {t['reward']} ريال", callback_data=f"task::{t['id']}"))
    bot.reply_to(m, "اختر المهمة التي تريد تنفيذها:", reply_markup=kb)

@bot.callback_query_handler(func=lambda call: call.data.startswith("task::"))
def handle_task(call):
    tid = call.data.split("::")[1]
    user = ensure_user(call.from_user.id, call.from_user.username)
    if tid == "task_invite_2":
        if user['invites'] >= 2:
            add_balance(user['user_id'], 200)
            bot.answer_callback_query(call.id, "تم منحك 200 ريال كمكافأة", show_alert=True)
        else:
            bot.answer_callback_query(call.id, "لديك أقل من 2 دعوة حاليا", show_alert=True)
    elif tid == "task_join_channel":
        if is_subscribed(user['user_id']):
            add_balance(user['user_id'], 100)
            bot.answer_callback_query(call.id, "تم منحك 100 ريال لمتابعتك للقناة", show_alert=True)
        else:
            bot.answer_callback_query(call.id, f"لم تشترك بعد في القناة: {REQUIRED_CHANNEL}", show_alert=True)

# ----------------- المساعدة -----------------
@bot.message_handler(func=lambda m: m.text == "المساعدة ℹ️")
def help_section(m):
    text = (
        "🔹 قائمة المساعدة:\n"
        "- اضغط على هدية اليوم لتحصل على مكافئة يومية.\n"
        "- ادعُ أصدقائك عبر رابط الدعوة لتحصل على نقاط.\n"
        "- سوق العروض يحتوي على خدمات لكل مواقع التواصل ويُرسَل طلبك للمطور لتنفيذ الصفقة.\n"
        "- العجلة تجريبية وتكلف 50 ريال.\n\n"
        f"للتواصل مع المطور: @{DEVELOPER_USERNAME}\n"
        f"أو الايدي: {DEVELOPER_ID}\n"
    )
    bot.reply_to(m, text)

# ----------------- زر: تحويل (إنشاء رابط نقل) -----------------
@bot.message_handler(func=lambda m: m.text == "تحويل 💸")
def start_transfer(m):
    ensure_user(m.from_user.id, m.from_user.username)
    bot.reply_to(m, "كم المبلغ اللي تبغى تحول؟ ارسل رقم (بدون ريال).")
    pending_transfer[m.from_user.id] = {"step": "await_amount"}

@bot.message_handler(func=lambda m: m.from_user.id in pending_transfer and pending_transfer[m.from_user.id]['step']=="await_amount")
def receive_transfer_amount(m):
    try:
        amount = int(m.text.strip())
        if amount <= 0:
            bot.reply_to(m, "اكتب مبلغ صحيح أكبر من صفر.")
            return
        # حساب الرسوم 10%
        fee = amount * 10 // 100  # 10% ، عملية حسابية صحيحة للأعداد الصحيحة
        total = amount + fee
        user = ensure_user(m.from_user.id, m.from_user.username)
        if user['balance'] < total:
            bot.reply_to(m, f"رصيدك غير كافي. المطلوب: {total} ريال (المبلغ {amount} + رسوم {fee}). رصيدك: {user['balance']} ريال")
            pending_transfer.pop(m.from_user.id, None)
            return
        # احفظ في الحالة المؤقتة
        pending_transfer[m.from_user.id] = {"step":"confirm", "amount":amount, "fee":fee, "total":total}
        kb = types.InlineKeyboardMarkup()
        kb.add(types.InlineKeyboardButton(f"تأكيد التحويل {amount} ريال (يُخصم {total} ريال) ✅", callback_data="confirm_transfer"))
        kb.add(types.InlineKeyboardButton("إلغاء ❌", callback_data="cancel_transfer"))
        bot.reply_to(m, f"انت بتحول {amount} ريال. سيتم خصم {total} ريال (المبلغ + رسوم {fee} ريال).\nاضغط تأكيد لإنشاء رابط الاستلام.", reply_markup=kb)
    except ValueError:
        bot.reply_to(m, "اكتب رقم صحيح للمبلغ.")
        return

@bot.callback_query_handler(func=lambda call: call.data in ["confirm_transfer", "cancel_transfer"])
def handle_confirm_transfer(call):
    uid = call.from_user.id
    data = pending_transfer.get(uid)
    if not data:
        bot.answer_callback_query(call.id, "لا توجد عملية تحويل حالياً.", show_alert=True)
        return
    if call.data == "cancel_transfer":
        pending_transfer.pop(uid, None)
        bot.edit_message_text("تم إلغاء عملية التحويل.", call.message.chat.id, call.message.message_id)
        bot.answer_callback_query(call.id)
        return
    # تأكيد التحويل: سجل في DB واطرح المبلغ + الرسوم من المرسل
    amount = data['amount']
    fee = data['fee']
    total = data['total']
    user = ensure_user(uid, call.from_user.username)
    if user['balance'] < total:
        bot.edit_message_text("رصيدك تغير ولم يعد كافياً.", call.message.chat.id, call.message.message_id)
        pending_transfer.pop(uid, None)
        return
    sub_balance(uid, total)
    now = int(time.time())
    c.execute("INSERT INTO transfers (sender_id, amount, fee, total_deducted, created_at) VALUES (?,?,?,?,?)", (uid, amount, fee, total, now))
    conn.commit()
    tr_id = c.lastrowid
    claim_link = f"https://t.me/{bot.get_me().username}?start=claim_transfer_{tr_id}"
    bot.edit_message_text(f"✅ تم إنشاء رابط التحويل.\nأرسله للشخص المستلم ليضغط عليه ويستلم {amount} ريال.\nرابط الاستلام:\n{claim_link}", call.message.chat.id, call.message.message_id)
    # إعلام المطور بوجود تحويل جديد (اختياري)
    try:
        bot.send_message(DEVELOPER_ID, f"تحويل جديد #{tr_id} من {uid}: المبلغ {amount}, رسوم {fee}, إجمالي مخصوم {total}")
    except:
        pass
    pending_transfer.pop(uid, None)
    bot.answer_callback_query(call.id, "تم إنشاء رابط التحويل")

# ----------------- لوحة المطور (خاص للمطور فقط) -----------------
@bot.message_handler(func=lambda m: m.text == "لوحة المطور ⚙️")
def dev_panel(m):
    if m.from_user.id != DEVELOPER_ID:
        bot.reply_to(m, "هذي اللوحة خاصة بالمطور.")
        return
    kb = types.InlineKeyboardMarkup()
    kb.add(types.InlineKeyboardButton("إنشاء كود خصم 🎫", callback_data="dev_create_voucher"))
    kb.add(types.InlineKeyboardButton("قائمة الأكواد 📋", callback_data="dev_list_vouchers"))
    kb.add(types.InlineKeyboardButton("إلغاء/عودة 🔙", callback_data="back_main"))
    bot.reply_to(m, "لوحة المطور:", reply_markup=kb)

@bot.callback_query_handler(func=lambda call: call.data == "dev_create_voucher")
def dev_create_voucher(call):
    if call.from_user.id != DEVELOPER_ID:
        bot.answer_callback_query(call.id, "غير مصرح لك", show_alert=True)
        return
    pending_voucher[call.from_user.id] = {"step":"await_code"}
    bot.answer_callback_query(call.id)
    bot.send_message(call.from_user.id, "أرسل الآن **رمز الكود** الذي تريد إنشاؤه (مثل GH67).")

@bot.message_handler(func=lambda m: m.from_user.id in pending_voucher and pending_voucher[m.from_user.id]['step']=="await_code")
def voucher_receive_code(m):
    code = m.text.strip().upper()
    # تحقق إن الكود ليس موجود
    c.execute("SELECT code FROM vouchers WHERE code=?", (code,))
    if c.fetchone():
        bot.reply_to(m, "هذا الكود موجود مسبقاً، اختر رمز آخر.")
        return
    pending_voucher[m.from_user.id] = {"step":"await_uses", "code":code}
    bot.reply_to(m, "كم عدد المرات المسموح بها لهذا الكود؟ (مثل 70)")

@bot.message_handler(func=lambda m: m.from_user.id in pending_voucher and pending_voucher[m.from_user.id]['step']=="await_uses")
def voucher_receive_uses(m):
    try:
        uses = int(m.text.strip())
        if uses <= 0:
            bot.reply_to(m, "اكتب عدد صحيح أكبر من صفر.")
            return
        data = pending_voucher[m.from_user.id]
        data['uses'] = uses
        data['step'] = "await_amount"
        pending_voucher[m.from_user.id] = data
        bot.reply_to(m, "كم الريال لكل استخدام؟ (مثال 30)")
    except ValueError:
        bot.reply_to(m, "اكتب رقم صحيح لعدد المرات.")

@bot.message_handler(func=lambda m: m.from_user.id in pending_voucher and pending_voucher[m.from_user.id]['step']=="await_amount")
def voucher_receive_amount(m):
    try:
        amount = int(m.text.strip())
        if amount <= 0:
            bot.reply_to(m, "اكتب مبلغ صحيح أكبر من صفر.")
            return
        data = pending_voucher[m.from_user.id]
        code = data['code']
        uses = data['uses']
        now = int(time.time())
        c.execute("INSERT INTO vouchers (code, uses_left, amount, created_by, created_at) VALUES (?,?,?,?,?)", (code, uses, amount, m.from_user.id, now))
        conn.commit()
        bot.reply_to(m, f"✅ تم إنشاء الكود {code}\nالكمية لكل مستخدم: {amount} ريال\nعدد الاستخدامات الكلي: {uses}\nالمرات المتبقية الآن: {uses}")
        # أزل الحالة المؤقتة
        pending_voucher.pop(m.from_user.id, None)
    except ValueError:
        bot.reply_to(m, "اكتب مبلغ صحيح.")

@bot.callback_query_handler(func=lambda call: call.data == "dev_list_vouchers")
def dev_list_vouchers(call):
    if call.from_user.id != DEVELOPER_ID:
        bot.answer_callback_query(call.id, "غير مصرح لك", show_alert=True)
        return
    c.execute("SELECT code, uses_left, amount, created_at FROM vouchers")
    rows = c.fetchall()
    if not rows:
        bot.answer_callback_query(call.id, "لا توجد أكواد حالياً", show_alert=True)
        return
    text = "قائمة الأكواد:\n"
    for r in rows:
        ts = datetime.fromtimestamp(r[3]).strftime("%Y-%m-%d %H:%M")
        text += f"- {r[0]} — {r[2]} ريال لكل استخدام — متبقي: {r[1]} — أنشئ: {ts}\n"
    bot.send_message(call.from_user.id, text)
    bot.answer_callback_query(call.id)

# ----------------- استرداد الكود من أي مستخدم (رسالة نصية) -----------------
@bot.message_handler(func=lambda m: m.text and m.text.startswith("/redeem ") or (m.text and len(m.text.strip())<=20 and m.text.strip().isalpha()))
def redeem_handler(m):
    # طريقه بسيطة: المستخدم يرسل "/redeem GH67" أو يرسل رمز الكود مباشرة
    text = m.text.strip()
    if text.startswith("/redeem "):
        code = text.split(" ",1)[1].strip().upper()
    else:
        code = text.strip().upper()
    c.execute("SELECT code, uses_left, amount FROM vouchers WHERE code=?", (code,))
    row = c.fetchone()
    if not row:
        bot.reply_to(m, "الكود غير موجود أو خاطئ.")
        return
    code, uses_left, amount = row
    if uses_left <= 0:
        bot.reply_to(m, "تم نفاذ هذا الكود جميعه.")
        return
    # اعطِ المستخدم المبلغ وقلل عدد الاستخدامات
    add_balance(m.from_user.id, amount)
    c.execute("UPDATE vouchers SET uses_left = uses_left - 1 WHERE code=?", (code,))
    conn.commit()
    bot.reply_to(m, f"✅ تم تطبيق الكود {code} وحصلت على {amount} ريال. رصيدك الآن: {get_user(m.from_user.id)['balance']} ريال")
    # إخطار المطور
    try:
        bot.send_message(DEVELOPER_ID, f"تم استبدال الكود {code} من قبل @{m.from_user.username or m.from_user.first_name}")
    except:
        pass

# ----------------- زر: رجوع للقائمة الرئيسية -----------------
@bot.callback_query_handler(func=lambda call: call.data == "back_main")
def back_main(call):
    bot.edit_message_text("تم الرجوع للقائمة الرئيسية.", call.message.chat.id, call.message.message_id)
    bot.send_message(call.message.chat.id, "القائمة الرئيسية:", reply_markup=main_keyboard(call.from_user.id))
    bot.answer_callback_query(call.id)

# ----------------- رسائل افتراضية -----------------
@bot.message_handler(func=lambda m: True)
def fallback(m):
    txt = m.text.lower()
    if txt.startswith("/balance") or txt == "رصيدي":
        my_points(m); return
    if txt.startswith("/help"):
        help_section(m); return
    # لو المستخدم ضغط زر لوحة المطور لكن ليس المطور
    if txt == "لوحة المطور ⚙️" and m.from_user.id != DEVELOPER_ID:
        bot.reply_to(m, "لوحة المطور خاصة بالمطور فقط.")
        return
    bot.reply_to(m, "اختر من الأزرار الموجودة في الأسفل أو ارسل /help للمساعدة.", reply_markup=main_keyboard(m.from_user.id))

# ----------------- تشغيل البوت -----------------
if __name__ == "__main__":
    print("Bot is polling...")
    bot.infinity_polling()
