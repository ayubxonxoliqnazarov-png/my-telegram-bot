# main.py
# Talab: pip install pyTelegramBotAPI
import telebot
from telebot import types
import json
import os
import time

# ====== BU YERGA SIZNING TOKEN VA ADMIN ID QO'YING ======
TOKEN = "8556467430:AAHHZhKGOULs4qAA81kKsOuECEq75OnD8Js"
ADMIN_ID = 8106932491
# ========================================================

bot = telebot.TeleBot(TOKEN)

DATA_FILE = "bot_data.json"

# default data
default_data = {
    "users": {},   # "uid": {"coins":0,"inviter":None,"invited":0}
    "chits": [],   # list of {"title": "...", "price": 10, "desc": "...", "file_id": None, "file_type": "document" or "photo"}
    "start_count": 0,
    "channels": []  # list of "@channelname"
}

# in-memory user states to handle multi-step admin actions
user_states = {}  # uid -> {"state": "adding_chit" / "awaiting_file" / "adding_channel" / "broadcast", "tmp": {...}}

def load_data():
    if not os.path.exists(DATA_FILE):
        save_data(default_data)
    with open(DATA_FILE, "r") as f:
        return json.load(f)

def save_data(d):
    with open(DATA_FILE, "w") as f:
        json.dump(d, f, indent=2, ensure_ascii=False)

data = load_data()

# ---------- Helpers ----------
def ensure_user(uid):
    s = str(uid)
    if s not in data["users"]:
        data["users"][s] = {"coins": 0, "inviter": None, "invited": 0}
        data["start_count"] = data.get("start_count", 0) + 1
        save_data(data)
    return data["users"][s]

def is_subscribed_any(user_id):
    """Agar kanal ro'yxati bo'lsa, foydalanuvchi shu kanallardan kamida biriga obuna bo'lishi kerak."""
    chans = data.get("channels", [])
    if not chans:
        return True  # agar kanal qo'shilmagan bo'lsa — majburiy emas
    for ch in chans:
        try:
            member = bot.get_chat_member(ch, user_id)
            if member.status in ("member", "creator", "administrator"):
                return True
        except Exception:
            # chanlga kirib bo'lmasa ham davom etamiz
            pass
    return False

def main_keyboard(uid=None):
    kb = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=1)
    kb.add(
        types.KeyboardButton("📁 Chitlar"),
        types.KeyboardButton("💳 Hisobim"),
        types.KeyboardButton("💸 Chit sotib olish"),
        types.KeyboardButton("📜 Chitlar ro'yxati"),
        types.KeyboardButton("👥 Referal"),
        types.KeyboardButton("📞 Aloqa")
    )
    if uid == ADMIN_ID:
        # admin button alohida pastda
        kb.add(types.KeyboardButton("🛠 Admin panel"))
    return kb

def admin_keyboard():
    kb = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    kb.add(
        types.KeyboardButton("🔫 Chit qo'shish"),
        types.KeyboardButton("🗑 Chit o'chirish"),
        types.KeyboardButton("📁 Chitlar ro'yxati"),
        types.KeyboardButton("➕ Kanal qo'shish"),
        types.KeyboardButton("➖ Kanal o'chirish"),
        types.KeyboardButton("📣 Xabar yuborish"),
        types.KeyboardButton("📊 Statistika"),
        types.KeyboardButton("🔙 Orqaga")
    )
    return kb

def chits_inline_markup():
    kb = types.InlineKeyboardMarkup()
    for c in data.get("chits", []):
        kb.add(types.InlineKeyboardButton(f"{c['title']} — {c['price']} t", callback_data=f"buy|{c['title']}"))
    return kb

def send_chit_list(uid):
    chs = data.get("chits", [])
    if not chs:
        bot.send_message(uid, "Hozir hech qanday chit yo'q.")
        return
    txt = "📁 Chitlar ro'yxati:\n\n"
    for c in chs:
        txt += f"• {c['title']} — {c['price']} t\n"
    bot.send_message(uid, txt)

