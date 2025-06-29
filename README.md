# Telgram-video-bot
 Telgram bot for downloading videos
import os
import logging
import requests
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, ContextTypes, filters
import yt_dlp
import asyncio
import nest_asyncio

BOT_TOKEN = os.getenv("BOT_TOKEN")  # التوكن من متغيرات البيئة

nest_asyncio.apply()
logging.basicConfig(level=logging.INFO)

def download_video(url):
    ydl_opts = {
        'format': 'bestvideo+bestaudio/best',
        'outtmpl': 'video.%(ext)s',
        'noplaylist': True,
        'merge_output_format': 'mp4',
        # لتجنب مشاكل بعض المواقع مثل تيك توك
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

def upload_to_gofile(file_path):
    with open(file_path, 'rb') as f:
        response = requests.post('https://store1.gofile.io/uploadFile', files={'file': f})
        data = response.json()
        if data['status'] == 'ok':
            return data['data']['downloadPage']
        else:
            raise Exception("فشل رفع الملف إلى GoFile")

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("👋 أهلاً! أرسل لي رابط فيديو من تيك توك، لايكا، إنستغرام أو فيسبوك وسأحمل لك الفيديو.")

async def handle_link(update: Update, context: ContextTypes.DEFAULT_TYPE):
    url = update.message.text
    await update.message.reply_text("⏳ جاري تحميل الفيديو...")

    try:
        loop = asyncio.get_event_loop()
        file_path = await loop.run_in_executor(None, download_video, url)
        size_mb = os.path.getsize(file_path) / (1024 * 1024)

        if size_mb <= 50:
            await update.message.reply_video(video=open(file_path, 'rb'))
        else:
            link = upload_to_gofile(file_path)
            await update.message.reply_text(f"📎 الفيديو كبير، تم رفعه هنا:\n{link}")

        os.remove(file_path)

    except Exception as e:
        await update.message.reply_text(f"❌ حدث خطأ أثناء التحميل:\n{e}")

async def main():
    app = ApplicationBuilder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_link))
    await app.run_polling()

asyncio.run(main())
