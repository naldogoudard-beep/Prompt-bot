import os
import logging
import google.generativeai as genai
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    filters,
    ContextTypes,
    CallbackQueryHandler,
)
from io import BytesIO

logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO,
)
logger = logging.getLogger(__name__)

TELEGRAM_TOKEN = os.environ.get("TELEGRAM_TOKEN")
GEMINI_API_KEY = os.environ.get("GEMINI_API_KEY")

genai.configure(api_key=GEMINI_API_KEY)
model = genai.GenerativeModel("gemini-1.5-flash")

PROMPT_INSTRUCOES = """Você é um especialista em fotografia de produto, marketing visual e geração de prompts para IA de imagem (Midjourney, DALL-E, Stable Diffusion).

Analise o produto na imagem e responda EXATAMENTE neste formato:

🎯 *PROMPT PRINCIPAL*
[Crie o melhor prompt em inglês para fotografar este produto. Inclua: modelo/pessoa ideal, cenário perfeito, iluminação, estilo fotográfico e qualidade técnica. Um parágrafo detalhado.]

📌 *Tradução:* [tradução do prompt acima em português]

---

📸 *5 ÂNGULOS DIFERENTES*

1️⃣ *Close-up / Detalhe*
[prompt em inglês focando nos detalhes e textura]
📌 *Tradução:* [em português]

2️⃣ *Lifestyle / Uso*
[prompt em inglês com produto sendo usado no dia a dia]
📌 *Tradução:* [em português]

3️⃣ *Flat Lay / Editorial*
[prompt em inglês vista de cima estilo revista]
📌 *Tradução:* [em português]

4️⃣ *Ambiente / Mood*
[prompt em inglês focando no cenário e atmosfera]
📌 *Tradução:* [em português]

5️⃣ *Minimalista / Estúdio*
[prompt em inglês fundo limpo produto como protagonista]
📌 *Tradução:* [em português]

---

📣 *DESCRIÇÃO PARA PROPAGANDA*

[Headline impactante em maiúsculas]

[Texto persuasivo de 3-4 linhas destacando os benefícios e despertando desejo]

[Call to action]

[20 hashtags relevantes em português e inglês]"""


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    texto = (
        "👋 *Olá! Sou seu Bot Gerador de Prompts!*\n\n"
        "📸 Me envie a foto de qualquer produto e vou gerar:\n\n"
        "🎯 Prompt principal com modelo e cenário ideal\n"
        "📸 5 variações de ângulos diferentes\n"
        "📣 Descrição de propaganda com hashtags\n\n"
        "🚀 *É só mandar a foto!*"
    )
    await update.message.reply_text(texto, parse_mode="Markdown")


async def handle_photo(update: Update, context: ContextTypes.DEFAULT_TYPE):
    loading = await update.message.reply_text(
        "⏳ *Analisando seu produto...*\n\nAguarde alguns segundos!",
        parse_mode="Markdown",
    )

    try:
        photo = update.message.photo[-1]
        photo_file = await context.bot.get_file(photo.file_id)

        image_bytes_io = BytesIO()
        await photo_file.download_to_memory(image_bytes_io)
        image_bytes = image_bytes_io.getvalue()

        import PIL.Image
        image = PIL.Image.open(BytesIO(image_bytes))

        response = model.generate_content([PROMPT_INSTRUCOES, image])
        resultado = response.text

        await loading.delete()

        # Divide em mensagens se for muito longo
        if len(resultado) <= 4096:
            await update.message.reply_text(resultado, parse_mode="Markdown")
        else:
            partes = [resultado[i:i+4000] for i in range(0, len(resultado), 4000)]
            for parte in partes:
                await update.message.reply_text(parte, parse_mode="Markdown")

        keyboard = [[InlineKeyboardButton("📸 Enviar outro produto", callback_data="novo")]]
        await update.message.reply_text(
            "✅ *Prompts gerados! Envie outra foto quando quiser.*",
            parse_mode="Markdown",
            reply_markup=InlineKeyboardMarkup(keyboard),
        )

    except Exception as e:
        logger.error(f"Erro: {e}")
        await loading.edit_text(
            "❌ *Erro ao processar a imagem. Tente novamente!*",
            parse_mode="Markdown",
        )


async def handle_documento(update: Update, context: ContextTypes.DEFAULT_TYPE):
    doc = update.message.document
    if not doc.mime_type or not doc.mime_type.startswith("image/"):
        await update.message.reply_text("⚠️ Envie apenas imagens (JPG ou PNG).")
        return

    loading = await update.message.reply_text(
        "⏳ *Analisando seu produto...*\n\nAguarde alguns segundos!",
        parse_mode="Markdown",
    )

    try:
        doc_file = await context.bot.get_file(doc.file_id)
        image_bytes_io = BytesIO()
        await doc_file.download_to_memory(image_bytes_io)
        image_bytes = image_bytes_io.getvalue()

        import PIL.Image
        image = PIL.Image.open(BytesIO(image_bytes))

        response = model.generate_content([PROMPT_INSTRUCOES, image])
        resultado = response.text

        await loading.delete()

        if len(resultado) <= 4096:
            await update.message.reply_text(resultado, parse_mode="Markdown")
        else:
            partes = [resultado[i:i+4000] for i in range(0, len(resultado), 4000)]
            for parte in partes:
                await update.message.reply_text(parte, parse_mode="Markdown")

    except Exception as e:
        logger.error(f"Erro: {e}")
        await loading.edit_text("❌ Erro ao processar. Tente novamente!")


async def button_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    await query.message.reply_text(
        "📸 *Mande a foto do produto!*", parse_mode="Markdown"
    )


async def handle_texto(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "📸 *Me envie uma foto do produto para gerar os prompts!*\n\nUse /start para ver as instruções.",
        parse_mode="Markdown",
    )


def main():
    app = Application.builder().token(TELEGRAM_TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.PHOTO, handle_photo))
    app.add_handler(MessageHandler(filters.Document.IMAGE, handle_documento))
    app.add_handler(CallbackQueryHandler(button_callback))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_texto))

    logger.info("Bot iniciado!")
    app.run_polling(allowed_updates=Update.ALL_TYPES)


if __name__ == "__main__":
    main()
