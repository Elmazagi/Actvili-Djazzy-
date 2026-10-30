import os
import json
import random
import time
import logging
import requests
from datetime import datetime, timedelta
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, MessageHandler, CallbackQueryHandler, ContextTypes, filters
from telegram.constants import ParseMode
import nest_asyncio
import asyncio
import sys

# تطبيق nest_asyncio
nest_asyncio.apply()

# إعداد التسجيل
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO,
    handlers=[
        logging.FileHandler('bot.log', encoding='utf-8'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

# بيانات البوت
BOT_TOKEN = "7039720502:AAHLfzTRWSVZOwdvD4A_mPcuTjgR3jqdV5U"
OWNER_ID = 6122326333
OWNER_USERNAME = "@k_o_b_i"

# File ID الخاص بالملصق
SUCCESS_STICKER_ID = "CAACAgQAAxkBAAIBaGmrHI7l6DMvTvl8SrBr8yCSwAK8AALIIQACqndYUQn8KkpSYjhpOgQ"

# ملفات التخزين
USERS_FILE = "users.json"
STATS_FILE = "stats.json"
REGISTERED_NUMBERS_FILE = "registered_numbers.json"

# رؤوس الطلبات من الكود الأصلي
HEADERS = {
    'User-Agent': "MobileApp/3.0.0",
    'Accept': "application/json",
    'Content-Type': "application/json",
    'accept-language': "ar",
    'Connection': "keep-alive"
}

# بيانات المستخدمين المؤقتة
user_sessions = {}
user_states = {}

# ============= دوال إدارة الملفات =============
def init_files():
    """إنشاء ملفات JSON إذا لم تكن موجودة"""
    files_data = {
        USERS_FILE: {},
        STATS_FILE: {"total_attempts": 0, "successful": 0, "failed": 0, "users": []},
        REGISTERED_NUMBERS_FILE: []
    }
    
    for file_path, default_data in files_data.items():
        if not os.path.exists(file_path):
            try:
                with open(file_path, 'w', encoding='utf-8') as f:
                    json.dump(default_data, f, ensure_ascii=False, indent=2)
                logger.info(f"✅ تم إنشاء ملف {file_path}")
            except Exception as e:
                logger.error(f"❌ خطأ في إنشاء ملف {file_path}: {e}")

def load_json(file_path, default=None):
    """تحميل ملف JSON بأمان"""
    try:
        if os.path.exists(file_path):
            with open(file_path, 'r', encoding='utf-8') as f:
                return json.load(f)
    except Exception as e:
        logger.error(f"خطأ في تحميل {file_path}: {e}")
    
    return default if default is not None else ({} if file_path == USERS_FILE else [])

def save_json(file_path, data):
    """حفظ ملف JSON بأمان"""
    try:
        with open(file_path, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
        return True
    except Exception as e:
        logger.error(f"خطأ في حفظ {file_path}: {e}")
        return False

def load_users():
    return load_json(USERS_FILE, {})

def save_users(users):
    return save_json(USERS_FILE, users)

def load_stats():
    return load_json(STATS_FILE, {"total_attempts": 0, "successful": 0, "failed": 0, "users": []})

def save_stats(stats):
    return save_json(STATS_FILE, stats)

def load_registered_numbers():
    return load_json(REGISTERED_NUMBERS_FILE, [])

def save_registered_number(number_data):
    numbers = load_registered_numbers()
    numbers.append(number_data)
    return save_json(REGISTERED_NUMBERS_FILE, numbers)

# ============= دوال MGM =============
def format_num(phone):
    phone = str(phone).strip()
    if phone.startswith('0'):
        return "213" + phone[1:]
    elif not phone.startswith('213'):
        return "213" + phone
    return phone

def generate_random_djezzy_no():
    prefix = random.choice(["077", "078", "079"])
    number = prefix + "".join([str(random.randint(0, 9)) for _ in range(7)])
    return number

def request_otp(msisdn):
    url = "https://apim.djezzy.dz/mobile-api/oauth2/registration"
    params = {
        'msisdn': msisdn,
        'client_id': "87pIExRhxBb3_wGsA5eSEfyATloa",
        'scope': "smsotp"
    }
    payload = {
        "consent-agreement": [{"marketing-notifications": False}],
        "is-consent": True
    }
    try:
        return requests.post(url, params=params, json=payload, headers=HEADERS, timeout=10)
    except Exception as e:
        logger.error(f"خطأ في طلب OTP: {e}")
        return None

def login_with_otp(mobile_number, otp):
    payload = {
        'otp': otp,
        'mobileNumber': mobile_number,
        'scope': "djezzyAppV2",
        'client_id': "87pIExRhxBb3_wGsA5eSEfyATloa",
        'client_secret': "uf82p68Bgisp8Yg1Uz8Pf6_v1XYa",
        'grant_type': "mobile"
    }
    try:
        res = requests.post(
            "https://apim.djezzy.dz/mobile-api/oauth2/token",
            data=payload,
            headers={'User-Agent': "MobileApp/3.0.0"},
            timeout=10
        )
        if res.status_code == 200:
            return f"Bearer {res.json().get('access_token')}"
        return None
    except Exception as e:
        logger.error(f"خطأ في تسجيل الدخول: {e}")
        return None

def send_invitation(token, sender, receiver):
    try:
        inv = requests.post(
            f"https://apim.djezzy.dz/mobile-api/api/v1/services/mgm/send-invitation/{sender}",
            json={"msisdnReciever": receiver},
            headers={**HEADERS, 'authorization': token},
            timeout=10
        )
        return inv.status_code in [200, 201]
    except Exception as e:
        logger.error(f"خطأ في إرسال الدعوة: {e}")
        return False

def activate_reward(token, sender):
    try:
        act = requests.post(
            f"https://apim.djezzy.dz/mobile-api/api/v1/services/mgm/activate-reward/{sender}",
            json={"packageCode": "MGMBONUS1Go"},
            headers={**HEADERS, 'authorization': token},
            timeout=10
        )
        return act.status_code in [200, 201]
    except Exception as e:
        logger.error(f"خطأ في تفعيل المكافأة: {e}")
        return False

def check_user_limits(user_id):
    """التحقق من حدود المستخدم"""
    registered = load_registered_numbers()
    user_activations = [r for r in registered if r.get('user_id') == user_id]
    
    if not user_activations:
        return True, "يمكنك التفعيل"
    
    user_activations.sort(key=lambda x: x.get('timestamp', ''), reverse=True)
    
    try:
        last_activation = datetime.strptime(user_activations[0]['timestamp'], "%Y-%m-%d %H:%M:%S")
        time_diff = datetime.now() - last_activation
        
        if time_diff < timedelta(hours=24):
            remaining = timedelta(hours=24) - time_diff
            hours = int(remaining.total_seconds() // 3600)
            minutes = int((remaining.total_seconds() % 3600) // 60)
            return False, f"⏳ يجب الانتظار {hours} ساعة و {minutes} دقيقة"
    except:
        pass
    
    one_week_ago = datetime.now() - timedelta(days=7)
    weekly_activations = 0
    
    for act in user_activations:
        try:
            act_time = datetime.strptime(act['timestamp'], "%Y-%m-%d %H:%M:%S")
            if act_time > one_week_ago:
                weekly_activations += 1
        except:
            pass
    
    if weekly_activations >= 5:
        return False, "⚠️ لقد استنفدت حصتك الأسبوعية (5 تفعيلات)"
    
    return True, "يمكنك التفعيل"

async def process_activation(user_id, phone_number, otp_code, user_name="غير معروف"):
    """معالجة عملية التفعيل"""
    stats = load_stats()
    stats["total_attempts"] = stats.get("total_attempts", 0) + 1
    save_stats(stats)
    
    can_activate, limit_reason = check_user_limits(user_id)
    if not can_activate:
        stats["failed"] = stats.get("failed", 0) + 1
        save_stats(stats)
        return False, limit_reason, None
    
    try:
        token = login_with_otp(phone_number, otp_code)
        if not token:
            stats["failed"] = stats.get("failed", 0) + 1
            save_stats(stats)
            return False, "❌ رمز التحقق غير صحيح", None
        
        for attempt in range(50):
            target = generate_random_djezzy_no()
            target_f = format_num(target)
            
            logger.info(f"محاولة {attempt + 1}: إرسال دعوة للرقم {target}")
            
            if send_invitation(token, phone_number, target_f):
                request_otp(target_f)
                await asyncio.sleep(2)
                
                if activate_reward(token, phone_number):
                    timestamp_str = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                    number_data = {
                        "user_id": user_id,
                        "user_name": user_name,
                        "sender": phone_number,
                        "target": target,
                        "timestamp": timestamp_str,
                        "status": "success"
                    }
                    save_registered_number(number_data)
                    
                    stats["successful"] = stats.get("successful", 0) + 1
                    save_stats(stats)
                    
                    return True, target, number_data
            
            await asyncio.sleep(1)
        
        stats["failed"] = stats.get("failed", 0) + 1
        save_stats(stats)
        return False, "❌ لم نتمكن من العثور على رقم صالح", None
        
    except Exception as e:
        logger.error(f"خطأ في التفعيل: {e}")
        stats["failed"] = stats.get("failed", 0) + 1
        save_stats(stats)
        return False, f"❌ حدث خطأ: {str(e)}", None

# ============= أمر البدء =============
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    user_id = user.id
    
    users = load_users()
    if str(user_id) not in users:
        users[str(user_id)] = {
            "username": user.username,
            "first_name": user.first_name,
            "joined_at": datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        }
        save_users(users)
        
        stats = load_stats()
        if "users" not in stats:
            stats["users"] = []
        if user_id not in stats["users"]:
            stats["users"].append(user_id)
        save_stats(stats)
    
    welcome_text = (
        f"🎉 *مرحباً بك يا {user.first_name}* 🎉\n\n"
        f"🤍 *بوت تفعيل 1GB من جيزي* 🤍\n"
        f"👑 *المطور*: {OWNER_USERNAME}\n\n"
        f"📌 *للاستخدام:*\n"
        f"1️⃣ اضغط على زر التفعيل\n"
        f"2️⃣ أرسل رقم جيزي\n"
        f"3️⃣ أرسل رمز التحقق\n"
        f"4️⃣ استلم 1GB هدية 🎁\n\n"
        f"⚠️ *ملاحظة:* يمكنك التفعيل مرة كل 24 ساعة\n"
        f"⚠️ *الحد الأقصى:* 5 مرات في الأسبوع\n\n"
        f"🤲 *لا تستعمل الانترنت فيما لا يرضي الله*\n\n"
        f"👇 *اختر من الأزرار أدناه*"
    )
    
    keyboard = [
        [InlineKeyboardButton("🚀 تفعيل 1GB", callback_data="activate")]
    ]
    
    if user_id == OWNER_ID:
        keyboard.append([
            InlineKeyboardButton("📊 الإحصائيات", callback_data="stats"),
            InlineKeyboardButton("⚙️ التحكم", callback_data="admin_panel")
        ])
    
    await update.message.reply_text(
        welcome_text,
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode=ParseMode.MARKDOWN
    )

# ============= معالج الأزرار =============
async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    user_id = query.from_user.id
    data = query.data
    
    # التعامل مع زر معلومات الأدمن الشفاف (معرف بـ admin_info_)
    if data.startswith("admin_info_"):
        if user_id != OWNER_ID:
            await query.answer("❌ هذه الميزة للمالك فقط", show_alert=True)
            return
        
        idx = int(data.split("_")[-1])
        registered = load_registered_numbers()
        
        if idx < len(registered):
            info = registered[idx]
            info_text = (
                f"👤 المستخدم: {info.get('user_name', 'غير معروف')}\n"
                f"🆔 الـ ID: {info.get('user_id')}\n"
                f"📱 رقم الهاتف (المرسل): {info.get('sender')}\n"
                f"🎯 الرقم المستلم: {info.get('target')}\n"
                f"⏰ الوقت: {info.get('timestamp')}"
            )
            await query.answer(info_text, show_alert=True)
        else:
            await query.answer("❌ لم يتم العثور على البيانات", show_alert=True)
        return

    await query.answer()
    
    if data == "activate":
        keyboard = [[InlineKeyboardButton("❌ إلغاء", callback_data="cancel")]]
        
        await query.edit_message_text(
            "📱 *أرسل رقم جيزي الخاص بك*\n\n"
            "📌 *مثال:* `0770123456`\n\n"
            "⚠️ *تأكد من صحة الرقم*\n"
            "❌ *للإلغاء اضغط الزر أدناه*",
            reply_markup=InlineKeyboardMarkup(keyboard),
            parse_mode=ParseMode.MARKDOWN
        )
        user_states[user_id] = "waiting_phone"
    
    elif data == "stats":
        if user_id != OWNER_ID:
            await query.edit_message_text("❌ *هذه الميزة للمالك فقط*", parse_mode=ParseMode.MARKDOWN)
            return
        
        stats = load_stats()
        registered = load_registered_numbers()
        users = load_users()
        
        total = stats.get('total_attempts', 0)
        success = stats.get('successful', 0)
        failed = stats.get('failed', 0)
        success_rate = (success / total * 100) if total > 0 else 0
        
        stats_text = (
            f"*📊 إحصائيات البوت*\n\n"
            f"*👥 المستخدمين:*\n"
            f"• الإجمالي: {len(users)}\n"
            f"• النشطين: {len(stats.get('users', []))}\n\n"
            f"*📱 عمليات التفعيل:*\n"
            f"• الإجمالي: {total}\n"
            f"• ✅ ناجحة: {success}\n"
            f"• ❌ فاشلة: {failed}\n"
            f"• 📈 نسبة النجاح: {success_rate:.1f}%\n\n"
            f"*💾 إجمالي المسجل:* {len(registered)}"
        )
        
        keyboard = [[InlineKeyboardButton("🔙 رجوع", callback_data="back_to_main")]]
        await query.edit_message_text(stats_text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode=ParseMode.MARKDOWN)
    
    elif data == "admin_panel":
        if user_id != OWNER_ID:
            return
        
        keyboard = [
            [InlineKeyboardButton("📨 رسالة جماعية", callback_data="broadcast")],
            [InlineKeyboardButton("📊 إحصائيات متقدمة", callback_data="advanced_stats")],
            [InlineKeyboardButton("🔙 رجوع", callback_data="back_to_main")]
        ]
        
        await query.edit_message_text(
            "*⚙️ لوحة التحكم*\n\nاختر ما تريد:",
            reply_markup=InlineKeyboardMarkup(keyboard),
            parse_mode=ParseMode.MARKDOWN
        )
    
    elif data == "broadcast":
        if user_id != OWNER_ID:
            return
        
        await query.edit_message_text(
            "*📨 رسالة جماعية*\n\n"
            "أرسل الرسالة التي تريد نشرها:\n\n"
            "⚠️ *قد تستغرق العملية وقتاً*",
            parse_mode=ParseMode.MARKDOWN
        )
        user_states[user_id] = "waiting_broadcast"
    
    elif data == "advanced_stats":
        if user_id != OWNER_ID:
            return
        
        registered = load_registered_numbers()
        users = load_users()
        
        today = datetime.now().strftime("%Y-%m-%d")
        today_count = sum(1 for r in registered if r.get('timestamp', '').startswith(today))
        
        user_counts = {}
        for r in registered:
            uid = r.get('user_id')
            if uid:
                user_counts[uid] = user_counts.get(uid, 0) + 1
        
        top_users = sorted(user_counts.items(), key=lambda x: x[1], reverse=True)[:5]
        
        text = (
            f"*📊 إحصائيات متقدمة*\n\n"
            f"*📅 عمليات اليوم:* {today_count}\n\n"
            f"*👑 أكثر المستخدمين تفعيلاً:*\n"
        )
        
        for i, (uid, count) in enumerate(top_users, 1):
            user_info = users.get(str(uid), {})
            name = user_info.get('first_name', 'غير معروف')
            text += f"{i}. {name} - {count} مرات\n"
        
        text += f"\n*📱 آخر 10 عمليات:*\n"
        for r in registered[-10:]:
            text += f"• {r.get('sender')} → {r.get('target')}\n"
        
        keyboard = [[InlineKeyboardButton("🔙 رجوع", callback_data="admin_panel")]]
        await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode=ParseMode.MARKDOWN)
    
    elif data == "cancel":
        if user_id in user_states:
            del user_states[user_id]
        if user_id in user_sessions:
            del user_sessions[user_id]
        
        keyboard = [
            [InlineKeyboardButton("🚀 تفعيل 1GB", callback_data="activate")]
        ]
        
        if user_id == OWNER_ID:
            keyboard.append([
                InlineKeyboardButton("📊 الإحصائيات", callback_data="stats"),
                InlineKeyboardButton("⚙️ التحكم", callback_data="admin_panel")
            ])
        
        await query.edit_message_text(
            "✅ *تم الإلغاء*\n\nاختر من الأزرار أدناه:",
            reply_markup=InlineKeyboardMarkup(keyboard),
            parse_mode=ParseMode.MARKDOWN
        )
    
    elif data == "back_to_main":
        keyboard = [
            [InlineKeyboardButton("🚀 تفعيل 1GB", callback_data="activate")]
        ]
        
        if user_id == OWNER_ID:
            keyboard.append([
                InlineKeyboardButton("📊 الإحصائيات", callback_data="stats"),
                InlineKeyboardButton("⚙️ التحكم", callback_data="admin_panel")
            ])
        
        await query.edit_message_text(
            "*مرحباً بعودتك*\n\nاختر من الأزرار أدناه:",
            reply_markup=InlineKeyboardMarkup(keyboard),
            parse_mode=ParseMode.MARKDOWN
        )

# ============= معالج الرسائل =============
async def message_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    user_id = user.id
    text = update.message.text
    
    if user_id not in user_states:
        await update.message.reply_text("❌ أرسل /start للبدء")
        return
    
    state = user_states[user_id]
    
    if state == "waiting_phone":
        phone = text.strip().replace(' ', '').replace('+', '')
        
        if not phone.isdigit() or len(phone) < 9:
            keyboard = [[InlineKeyboardButton("❌ إلغاء", callback_data="cancel")]]
            await update.message.reply_text(
                "❌ *رقم غير صحيح*\n\n"
                "الرجاء إرسال رقم صحيح\n"
                "مثال: `0770123456`",
                reply_markup=InlineKeyboardMarkup(keyboard),
                parse_mode=ParseMode.MARKDOWN
            )
            return
        
        user_sessions[user_id] = {"phone": phone}
        user_states[user_id] = "waiting_otp"
        
        phone_f = format_num(phone)
        await update.message.reply_text("🔄 *جاري إرسال رمز التحقق...*", parse_mode=ParseMode.MARKDOWN)
        
        otp_response = request_otp(phone_f)
        
        if otp_response and otp_response.status_code in [200, 201]:
            keyboard = [[InlineKeyboardButton("❌ إلغاء", callback_data="cancel")]]
            await update.message.reply_text(
                "✅ *تم إرسال رمز التحقق*\n\n"
                "📨 أرسل الرمز المكون من 6 أرقام:",
                reply_markup=InlineKeyboardMarkup(keyboard),
                parse_mode=ParseMode.MARKDOWN
            )
        else:
            del user_states[user_id]
            del user_sessions[user_id]
            keyboard = [[InlineKeyboardButton("🔄 محاولة جديدة", callback_data="activate")]]
            await update.message.reply_text(
                "❌ *فشل إرسال رمز التحقق*\n\n"
                "تحقق من الرقم وحاول مرة أخرى",
                reply_markup=InlineKeyboardMarkup(keyboard),
                parse_mode=ParseMode.MARKDOWN
            )
    
    elif state == "waiting_otp":
        otp = text.strip()
        
        if not otp.isdigit() or len(otp) != 6:
            keyboard = [[InlineKeyboardButton("❌ إلغاء", callback_data="cancel")]]
            await update.message.reply_text(
                "❌ *رمز غير صحيح*\n\n"
                "الرجاء إرسال 6 أرقام فقط",
                reply_markup=InlineKeyboardMarkup(keyboard),
                parse_mode=ParseMode.MARKDOWN
            )
            return
        
        phone = user_sessions.get(user_id, {}).get("phone")
        if not phone:
            del user_states[user_id]
            await update.message.reply_text("❌ حدث خطأ، أرسل /start")
            return
        
        processing_msg = await update.message.reply_text(
            "⏳ *جاري إرسال الانترنت...* 📲\n\n"
            "الرجاء الانتظار",
            parse_mode=ParseMode.MARKDOWN
        )
        
        success, result, number_data = await process_activation(user_id, format_num(phone), otp, user.first_name)
        
        await processing_msg.delete()
        
        if success:
            try:
                await update.message.reply_sticker(sticker=SUCCESS_STICKER_ID)
            except:
                pass
            
            keyboard = []
            
            # زر شفاف خاص بالأدمن تحت الرسالة يعرض رقم الهاتف كاملاً مع المعلومات
            if user_id == OWNER_ID:
                registered = load_registered_numbers()
                idx = len(registered) - 1
                keyboard.append([InlineKeyboardButton("🔍 تفاصيل المستخدم (أدمن)", callback_data=f"admin_info_{idx}")])
            
            await update.message.reply_text(
                f"🎉 *تم الإرسال بنجاح* 🥳\n\n"
                f"✅ تم تفعيل 1GB\n\n"
                f"🤲 *لا تستعمل الانترنت فيما لا يرضي الله*",
                reply_markup=InlineKeyboardMarkup(keyboard) if keyboard else None,
                parse_mode=ParseMode.MARKDOWN
            )
        else:
            keyboard = [
                [InlineKeyboardButton("🔄 محاولة جديدة", callback_data="activate")]
            ]
            
            await update.message.reply_text(
                f"❌ *فشل التفعيل*\n\n{result}",
                reply_markup=InlineKeyboardMarkup(keyboard),
                parse_mode=ParseMode.MARKDOWN
            )
        
        if user_id in user_states:
            del user_states[user_id]
        if user_id in user_sessions:
            del user_sessions[user_id]
    
    elif state == "waiting_broadcast" and user_id == OWNER_ID:
        await update.message.reply_text("📨 *جاري النشر...*", parse_mode=ParseMode.MARKDOWN)
        
        users = load_users()
        success = 0
        failed = 0
        
        for uid_str in users.keys():
            try:
                await context.bot.send_message(
                    chat_id=int(uid_str),
                    text=f"*📢 رسالة من المطور*\n\n{text}",
                    parse_mode=ParseMode.MARKDOWN
                )
                success += 1
            except:
                failed += 1
            
            await asyncio.sleep(0.05)
        
        await update.message.reply_text(
            f"*✅ تم النشر*\n\n"
            f"✅ نجح: {success}\n"
            f"❌ فشل: {failed}\n"
            f"👥 الإجمالي: {len(users)}",
            parse_mode=ParseMode.MARKDOWN
        )
        
        del user_states[user_id]

# ============= التشغيل الرئيسي =============
def main():
    print("=" * 60)
    print("           🚀 بوت تفعيل 1GB من جيزي")
    print("=" * 60)
    print(f"👤 المالك: {OWNER_ID}")
    print("=" * 60)
    
    init_files()
    
    application = Application.builder().token(BOT_TOKEN).build()
    
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CallbackQueryHandler(button_handler))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, message_handler))
    
    print("✅ البوت يعمل...")
    print("=" * 60)
    
    try:
        application.run_polling(allowed_updates=Update.ALL_TYPES)
    except KeyboardInterrupt:
        print("\n👋 تم إيقاف البوت")
    except Exception as e:
        logger.error(f"خطأ: {e}")
        print(f"❌ خطأ: {e}")

if __name__ == "__main__":
    main()
