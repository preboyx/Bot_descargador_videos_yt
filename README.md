#bot descargardor de vídeos de YouTube
from pytube import YouTube
from telebot import TeleBot
from telebot import types
from pathlib import Path
import re

bot = TeleBot("6390514495:AAGyZVKTICOuDO1TS49ZbG9Y32jRuYMPm1g")

active_downloads = {}

@bot.message_handler(commands=["start"])
def ask_for_link(message):
    dev = types.InlineKeyboardButton("👨‍💻 Creador", url="https://t.me/Preboyx")
    channel = types.InlineKeyboardButton("📣 Canal", url="https://t.me/XpReBoY")
    markup = types.InlineKeyboardMarkup(row_width=2)
    markup.add(dev)
    markup.add(channel)
    bot.send_message(message.chat.id, """*
🚀 ¡Bienvenido al Bot Descargador de vídeos de YouTube!

📥 Este bot te permite descargar videos de YouTube fácilmente. Simplemente envía el enlace del vídeo que deseas descargar y el bot se encargará del resto.

🔧 A continuación se explica cómo utilizar el bot:
1. Envíe el enlace del video de YouTube que desea descargar.
2. Espere a que el bot procese la descarga.
3. Una vez que se complete la descarga, el bot te enviará el video.

📌 Tenga en cuenta:
⋅ No se pueden descargar vídeos de más de 15 minutos.
∙ Solo se pueden descargar videos públicos.
∙ Es posible que el bot no funcione correctamente con videos privados o con restricción de edad.

🎉 ¡Disfruta descargando tus videos favoritos de YouTube!*""", parse_mode="Markdown", reply_markup=markup)

@bot.message_handler(func=lambda message: True)
def download_and_send_video(message):
    try:
        user_id = message.from_user.id
        if user_id in active_downloads:
            bot.send_message(message.chat.id, "🕐 *Se está realizando otra descarga de vídeo. Espere por favor.*", parse_mode="Markdown")
            return

        link = message.text
        if "youtu.be" in link:
            video_id = re.search(r"youtu.be/([^?]+)", link).group(1)
            link = f"https://www.youtube.com/watch?v={video_id}"
        if not re.match(r'^(https?://)?(www\.)?youtube\.com/watch\?v=[\w-]+(&\S*)?$', link):
            bot.send_message(message.chat.id, "❌ *Por favor envíe un enlace válido*", parse_mode="Markdown")
            return
        yt = YouTube(link)
        if yt.length > 900:
            bot.send_message(message.chat.id, "⏰ *No se pueden descargar vídeos de más de 15 minutos.*", parse_mode="Markdown")
            return

        def show_progress(stream, chunk, bytes_remaining):
            percent = round((1 - bytes_remaining / stream.filesize) * 100, 1)
            global msg
            if percent % 1 == 0:
                try:
                    if 'msg' in globals():
                        bot.edit_message_text(f"📥 *Video is {percent}% downloaded...*", message.chat.id, msg.message_id, parse_mode="Markdown")
                    else:
                        msg = bot.send_message(message.chat.id, f"📥 *Video is {percent}% downloaded...*", parse_mode="Markdown")

                except:
                    pass
        started_message = bot.send_message(message.chat.id, "📥 *La descarga comenzó!*", parse_mode="Markdown")
        active_downloads[user_id] = True
        video = yt.streams.get_highest_resolution().download(Path("./tmp")) 
        bot.delete_message(message.chat.id, started_message.message_id)
        bot.send_message(message.chat.id, "✅ *¡El proceso de descarga de video se completó exitosamente! Enviando el vídeo...*", parse_mode="Markdown")
        bot.send_chat_action(message.chat.id, "upload_video")

        with open(video, "rb") as video_file:
            bot.send_video(message.chat.id, video_file)
        Path(video).unlink()
        del active_downloads[user_id]
    except Exception as e:
        bot.send_message(message.chat.id, f"❌ *Ocurrió un error:* {e}\n*Inténtalo de nuevo*", parse_mode="Markdown")

bot.infinity_polling()