# ---------- Command handlers ----------
@bot.message_handler(commands=['start'])
def start_cmd(message):
    uid = message.from_user.id
    args = message.text.split()
    # referral
    if len(args) > 1 and args[1].isdigit():
        inviter = args[1]
        # create current user if not exists
        ensure_user(uid)
        # only give inviter coins if inviter exists and inviter != me and inviter hasn't been set as inviter for this user
        if inviter != str(uid):
            if inviter in data["users"]:
                # if this user didn't have inviter before, set it and credit inviter
                if data["users"][str(uid)].get("inviter") is None:
                    data["users"][str(uid)]["inviter"] = inviter
                    data["users"][inviter]["coins"] = data["users"][inviter].get("coins",0) + 3  # 3 tang per referral
                    data["users"][inviter]["invited"] = data["users"][inviter].get("invited",0) + 1
                    save_data(data)

    ensure_user(uid)

    # subscription check
    if not is_subscribed_any(uid):
        # if there are channels, prompt to join first one
        chans = data.get("channels", [])
        if chans:
            first = chans[0]
            kb = types.InlineKeyboardMarkup()
            kb.add(types.InlineKeyboardButton("📢 Kanalga obuna bo'lish", url=f"https://t.me/{first.replace('@','')}"))
            kb.add(types.InlineKeyboardButton("✅ Obuna bo'ldim — Tekshirish", callback_data="check_sub"))
            bot.send_message(uid, "Botdan foydalanish uchun kanalga obuna bo'ling:", reply_markup=kb)
            return

    # welcome + keyboard
    name = message.from_user.first_name or "Foydalanuvchi"
    if uid == ADMIN_ID:
        bot.send_message(uid, f"Assalomu alaykum, admin {name}!", reply_markup=admin_keyboard())
    else:
        bot.send_message(uid, f"Assalomu alaykum, {name}!\nQuyidagi tugmalardan foydalaning.", reply_markup=main_keyboard(uid))

@bot.callback_query_handler(func=lambda c: True)
def callback_query(call):
    cid = call.from_user.id
    data_local = data
    if call.data == "check_sub":
        if is_subscribed_any(cid):
            bot.answer_callback_query(call.id, "✅ Obuna tasdiqlandi.")
            bot.send_message(cid, "Obuna tasdiqlandi. /start ni qayta bosing.")
        else:
            bot.answer_callback_query(call.id, "❌ Hali obuna bo‘lmadingiz.")
        return

    if call.data.startswith("buy|"):
        title = call.data.split("|",1)[1]
        # find chit
        for c in data.get("chits", []):
            if c["title"] == title:
                user = ensure_user(cid)
                price = c["price"]
                if user["coins"] >= price:
                    user["coins"] -= price
                    save_data(data)
                    bot.answer_callback_query(call.id, f"Sotib olindi: {title}")
                    # send file if exists
                    fid = c.get("file_id")
                    ftype = c.get("file_type")
                    if fid:
                        try:
                            if ftype == "document":
                                bot.send_document(cid, fid, caption=f"Siz `{title}` chitini sotib oldingiz.", parse_mode="Markdown")
                            elif ftype == "photo":
                                bot.send_photo(cid, fid, caption=f"Siz `{title}` chitini sotib oldingiz.", parse_mode="Markdown")
                            else:
                                bot.send_message(cid, f"Siz `{title}` chitini sotib oldingiz. Fayl mavjud emas.")
                        except Exception:
                            bot.send_message(cid, f"Siz `{title}` chitini sotib oldingiz. Fayl jo'natishda xatolik yuz berdi.")
                    else:
                        bot.send_message(cid, f"Siz `{title}` chitini sotib oldingiz. Fayl hali yuklanmagan.")
                else:
                    bot.answer_callback_query(call.id, "Tangalar yetarli emas.")
                return

