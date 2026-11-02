# Jo1dasbaev_bot

import os
import sqlite3
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo

from telegram import Update, ReplyKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    ConversationHandler,
    ContextTypes,
    filters,
)

TOKEN = os.getenv("BOT_TOKEN")
TZ = ZoneInfo("Asia/Tashkent")
DB = "xarajatlar.db"

AMOUNT, NOTE = range(2)

MENU = ReplyKeyboardMarkup(
    [
        ["➕ Xarajat qo‘shish", "💰 Daromad qo‘shish"],
        ["📊 Bugungi hisobot", "📅 Haftalik hisobot"],
        ["🗓 Oylik hisobot", "💵 Umumiy balans"],
        ["📜 Tarix"],
    ],
    resize_keyboard=True
)


def db():
    return sqlite3.connect(DB)


def init_db():
    with db() as con:
        con.execute("""
            CREATE TABLE IF NOT EXISTS records (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                kind TEXT NOT NULL,
                amount INTEGER NOT NULL,
                note TEXT,
                created_at TEXT NOT NULL
            )
        """)


def money(n):
    return f"{n:,}".replace(",", " ") + " so‘m"


def add_record(user_id, kind, amount, note):
    with db() as con:
        con.execute(
            """INSERT INTO records
            (user_id, kind, amount, note, created_at)
            VALUES (?, ?, ?, ?, ?)""",
            (
                user_id,
                kind,
                amount,
                note,
                datetime.now(TZ).isoformat(timespec="seconds")
            )
        )


def totals(user_id, start=None, end=None):
    query = """
        SELECT kind, SUM(amount)
        FROM records
        WHERE user_id = ?
    """
    params = [user_id]

    if start:
        query += " AND created_at >= ?"
        params.append(start.isoformat(timespec="seconds"))

    if end:
        query += " AND created_at < ?"
        params.append(end.isoformat(timespec="seconds"))

    query += " GROUP BY kind"

    result = {"income": 0, "expense": 0}

    with db() as con:
        for kind, amount in con.execute(query, params):
            result[kind] = amount or 0

    return result


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "Assalomu alaykum! 👋\n"
        "Xarajatlar hisob-kitob botiga xush kelibsiz!\n\n"
        "Kerakli bo‘limni tanlang:",
        reply_markup=MENU
    )


async def begin_expense(update, context):
    context.user_data["kind"] = "expense"
    await update.message.reply_text(
        "💸 Xarajat miqdorini so‘mda kiriting.\n"
        "Masalan: 25000\n\n"
        "Bekor qilish: /cancel"
    )
    return AMOUNT


async def begin_income(update, context):
    context.user_data["kind"] = "income"
    await update.message.reply_text(
        "💰 Daromad miqdorini so‘mda kiriting.\n"
        "Masalan: 500000"
    )
    return AMOUNT


async def get_amount(update, context):
    raw = update.message.text.strip().replace(" ", "")

    if not raw.isdigit() or int(raw) <= 0:
        await update.message.reply_text(
            "Iltimos, faqat musbat butun son kiriting. Masalan: 25000"
        )
        return AMOUNT

    context.user_data["amount"] = int(raw)

    await update.message.reply_text(
        "📝 Izoh yozing.\nMasalan: Ovqat, yo‘l haqi yoki ish haqi."
    )
    return NOTE


async def get_note(update, context):
    note = update.message.text.strip()

    if not note or len(note) > 200:
        await update.message.reply_text(
            "Izoh 1 dan 200 tagacha belgidan iborat bo‘lsin."
        )
        return NOTE

    kind = context.user_data["kind"]
    amount = context.user_data["amount"]

    add_record(
        update.effective_user.id,
        kind,
        amount,
        note
    )

    label = "Xarajat" if kind == "expense" else "Daromad"

    await update.message.reply_text(
        f"✅ {label} saqlandi!\n\n"
        f"💵 Miqdor: {money(amount)}\n"
        f"📝 Izoh: {note}",
        reply_markup=MENU
    )

    context.user_data.clear()
    return ConversationHandler.END


async def cancel(update, context):
    context.user_data.clear()
    await update.message.reply_text(
        "Amal bekor qilindi.",
        reply_markup=MENU
    )
    return ConversationHandler.END


