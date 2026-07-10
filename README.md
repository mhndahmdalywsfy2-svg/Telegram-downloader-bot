import os
import asyncio
import yt_dlp
from aiogram import Bot, Dispatcher, F
from aiogram.types import Message, InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.filters import CommandStart

API_TOKEN = os.getenv("API_TOKEN")
CHANNEL_ID = "@ulxath"  # قناة الاشتراك الإجباري

bot = Bot(token=API_TOKEN)
dp = Dispatcher()

# 1. دالة التحقق من الاشتراك
async def is_subscribed(user_id: int) -> bool:
    try:
        member = await bot.get_chat_member(chat_id=CHANNEL_ID, user_id=user_id)
        return member.status in ["member", "administrator", "creator"]
    except Exception as e:
        print(f"Error checking subscription: {e}")
        return False

# 2. كيبورد الاشتراك
def get_subscribe_keyboard():
    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="الاشتراك في القناة ✅", url=f"https://t.me/ulxath")],
        [InlineKeyboardButton(text="تحقق مرة أخرى 🔄", callback_data="check_sub")]
    ])
    return keyboard

# 3. أمر /start
@dp.message(CommandStart())
async def start_cmd(message: Message):
    welcome_text = """
أهلاً بك 👋
أنا بوت تحميل الفيديوهات بدون علامة مائية.

للبدء يجب عليك الاشتراك في القناة أولاً ثم أرسل لي رابط الفيديو.
"""
    if await is_subscribed(message.from_user.id):
        await message.answer("أهلاً بك! أرسل لي رابط تيك توك أو انستا أو يوتيوب وسأحمله لك فوراً.")
    else:
        await message.answer(welcome_text, reply_markup=get_subscribe_keyboard())

# 4. التحقق عند الضغط على زر "تحقق مرة أخرى"
@dp.callback_query(F.data == "check_sub")
async def check_sub_callback(callback_query):
    user_id = callback_query.from_user.id
    if await is_subscribed(user_id):
        await callback_query.message.edit_text("تم التحقق ✅ الآن أرسل لي الرابط")
    else:
        await callback_query.answer("عذراً، لم تشترك بعد", show_alert=True)

# 5. معالجة الروابط والتحميل
@dp.message(F.text)
async def handle_link(message: Message):
    user_id = message.from_user.id
    url = message.text.strip()

    # تحقق الاشتراك أولاً
    if not await is_subscribed(user_id):
        await message.answer(
            "عذراً، يجب عليك الاشتراك في القناة أولاً: https://t.me/ulxath",
            reply_markup=get_subscribe_keyboard()
        )
        return

    # تحقق أن النص رابط
    if not url.startswith("http"):
        await message.answer("الرجاء إرسال رابط فيديو صالح.")
        return

    status_msg = await message.answer("⏳ جاري التحميل... انتظر قليلاً")

    try:
        # إعدادات yt-dlp لتحميل بدون علامة مائية
        ydl_opts = {
            'outtmpl': 'downloads/%(id)s.%(ext)s',
            'format': 'mp4/best',
            'merge_output_format': 'mp4',
            'noplaylist': True,
            'quiet': True,
        }

        # تأكد من وجود مجلد التحميل
        os.makedirs("downloads", exist_ok=True)

        # التحميل
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(url, download=True)
            file_path = ydl.prepare_filename(info)
            title = info.get('title', 'video')

        # إرسال الفيديو
        await bot.send_video(
            chat_id=user_id,
            video=FSInputFile(file_path),
            caption=f"✅ {title}"
        )
        
        # حذف الملف بعد الإرسال لتوفير المساحة
        os.remove(file_path)
        await status_msg.delete()

    except yt_dlp.utils.DownloadError:
        await status_msg.edit_text("❌ الرابط غير صالح أو الفيديو محذوف.")
    except Exception as e:
        print(f"Download error: {e}")
        await status_msg.edit_text("❌ حدث خطأ أثناء التحميل. تأكد من الرابط وحاول مرة أخرى.")

async def main():
    print("Bot is running...")
    await dp.start_polling(bot)

if __name__ == "__main__":
    from aiogram.types import FSInputFile
    asyncio.run(main())