# ---------- Message handler (text + admin flows) ----------
@bot.message_handler(content_types=['text'])
def all_messages(message):
    uid = message.from_user.id
    txt = message.text.strip()
    ensure_user(uid)

    # If user is in a special state, handle first
    state = user_states.get(uid)
    if state:
        st = state.get("state")
        if st == "adding_chit_step":
            # expecting: Title|Price|Optional description
            parts = txt.split("|")
            if len(parts) < 2:
                bot.send_message(uid, "Xato. Masalan: SKIN HACK|20 (yoki SKIN HACK|20|Qisqacha tavsif)")
                return
            title = parts[0].strip()
            try:
                price = int(parts[1].strip())
            except:
                bot.send_message(uid, "Narxni to'g'ri kiriting (raqam).")
                return
            desc = parts[2].strip() if len(parts) > 2 else ""
            # create chit with file pending
            data.setdefault("chits", []).append({
                "title": title,
                "price": price,
                "desc": desc,
                "file_id": None,
                "file_type": None
            })
            save_data(data)
            # set state to waiting for file
            user_states[uid] = {"state": "awaiting_file", "tmp": {"title": title}}
            bot.send_message(uid, f"Chit qo'shildi: {title} — {price} t.\nIltimos, shu chitga tegishli faylni (document yoki photo) yuboring. Agar fayl yo'q bo'lsa 'NOFILE' deb yuboring.")
            return

        if st == "awaiting_file":
            # admin should send a file (document or photo) or send NOFILE
            if txt.upper() == "NOFILE":
                # leave file_id None
                user_states.pop(uid, None)
                bot.send_message(uid, "Fayl qo'yilmadi. Chit ro'yxatda, ammo fayl yo'q.")
                return
            else:
                bot.send_message(uid, "Iltimos faylni yuboring (document yoki photo). Agar yo'q bo'lsa 'NOFILE' deb yozing.")
                return

        if st == "adding_channel":
            ch = txt.strip()
            if ch not in data.get("channels", []):
                data.setdefault("channels", []).append(ch)
                save_data(data)
                user_states.pop(uid, None)
                bot.send_message(uid, f"Kanal qo'shildi: {ch}")
            else:
                user_states.pop(uid, None)
                bot.send_message(uid, "Bu kanal oldin qo'shilgan.")
            return

        if st == "remove_channel":
            ch = txt.strip()
            if ch in data.get("channels", []):
                data["channels"].remove(ch)
                save_data(data)
                user_states.pop(uid, None)
                bot.send_message(uid, f"Kanal o'chirildi: {ch}")
            else:
                user_states.pop(uid, None)
                bot.send_message(uid, "Bunday kanal topilmadi.")
            return

        if st == "broadcast":
            # send message to all users
            text_to_send = txt
            users = list(data.get("users", {}).keys())
            sent = 0
            for u in users:
                try:
                    bot.send_message(int(u), text_to_send)
                    sent += 1
                except Exception:
                    pass
            user_states.pop(uid, None)
            bot.send_message(uid, f"Xabar yuborildi: {sent} ta foydalanuvchiga.")
            return

    # Admin text commands
    if uid == ADMIN_ID:
        if txt == "🔫 Chit qo'shish":
            bot.send_message(uid, "Chit nomi va narxini kiriting. Masalan: SKIN HACK|20 yoki SKIN HACK|20|Qisqacha tavsif")
            user_states[uid] = {"state": "adding_chit_step", "tmp": {}}
            return
        if txt == "🗑 Chit o'chirish":
            if not data.get("chits"):
                bot.send_message(uid, "Hozir hech qanday chit yo'q.")
                return
            kb = types.InlineKeyboardMarkup()
            for c in data["chits"]:
                kb.add(types.InlineKeyboardButton(f"O'chirish: {c['title']}", callback_data=f"delchit|{c['title']}"))
            bot.send_message(uid, "O'chirmoqchi bo'lgan chitni tanlang:", reply_markup=kb)
            return
        if txt == "📁 Chitlar ro'yxati":
            send_chit_list(uid)
            return
        if txt == "➕ Kanal qo'shish":
            bot.send_message(uid, "Kanal usernameni kiriting (misol: @kanal_nomi):")
            user_states[uid] = {"state": "adding_channel"}
            return
        if txt == "➖ Kanal o'chirish":
            bot.send_message(uid, "O'chirmoqchi bo'lgan kanalni kiriting (misol: @kanal_nomi):")
            user_states[uid] = {"state": "remove_channel"}
            return
        if txt == "📣 Xabar yuborish":
            bot.send_message(uid, "Barcha foydalanuvchilarga jo'natiladigan xabarni yozing:")
            user_states[uid] = {"state": "broadcast"}
            return
        if txt == "📊 Statistika":
            total = len(data.get("users", {}))
            starts = data.get("start_count", 0)
            bot.send_message(uid, f"📊 Jami foydalanuvchilar: {total}\n/start bosganlar: {starts}")
            return
        if txt == "🔙 Orqaga":
            bot.send_message(uid, "Admin menyusiga qaytdingiz.", reply_markup=admin_keyboard())
            return

    # User buttons (regular)
    if txt == "📁 Chitlar" or txt == "📜 Chitlar ro'yxati":
        send_chit_list(uid)
        return

    if txt == "💸 Chit sotib olish" or txt == "Chit sotib olish":
        chs = data.get("chits", [])
        if not chs:
            bot.send_message(uid, "Hozir hech qanday chit yo'q.")
            return
        kb = chits_inline_markup()
        bot.send_message(uid, "Quyidagi tugmalardan chit tanlang:", reply_markup=kb)
        return

    if txt == "💳 Hisobim":
        u = data["users"].get(str(uid), {"coins":0,"inviter":None,"invited":0})
        inv = u.get("inviter") or "Yo'q"
        bot.send_message(uid, f"👤 Hisobim:\nID: {uid}\n💰 Tangalar: {u.get('coins',0)}\n🔗 Referrer: {inv}\n👥 Takliflar: {u.get('invited',0)}")
        return

    if txt == "👥 Referal":
        me = bot.get_me()
        link = f"https://t.me/{me.username}?start={uid}"
        bot.send_message(uid, f"👥 Do'stlaringizni taklif qiling va har bir taklif uchun *3 tanga* oling.\n\nSizning link: {link}", parse_mode="Markdown")
        return

    if txt == "📞 Aloqa":
        bot.send_message(uid, "Aloqa: @admin (yoki bot adminiga yozing).")
        return

    if txt == "🛠 Admin panel" and uid == ADMIN_ID:
        bot.send_message(uid, "🔐 Admin panel:", reply_markup=admin_keyboard())
        return

    # default fallback
    bot.send_message(uid, "Tugmalardan birini bosing yoki /start ni yuboring.", reply_markup=main_keyboard(uid))