async def daily(update, context):
    now = datetime.now(TZ)
    start_time = now.replace(hour=0, minute=0, second=0, microsecond=0)
    result = totals(update.effective_user.id, start_time)

    await update.message.reply_text(
        "📊 BUGUNGI HISOBOT\n\n"
        f"💰 Daromad: {money(result['income'])}\n"
        f"💸 Xarajat: {money(result['expense'])}\n"
        f"📈 Farq: {money(result['income'] - result['expense'])}",
        reply_markup=MENU
    )


async def weekly(update, context):
    now = datetime.now(TZ)
    start_time = (now - timedelta(days=now.weekday())).replace(
        hour=0, minute=0, second=0, microsecond=0
    )
    result = totals(update.effective_user.id, start_time)

    await update.message.reply_text(
        "📅 HAFTALIK HISOBOT\n\n"
        f"💰 Daromad: {money(result['income'])}\n"
        f"💸 Xarajat: {money(result['expense'])}\n"
        f"📈 Farq: {money(result['income'] - result['expense'])}",
        reply_markup=MENU
    )


async def monthly(update, context):
    now = datetime.now(TZ)
    start_time = now.replace(
        day=1, hour=0, minute=0, second=0, microsecond=0
    )
    result = totals(update.effective_user.id, start_time)

    await update.message.reply_text(
        "🗓 OYLIK HISOBOT\n\n"
        f"💰 Daromad: {money(result['income'])}\n"
        f"💸 Xarajat: {money(result['expense'])}\n"
        f"📈 Farq: {money(result['income'] - result['expense'])}",
        reply_markup=MENU
    )


async def balance(update, context):
    result = totals(update.effective_user.id)

    await update.message.reply_text(
        "💵 UMUMIY BALANS\n\n"
        f"💰 Jami daromad: {money(result['income'])}\n"
        f"💸 Jami xarajat: {money(result['expense'])}\n"
        f"📊 Qoldiq: {money(result['income'] - result['expense'])}",
        reply_markup=MENU
    )


async def history(update, context):
    with db() as con:
        rows = con.execute(
            """SELECT kind, amount, note, created_at
            FROM records WHERE user_id = ?
            ORDER BY id DESC LIMIT 10""",
            (update.effective_user.id,)
        ).fetchall()

    if not rows:
        await update.message.reply_text(
            "Hozircha tarix bo‘sh.", reply_markup=MENU
        )
        return

    lines = ["📜 OXIRGI 10 TA YOZUV\n"]

    for kind, amount, note, created_at in rows:
        label = "💸 Xarajat" if kind == "expense" else "💰 Daromad"
        date = datetime.fromisoformat(created_at).strftime("%d.%m.%Y %H:%M")
        lines.append(
            f"{label}: {money(amount)}\n"
            f"📝 {note}\n"
            f"🕒 {date}\n"
        )

    await update.message.reply_text(
        "\n".join(lines), reply_markup=MENU
    )


def main():
    if not TOKEN:
        raise RuntimeError("BOT_TOKEN muhit o‘zgaruvchisi kiritilmagan!")

    init_db()

    app = ApplicationBuilder().token(TOKEN).build()

    conversation = ConversationHandler(
        entry_points=[
            MessageHandler(
                filters.Regex(r"^➕ Xarajat qo‘shish$"),
                begin_expense
            ),
            MessageHandler(
                filters.Regex(r"^💰 Daromad qo‘shish$"),
                begin_income
            ),
        ],
        states={
            AMOUNT: [
                MessageHandler(filters.TEXT & ~filters.COMMAND, get_amount)
            ],
            NOTE: [
                MessageHandler(filters.TEXT & ~filters.COMMAND, get_note)
            ],
        },
        fallbacks=[CommandHandler("cancel", cancel)],
    )

    app.add_handler(CommandHandler("start", start))
    app.add_handler(conversation)

    app.add_handler(MessageHandler(
        filters.Regex(r"^📊 Bugungi hisobot$"), daily
    ))
    app.add_handler(MessageHandler(
        filters.Regex(r"^📅 Haftalik hisobot$"), weekly
    ))
    app.add_handler(MessageHandler(
        filters.Regex(r"^🗓 Oylik hisobot$"), monthly
    ))
    app.add_handler(MessageHandler(
        filters.Regex(r"^💵 Umumiy balans$"), balance
    ))
    app.add_handler(MessageHandler(
        filters.Regex(r"^📜 Tarix$"), history
    ))

    print("Bot ishga tushdi!")
    app.run_polling()


if __name__ == "__main__":
    main()
