from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    CallbackQueryHandler,
    MessageHandler,
    ContextTypes,
    filters,
)
import json
import os
from datetime import datetime


# ============================================================
# SETTINGS
# ============================================================

BOT_TOKEN = "YOUR_BOT_TOKEN_HERE"

# Telegram GROUP ID
GROUP_CHAT_ID = -1001234567890

# Telegram ADMIN USER ID
ADMIN_CHAT_ID = 123456789

DATA_FILE = "feedback_data.json"


# ============================================================
# DATABASE
# ============================================================

def load_data():

    if not os.path.exists(DATA_FILE):
        return {
            "next_number": 1,
            "submissions": []
        }

    with open(DATA_FILE, "r", encoding="utf-8") as file:
        return json.load(file)


def save_data(data):

    with open(DATA_FILE, "w", encoding="utf-8") as file:
        json.dump(
            data,
            file,
            ensure_ascii=False,
            indent=4
        )


data = load_data()


# ============================================================
# START
# ============================================================

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):

    context.user_data.clear()

    keyboard = [
        [
            InlineKeyboardButton(
                "📝 Feedback",
                callback_data="feedback"
            )
        ],
        [
            InlineKeyboardButton(
                "💡 Suggestion",
                callback_data="suggestion"
            )
        ],
        [
            InlineKeyboardButton(
                "⚠️ Complaint",
                callback_data="complaint"
            )
        ]
    ]

    await update.message.reply_text(
        "👋 Welcome to the Student Feedback Bot.\n\n"
        "You can submit your feedback, suggestion, "
        "or complaint here.\n\n"
        "Your submission will be posted anonymously "
        "in the student group.\n\n"
        "Please choose an option:",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )


# ============================================================
# SELECT TYPE
# ============================================================

async def select_type(update: Update, context: ContextTypes.DEFAULT_TYPE):

    query = update.callback_query
    await query.answer()

    context.user_data["type"] = query.data

    await query.message.reply_text(
        "🏫 Please enter your department.\n\n"
        "Example:\n"
        "Electrical Engineering"
    )


# ============================================================
# RECEIVE STUDENT INFORMATION
# ============================================================