# ---------- Document & Photo handlers (for admin file upload) ----------
@bot.message_handler(content_types=['document', 'photo'])
def handle_files(message):
    uid = message.from_user.id
    state = user_states.get(uid)
    if not state:
        # not expecting file
        bot.send_message(uid, "Hozir hech qanday fayl yuklash jarayoni yo'q.")
        return
    if state.get("state") != "awaiting_file":
        bot.send_message(uid, "Hozir fayl kutilyapti emas.")
        return

    title = state.get("tmp", {}).get("title")
    if not title:
        user_states.pop(uid, None)
        bot.send_message(uid, "Ichki xatolik: chit topilmadi.")
        return

    # find chit by title and attach file_id
    for c in data.get("chits", []):
        if c["title"] == title:
            if message.content_type == "document":
                fid = message.document.file_id
                c["file_id"] = fid
                c["file_type"] = "document"
                save_data(data)
                user_states.pop(uid, None)
                bot.send_message(uid, f"Fayl yuklandi va chitga biriktirildi: {title}")
                return
            elif message.content_type == "photo":
                # choose biggest size
                photo = message.photo[-1]
                fid = photo.file_id
                c["file_id"] = fid
                c["file_type"] = "photo"
                save_data(data)
                user_states.pop(uid, None)
                bot.send_message(uid, f"Foto yuklandi va chitga biriktirildi: {title}")
                return

    bot.send_message(uid, "Chit topilmadi yoki ichki xatolik yuz berdi.")
    user_states.pop(uid, None)

# ---------- Inline callback for deleting chits ----------
@bot.callback_query_handler(func=lambda c: c.data.startswith("delchit|"))
def del_chit_handler(call):
    uid = call.from_user.id
    if uid != ADMIN_ID:
        bot.answer_callback_query(call.id, "Faqat admin qilishi mumkin")
        return
    title = call.data.split("|",1)[1]
    before = len(data.get("chits",[]))
    data["chits"] = [c for c in data.get("chits",[]) if c["title"] != title]
    save_data(data)
    bot.answer_callback_query(call.id, f"O'chirildi: {title}")

# ---------- Run bot with reconnect ----------
def run_bot():
    while True:
        try:
            bot.infinity_polling(timeout=30, long_polling_timeout = 30)
        except Exception as e:
            print("Bot xatolik bilan to'xtadi, qayta ishga tushiriladi...", e)
            time.sleep(3)

if __name__ == "__main__":
    data = load_data()
    print("Bot ishga tushmoqda...")
    run_bot()
