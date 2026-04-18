# ent_bot.py
import logging
import os
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application, CommandHandler, CallbackQueryHandler,
    ContextTypes, ConversationHandler
)

BOT_TOKEN = os.getenv("BOT_TOKEN
")
FREE_LIMIT = 5
PAYMENT_TEXT = "💳 Оплатить подписку: @erdgxz"

logging.basicConfig(level=logging.INFO)

CHOOSING_SUBJECT, ANSWERING = range(2)

QUESTIONS = {
    "Математика": [
        {
            "q": "2 + 2 = ?",
            "options": ["3", "4", "5", "6"],
            "answer": 1,
            "explain": "Правильный ответ: 4"
        }
    ]
}

user_data_store = {}

def get_user(user_id):
    if user_id not in user_data_store:
        user_data_store[user_id] = {
            "count": 0,
            "correct": 0,
            "q_index": 0,
            "subject": None,
            "paid": False,
        }
    return user_data_store[user_id]

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [
        [InlineKeyboardButton("📐 Математика", callback_data="Математика")]
    ]
    await update.message.reply_text(
        "📚 Выбери предмет:",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )
    return CHOOSING_SUBJECT

async def choose_subject(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    user = get_user(query.from_user.id)
    user["subject"] = query.data
    user["q_index"] = 0

    await send_question(query, user)
    return ANSWERING

async def send_question(query, user):
    q = QUESTIONS[user["subject"]][user["q_index"]]

    keyboard = [
        [InlineKeyboardButton(opt, callback_data=f"ans_{i}")]
        for i, opt in enumerate(q["options"])
    ]

    await query.edit_message_text(
        q["q"],
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

async def handle_answer(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    user = get_user(query.from_user.id)
    q = QUESTIONS[user["subject"]][user["q_index"]]

    selected = int(query.data.split("_")[1])

    if selected == q["answer"]:
        text = "✅ Правильно!"
    else:
        text = f"❌ Неправильно\nПравильный ответ: {q['options'][q['answer']]}"

    await query.edit_message_text(text)
    return CHOOSING_SUBJECT

def main():
    app = Application.builder().token(BOT_TOKEN).build()

    conv = ConversationHandler(
        entry_points=[CommandHandler("start", start)],
        states={
            CHOOSING_SUBJECT: [CallbackQueryHandler(choose_subject)],
            ANSWERING: [CallbackQueryHandler(handle_answer, pattern="^ans_")],
        },
        fallbacks=[CommandHandler("start", start)],
    )

    app.add_handler(conv)

    print("🤖 Бот запущен!")
    app.run_polling()

if __name__ == "__main__":
    main()