async def receive_message(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if not update.message:
        return

    text = update.message.text.strip()

    # --------------------------------------------------------
    # DEPARTMENT
    # --------------------------------------------------------

    if "department" not in context.user_data:

        context.user_data["department"] = text

        await update.message.reply_text(
            "🎓 Please enter your year of study.\n\n"
            "Example:\n"
            "3rd Year"
        )

        return

    # --------------------------------------------------------
    # YEAR
    # --------------------------------------------------------

    if "year" not in context.user_data:

        context.user_data["year"] = text

        await update.message.reply_text(
            "✍️ Please write your feedback, "
            "suggestion, or complaint.\n\n"
            "You can write as much detail as necessary."
        )

        return

    # --------------------------------------------------------
    # FEEDBACK MESSAGE
    # --------------------------------------------------------

    if "message" not in context.user_data:

        context.user_data["message"] = text

        type_names = {
            "feedback": "📝 Feedback",
            "suggestion": "💡 Suggestion",
            "complaint": "⚠️ Complaint"
        }

        submission_type = type_names.get(
            context.user_data["type"],
            "Submission"
        )

        keyboard = [
            [
                InlineKeyboardButton(
                    "✅ Submit",
                    callback_data="submit"
                )
            ],
            [
                InlineKeyboardButton(
                    "✏️ Edit",
                    callback_data="edit"
                )
            ]
        ]

        confirmation = (
            "🔍 Please check your submission:\n\n"
            f"Type: {submission_type}\n"
            f"Department: {context.user_data['department']}\n"
            f"Year: {context.user_data['year']}\n\n"
            f"Message:\n{text}\n\n"
            "Do you want to submit it?"
        )

        await update.message.reply_text(
            confirmation,
            reply_markup=InlineKeyboardMarkup(keyboard)
        )


# ============================================================
# SUBMIT / EDIT
# ============================================================

async def submit(update: Update, context: ContextTypes.DEFAULT_TYPE):

    query = update.callback_query
    await query.answer()

    action = query.data

    # --------------------------------------------------------
    # EDIT
    # --------------------------------------------------------

    if action == "edit":

        context.user_data.pop("message", None)

        await query.message.reply_text(
            "✏️ Please write your message again:"
        )

        return

    # --------------------------------------------------------
    # SUBMIT
    # --------------------------------------------------------

    if action != "submit":
        return

    global data

    reference = f"FB-{data['next_number']:04d}"

    data["next_number"] += 1

    type_names = {
        "feedback": "📝 Feedback",
        "suggestion": "💡 Suggestion",
        "complaint": "⚠️ Complaint"
    }

    submission_type = type_names.get(
        context.user_data["type"],
        "Submission"
    )

    department = context.user_data["department"]
    year = context.user_data["year"]
    message = context.user_data["message"]

    # --------------------------------------------------------
    # GET USERNAME
    # --------------------------------------------------------

    user = update.effective_user

    if user.username:
        username = "@" + user.username
    else:
        username = "No username"

    # --------------------------------------------------------
    # SAVE DATA
    #
    # Username is stored here so the administrator can identify
    # the sender if necessary.
    # --------------------------------------------------------

    submission = {
        "reference": reference,
        "type": submission_type,
        "username": username,
        "department": department,
        "year": year,
        "message": message,
        "date": datetime.now().strftime(
            "%Y-%m-%d %H:%M:%S"
        )
    }

    data["submissions"].append(submission)

    save_data(data)

    # ========================================================
    # POST TO GROUP
    #
    # IMPORTANT:
    # USERNAME IS NOT INCLUDED HERE
    # ========================================================

    group_message = (
        "📨 ANONYMOUS STUDENT SUBMISSION\n\n"
        f"Reference: #{reference}\n"
        f"Type: {submission_type}\n"
        f"Department: {department}\n"
        f"Year: {year}\n\n"
        "Message:\n"
        f"{message}\n\n"
        "────────────────────\n"
        "🤖 Student Feedback Bot"
    )

    try:

        await context.bot.send_message(
            chat_id=GROUP_CHAT_ID,
            text=group_message
        )

    except Exception as error:

        print("Group sending error:", error)


    # ========================================================
    # SEND INFORMATION TO ADMIN
    # ========================================================

    admin_message = (
        "🔐 ADMIN NOTIFICATION\n\n"
        f"Reference: #{reference}\n"
        f"Telegram username: {username}\n"
        f"Type: {submission_type}\n"
        f"Department: {department}\n"
        f"Year: {year}\n\n"
        "Student message:\n"
        f"{message}\n\n"
        f"Submitted: {submission['date']}"
    )

    try:

        await context.bot.send_message(
            chat_id=ADMIN_CHAT_ID,
            text=admin_message
        )

    except Exception as error:

        print("Admin sending error:", error)


    # ========================================================
    # STUDENT CONFIRMATION
    # ========================================================

    keyboard = [
        [
            InlineKeyboardButton(
                "➕ Submit Another",
                callback_data="another"
            )
        ],
        [
            InlineKeyboardButton(
                "🏠 Main Menu",
                callback_data="menu"
            )
        ]
    ]

    await query.message.reply_text(
        "✅ Your submission has been received.\n\n"
        f"Reference: #{reference}\n\n"
        "Your submission was posted anonymously "
        "to the group.",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

    # Clear current submission
    context.user_data.clear()


# ============================================================
# SUBMIT ANOTHER
# ============================================================

async def another(update: Update, context: ContextTypes.DEFAULT_TYPE):

    query = update.callback_query
    await query.answer()

    context.user_data.clear()

    keyboard = [
        [
            InlineKeyboardButton(
                "📝 Feedback",
                callback_data="feedback"
            )
        ],
        [
            InlineKeyboardButton(
                "💡 Suggestion",
                callback_data="suggestion"
            )
        ],
        [
            InlineKeyboardButton(
                "⚠️ Complaint",
                callback_data="complaint"
            )
        ]
    ]

    await query.message.reply_text(
        "➕ Submit another message.\n\n"
        "Choose the type:",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )


# ============================================================
# MAIN MENU
# ============================================================

async def menu(update: Update, context: ContextTypes.DEFAULT_TYPE):

    query = update.callback_query
    await query.answer()

    context.user_data.clear()

    keyboard = [
        [
            InlineKeyboardButton(
                "📝 Feedback",
                callback_data="feedback"
            )
        ],
        [
            InlineKeyboardButton(
                "💡 Suggestion",
                callback_data="suggestion"
            )
        ],
        [
            InlineKeyboardButton(
                "⚠️ Complaint",
                callback_data="complaint"
            )
        ]
    ]

    await query.message.reply_text(
        "🏠 Main Menu\n\n"
        "Choose an option:",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )


# ============================================================
# ERROR HANDLER
# ============================================================

async def error_handler(update, context):

    print("Error:", context.error)


# ============================================================
# RUN BOT
# ============================================================

def main():

    if BOT_TOKEN == "YOUR_BOT_TOKEN_HERE":

        print("❌ Please enter your bot token.")
        return

    application = (
        Application.builder()
        .token(BOT_TOKEN)
        .build()
    )

    # Start
    application.add_handler(
        CommandHandler("start", start)
    )

    # Feedback / Suggestion / Complaint
    application.add_handler(
        CallbackQueryHandler(
            select_type,
            pattern="^(feedback|suggestion|complaint)$"
        )
    )

    # Submit / Edit
    application.add_handler(
        CallbackQueryHandler(
            submit,
            pattern="^(submit|edit)$"
        )
    )

    # Another submission
    application.add_handler(
        CallbackQueryHandler(
            another,
            pattern="^another$"
        )
    )

    # Main menu
    application.add_handler(
        CallbackQueryHandler(
            menu,
            pattern="^menu$"
        )
    )

    # Text messages
    application.add_handler(
        MessageHandler(
            filters.TEXT & ~filters.COMMAND,
            receive_message
        )
    )

    application.add_error_handler(error_handler)

    print("🤖 Student Feedback Bot is running...")

    application.run_polling()


# ============================================================
# START
# ============================================================

if __name__ == "__main__":
    main()
