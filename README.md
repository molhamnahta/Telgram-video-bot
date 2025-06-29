import os
import logging
import requests
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, ContextTypes, filters
import yt_dlp
import asyncio
import nest_asyncio

# ✅ التوكن مدمج داخل الكود
BOT_TOKEN = "7571947352:AAHoC4-gkJ9b6qQzG9wgR4Dj6ElAqRI3lJY"

nest_asyncio.apply()
logging.basicConfig(level=logging.INFO)

# تحميل الفيديو من الرابط
def download_video(url):
    ydl_opts = {
        'format': 'bestvideo+bestaudio/best',
        'outtmpl': 'video.%(ext)s',
        'noplaylist': True,
        'merge_output_format': 'mp4',
        'quiet': True,
        'no_warnings': True,
        'ignoreerrors': True,
    }
    with yt_dlp.YoutubeDL(ydl_opts) as ydl:
        info = ydl.extract_info(url, download=True)
        filename = ydl.prepare_filename(info)
        if not filename.endswith(".mp4"):
            filename = filename.replace(".webm", ".mp4")
        return filename

# رفع الفيديو إلى GoFile في حال كان كبير الحجم
def upload_to_gofile(file_path):
    with open(file_path, 'rb') as f:
        response = requests.post('https://store1.gofile.io/uploadFile', files={'file': f})
        data = response.json()
        if data['status'] == 'ok':
            return data['data']['downloadPage']
        else:
            raise Exception("فشل رفع الملف إلى GoFile")

# أمر /start
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("👋 أهلاً! أرسل لي رابط فيديو من يوتيوب، تيك توك، لايكا، إنستغرام أو فيسبوك.")

# معالجة الروابط المرسلة
async def handle_link(update: Update, context: ContextTypes.DEFAULT_TYPE):
    url = update.message.text.strip()
    await update.message.reply_text("⏳ جاري تحميل الفيديو...")

    try:
        loop = asyncio.get_event_loop()
        file_path = await loop.run_in_executor(None, download_video, url)
        size_mb = os.path.getsize(file_path) / (1024 * 1024)

        if size_mb <= 50:
            with open(file_path, 'rb') as video_file:
                await update.message.reply_video(video=video_file)
        else:
            link = upload_to_gofile(file_path)
            await update.message.reply_text(f"📎 الفيديو كبير، تم رفعه هنا:\n{link}")

        os.remove(file_path)

    except Exception as e:
        await update.message.reply_text(f"❌ حدث خطأ أثناء التحميل:\n{e}")

# تشغيل البوت
async def main():
    app = ApplicationBuilder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_link))
    await app.run_polling()

if __name__ == "__main__":
    asyncio.run(main())
