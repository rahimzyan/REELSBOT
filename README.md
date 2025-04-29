import os
import openai
from dotenv import load_dotenv
from telegram import Update
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    ConversationHandler,
    filters,
    ContextTypes,
)

# Загрузка ключей
load_dotenv()
BOT_TOKEN = os.getenv("BOT_TOKEN")
openai.api_key = os.getenv("OPENAI_API_KEY")

# Этапы опроса
(
    ROLE, PRODUCT, AUDIENCE, VIBE,
    PAIN, IRONY, EXAMPLES, END
) = range(8)

user_data = {}

# Сценарий-промт
def build_prompt(data):
    return f"""
Ты креативный копирайтер. Сгенерируй сценарий Reels для клиента по структуре:
1. Крючок (боль/ирония)
2. Усугубление
3. Проблема
4. Примеры
5. Решение
6. Призыв к действию

Данные клиента:
- Роль: {data['role']}
- Продукт: {data['product']}
- Аудитория: {data['audience']}
- Вайб: {data['vibe']}
- Боль/триггер: {data['pain']}
- Ирония допустима: {data['irony']}
- Примеры: {data['examples']}
"""

# Команды
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Привет! Расскажи, кто ты по профессии?")
    return ROLE

async def role(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_data['role'] = update.message.text
    await update.message.reply_text("Какой продукт ты продвигаешь?")
    return PRODUCT

async def product(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_data['product'] = update.message.text
    await update.message.reply_text("Кто твоя целевая аудитория?")
    return AUDIENCE

async def audience(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_data['audience'] = update.message.text
    await update.message.reply_text("Опиши вайб или эмоцию аудитории.")
    return VIBE

async def vibe(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_data['vibe'] = update.message.text
    await update.message.reply_text("Какая главная боль или триггер у клиента?")
    return PAIN

async def pain(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_data['pain'] = update.message.text
    await update.message.reply_text("Допускается ли ирония или юмор? (да/нет)")
    return IRONY

async def irony(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_data['irony'] = update.message.text
    await update.message.reply_text("Если есть примеры использования продукта, приведи их.")
    return EXAMPLES

async def examples(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_data['examples'] = update.message.text

    # Генерация промта
    prompt = build_prompt(user_data)
    await update.message.reply_text("Генерирую сценарий...")

    # Вызов OpenAI
    response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.8
    )
    text = response['choices'][0]['message']['content']
    await update.message.reply_text(f"Вот твой сценарий:\n\n{text}")
    return ConversationHandler.END

async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Отменено.")
    return ConversationHandler.END

# Запуск бота
app = ApplicationBuilder().token(BOT_TOKEN).build()

conv_handler = ConversationHandler(
    entry_points=[CommandHandler("start", start)],
    states={
        ROLE: [MessageHandler(filters.TEXT & ~filters.COMMAND, role)],
        PRODUCT: [MessageHandler(filters.TEXT & ~filters.COMMAND, product)],
        AUDIENCE: [MessageHandler(filters.TEXT & ~filters.COMMAND, audience)],
        VIBE: [MessageHandler(filters.TEXT & ~filters.COMMAND, vibe)],
        PAIN: [MessageHandler(filters.TEXT & ~filters.COMMAND, pain)],
        IRONY: [MessageHandler(filters.TEXT & ~filters.COMMAND, irony)],
        EXAMPLES: [MessageHandler(filters.TEXT & ~filters.COMMAND, examples)],
    },
    fallbacks=[CommandHandler("cancel", cancel)],
)

app.add_handler(conv_handler)
app.run_polling()
