# python-s2
Telegram assistant
import os, time, re, sqlite3, random, asyncio, datetime
from telegram import Update, ChatPermissions, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder, CommandHandler, MessageHandler, CallbackQueryHandler,
    ContextTypes, filters
)
import openai

# ================== CONFIG ==================
TOKEN = os.getenv("BOT_TOKEN")
OPENAI_KEY = os.getenv("OPENAI_KEY")
OWNER_ID = int(os.getenv("OWNER_ID"))

openai.api_key = OPENAI_KEY

# ================== DATABASE ==================
db = sqlite3.connect("bot.db", check_same_thread=False)
cur = db.cursor()
cur.execute("""
CREATE TABLE IF NOT EXISTS users(
    user_id INTEGER PRIMARY KEY,
    warns INTEGER DEFAULT 0
)
""")
db.commit()

# ================== SETTINGS ==================
BAD_WORDS = ["spam", "scam", "fake"]
LINK_REGEX = r"(https?://|www\.)"
FLOOD_LIMIT = 5
FLOOD_TIME = 5
flood = {}
captcha_users = {}

# ================== BOT FUNCTIONS ==================

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "👑 Ultimate Boss Bot Online!\nAll features are free for everyone!\nAI + Admin + Multi-Niche active."
    )

# Welcome + CAPTCHA
async def welcome(update: Update, context: ContextTypes.DEFAULT_TYPE):
    for user in update.message.new_chat_members:
        a, b = random.randint(1,9), random.randint(1,9)
        captcha_users[user.id] = a + b

        await context.bot.restrict_chat_member(
            update.effective_chat.id,
            user.id,
            ChatPermissions(can_send_messages=False)
        )

        await update.message.reply_text(
            f"👋 Welcome {user.first_name}\n🔐 CAPTCHA: {a} + {b} ? (60s)"
        )

        await asyncio.sleep(60)
        if user.id in captcha_users:
            await context.bot.ban_chat_member(update.effective_chat.id, user.id)
            del captcha_users[user.id]

# CAPTCHA CHECK
async def captcha_check(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.message.from_user.id
    if uid in captcha_users and update.message.text.isdigit():
        if int(update.message.text) == captcha_users[uid]:
            del captcha_users[uid]
            await context.bot.restrict_chat_member(
                update.effective_chat.id,
                uid,
                ChatPermissions(can_send_messages=True)
            )
            await update.message.reply_text("✅ Verified! Welcome 🎉")

# Admin Commands
async def warn(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.reply_to_message:
        uid = update.message.reply_to_message.from_user.id
        cur.execute("INSERT OR IGNORE INTO users(user_id) VALUES(?)", (uid,))
        cur.execute("UPDATE users SET warns = warns + 1 WHERE user_id=?", (uid,))
        db.commit()
        warns = cur.execute("SELECT warns FROM users WHERE user_id=?", (uid,)).fetchone()[0]
        if warns >= 3:
            await context.bot.ban_chat_member(update.effective_chat.id, uid)
            await update.message.reply_text("🚫 User banned (3 warnings)")
        else:
            await update.message.reply_text(f"⚠ Warning {warns}/3")

async def mute(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.reply_to_message:
        uid = update.message.reply_to_message.from_user.id
        await context.bot.restrict_chat_member(update.effective_chat.id, uid, ChatPermissions(can_send_messages=False))
        await update.message.reply_text("🔇 User muted")

async def unmute(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.reply_to_message:
        uid = update.message.reply_to_message.from_user.id
        await context.bot.restrict_chat_member(update.effective_chat.id, uid, ChatPermissions(can_send_messages=True))
        await update.message.reply_text("🔊 User unmuted")

async def ban(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.reply_to_message:
        uid = update.message.reply_to_message.from_user.id
        await context.bot.ban_chat_member(update.effective_chat.id, uid)
        await update.message.reply_text("🚫 User banned")

# Anti-Spam / Filter
async def anti_spam(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text.lower()
    if re.search(LINK_REGEX, text):
        await update.message.delete()
        await update.message.reply_text("🚫 Links not allowed")
        return

    if any(word in text for word in BAD_WORDS):
        await update.message.delete()
        await update.message.reply_text("🚫 Bad words not allowed")

# Flood Control
async def anti_flood(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.message.from_user.id
    now = time.time()
    flood.setdefault(uid, []).append(now)
    flood[uid] = [t for t in flood[uid] if now - t < FLOOD_TIME]
    if len(flood[uid]) > FLOOD_LIMIT:
        await context.bot.restrict_chat_member(update.effective_chat.id, uid, ChatPermissions(can_send_messages=False))
        await update.message.reply_text("🚨 Flood detected – muted")

# AI Reply
async def ai_reply(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user_text = update.message.text
        response = openai.ChatCompletion.create(
            model="gpt-3.5-turbo",
            messages=[
                {"role":"system","content":"You are a helpful Telegram bot assistant."},
                {"role":"user","content":user_text}
            ]
        )
        reply = response["choices"][0]["message"]["content"]
        await update.message.reply_text(reply)
    except:
        await update.message.reply_text("⚠ AI Error, try again later.")

# Bot Start
def main():
    app = ApplicationBuilder().token(TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("warn", warn))
    app.add_handler(CommandHandler("mute", mute))
    app.add_handler(CommandHandler("unmute", unmute))
    app.add_handler(CommandHandler("ban", ban))

    app.add_handler(MessageHandler(filters.StatusUpdate.NEW_CHAT_MEMBERS, welcome))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, captcha_check))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, anti_flood))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, anti_spam))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, ai_reply))

    print("👑 BOT IS RUNNING")
    app.run_polling()

if __name__ == "__main__":
    main()
