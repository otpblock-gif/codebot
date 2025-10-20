import requests
from bs4 import BeautifulSoup
from telegram import (
    Update,
    InlineKeyboardMarkup,
    InlineKeyboardButton,
)
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    filters,
    ContextTypes,
    CallbackQueryHandler,
)
import json
import os

BOT_TOKEN = "7576294576:AAEfgCG3qDaS5Z3bvt3mYNCzoTZMFRKZR8I"
ADMIN_ID = 6550324099  # Replace with your Telegram user ID
user_state = {}

STATS_FILE = "stats.json"
users_data = {}

# ====== Load stats from file ======
def load_stats():
    global users_data
    if not os.path.isfile(STATS_FILE):
        users_data = {}
        return

    try:
        with open(STATS_FILE, "r") as f:
            data = json.load(f)
            if isinstance(data, dict):
                users_data = data
            else:
                print("Warning: stats.json content is not a dict. Resetting data.")
                users_data = {}
    except Exception as e:
        print(f"Error loading stats.json: {e}")
        users_data = {}

# ====== Save stats to file ======
def save_stats():
    try:
        with open(STATS_FILE, "w") as f:
            json.dump(users_data, f, indent=2)
    except Exception as e:
        print(f"Error saving stats.json: {e}")

# ====== Inline Keyboard for search type ======
def get_search_inline_keyboard():
    # MODIFICATION: Removed CNIC search button as the new website doesn't support it
    keyboard = [
        [
            InlineKeyboardButton("🔍 Search by Number", callback_data="search_number"),
        ]
    ]
    return InlineKeyboardMarkup(keyboard)

# ====== /start Command ======
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    user_id = str(user.id)
    username = user.username or user.first_name or "Unknown"

    if user_id not in users_data:
        users_data[user_id] = {
            "username": username,
            "search_count": 0,
            "searches": []
        }
        save_stats()
    else:
        users_data[user_id]["username"] = username
        save_stats()

    user_state.pop(update.effective_chat.id, None)

    await update.message.reply_text(
        "👋 Welcome! Please choose an option to search:",
        reply_markup=get_search_inline_keyboard()
    )

# ====== Callback Query Handler for button presses ======
async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()  # Acknowledge callback to remove loading spinner
    chat_id = query.message.chat_id

    if query.data == "search_number":
        user_state[chat_id] = "number"
        # MODIFICATION: Updated prompt to match new website requirements
        await query.message.reply_text(
            "📱 Please enter the mobile number (12 digits, starting with 92):\n"
            "Example: 923067632070"
        )
    # MODIFICATION: Removed 'search_cnic' handler
    else:
        await query.message.reply_text("⚠ Unknown option selected.")

# ====== Handler for text messages after choosing search type ======
async def menu_choice(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.effective_chat.id
    user = update.effective_user
    user_id = str(user.id)
    username = user.username or user.first_name or "Unknown"
    text = update.message.text.strip()

    if user_id not in users_data:
        users_data[user_id] = {
            "username": username,
            "search_count": 0,
            "searches": []
        }
    else:
        users_data[user_id]["username"] = username
    save_stats()

    if chat_id not in user_state:
        # User hasn't selected search type yet
        await update.message.reply_text(
            "⚠ Please start by typing /start and selecting an option:",
            reply_markup=get_search_inline_keyboard()
        )
        return

    search_type = user_state[chat_id]

    # Validate input
    if search_type == "number":
        # MODIFICATION: Updated validation for 12 digits starting with 92
        if not text.isdigit() or len(text) != 12 or not text.startswith("92"):
            await update.message.reply_text("❌ Invalid mobile number. Enter 12 digits starting with 92 (e.g., 923067632070).")
            return
    # MODIFICATION: Removed CNIC validation block as it's no longer an option

    # Update stats
    users_data[user_id]["search_count"] += 1
    users_data[user_id]["searches"].append({"type": search_type, "query": text})
    save_stats()

    await update.message.reply_text("🔍 Searching... Please wait.")

    # MODIFICATION: New payload and URL
    payload = {"search_query": text}
    url = "https://pakistandatabase.com/databases/sim.php"

    try:
        response = requests.post(url, data=payload)
        soup = BeautifulSoup(response.text, "html.parser")

        # MODIFICATION: New parsing logic for <table class="api-response">
        result_table = soup.find("table", class_="api-response")
        if not result_table:
            await update.message.reply_text("⚠ No result found. (Result table not found on page)")
            await send_developer_info(update)
            return

        tbody = result_table.find("tbody")
        if not tbody:
            await update.message.reply_text("⚠ No result found. (Result body not found on page)")
            await send_developer_info(update)
            return

        rows = tbody.find_all("tr")
        
        # MODIFICATION: Updated check. We just need to see if any rows exist in tbody
        if not rows:
            await update.message.reply_text("⚠ No result found.")
            await send_developer_info(update)
            return

        result_text = ""
        # MODIFICATION: Loop starts from the first row (no header in tbody)
        for row in rows: 
            cols = [col.get_text(strip=True) for col in row.find_all("td")]
            if cols:
                # The column order (Mobile, Name, CNIC, Address) is the same
                result_text += (
                    f"📱 Mobile: {cols[0]}\n"
                    f"👤 Name: {cols[1]}\n"
                    f"🆔 CNIC: {cols[2]}\n"
                    f"🏠 Address: {cols[3]}\n\n"
                )

        await update.message.reply_text(result_text.strip() or "⚠ No data found.")
        await send_developer_info(update)

    except Exception as e:
        await update.message.reply_text(f"❌ Error occurred: {e}")
        await send_developer_info(update)

# ====== Developer info ======
async def send_developer_info(update: Update):
    developer_msg = "🤖 Bot developed by Muazam Ali\n📞 WhatsApp: +923067632070"
    await update.message.reply_text(developer_msg)
    await update.message.reply_text(
        "Choose your search type:",
        reply_markup=get_search_inline_keyboard()
    )
    user_state.pop(update.effective_chat.id, None)

# ====== /stats command for admin ======
async def stats_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user_id = update.effective_user.id
        if user_id != ADMIN_ID:
            await update.message.reply_text("⛔ You are not authorized to view stats.")
            return

        if not users_data:
            await update.message.reply_text("No users or searches recorded yet.")
            return

        msg = f"📊 Bot Stats:\nTotal users: {len(users_data)}\n\n"
        for uid, data in users_data.items():
            if not isinstance(data, dict):
                print(f"Skipping invalid user data for user id: {uid}")
                continue

            username_display = data.get('username') or "Unknown"
            # Show username only (no user ID)
            msg += f"👤 @{username_display}\n"
            msg += f"🔍 Searches made: {data.get('search_count', 0)}\n"
            if data.get('searches'):
                msg += "Search queries:\n"
                for s in data['searches']:
                    # MODIFICATION: Type will now always be 'number'
                    qtype = "Number" if s['type'] == "number" else "Unknown" 
                    msg += f" - [{qtype}] {s['query']}\n"
            msg += "\n"

            # Telegram message size limit safeguard
            if len(msg) > 3500:
                await update.message.reply_text(msg)
                msg = ""

        if msg:
            await update.message.reply_text(msg)

    except Exception as e:
        await update.message.reply_text(f"❌ Error: {e}")

# ====== Main ======
if __name__ == "__main__":
    load_stats()

    app = ApplicationBuilder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("stats", stats_command))
    app.add_handler(CallbackQueryHandler(button_handler))  # For inline buttons
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, menu_choice))

    print("🤖 Bot is running...")
    app.run_polling()
