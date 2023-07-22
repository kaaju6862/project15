import logging
from telegram import Update, ParseMode
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters, ConversationHandler, CallbackContext

# Replace 'YOUR_BOT_TOKEN' with the token you obtained from the BotFather
TOKEN = '6648183065:AAG0YneFoI746robQ6FfPd0Hj4H54pdHyZY'

# Set up logging
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

# Dictionary to store the last message from each user
user_last_messages = {}

# Conversation states for the /reply command
SELECT_USER, SELECT_MESSAGE = range(2)

# Function to handle /start command
def start(update: Update, _: CallbackContext) -> None:
    update.message.reply_text("Hello! I'm your admin bot. I'll let you know when I'm online.")

# Function to handle regular messages
def reply_online(update: Update, _: CallbackContext) -> None:
    # Check if the message is from the admin
    admin_id = '5788082596'  # Replace with the admin's Telegram user ID
    if str(update.message.from_user.id) == admin_id:
        update.message.reply_text("I'm online and ready to respond!")
        return

    # Store the user's last message
    user_id = str(update.message.from_user.id)
    user_last_messages[user_id] = update.message.text

# Function to handle /reply command
def reply(update: Update, _: CallbackContext) -> int:
    # Check if the message is from the admin
    admin_id = '5788082596'  # Replace with the admin's Telegram user ID
    if str(update.message.from_user.id) == admin_id:
        # Check if there are any users with messages stored
        if not user_last_messages:
            update.message.reply_text("No messages from users to reply to.")
            return ConversationHandler.END

        # Ask the admin to select a user to reply to
        user_list = "\n".join([f"{user_id}: {user_last_messages[user_id]}" for user_id in user_last_messages])
        update.message.reply_text(f"Select a user ID to reply:\n{user_list}")
        return SELECT_USER
    else:
        return ConversationHandler.END

def select_user(update: Update, _: CallbackContext) -> int:
    user_id = update.message.text.strip()

    # Check if the selected user ID is valid
    if user_id in user_last_messages:
        # Store the selected user ID for the reply
        user_id = update.message.text.strip()
        context = _.user_data
        context["selected_user_id"] = user_id

        # Ask the admin to enter the reply message
        update.message.reply_text("Enter your reply message:")
        return SELECT_MESSAGE
    else:
        # Invalid user ID, ask again
        update.message.reply_text("Invalid user ID. Please select a valid user ID.")
        return SELECT_USER

def select_message(update: Update, _: CallbackContext) -> int:
    # Get the selected user ID and reply message
    user_id = _.user_data.get("selected_user_id")
    reply_message = update.message.text.strip()

    # Send the reply to the selected user
    update.message.bot.send_message(chat_id=user_id, text=reply_message)

    # Inform the admin that the reply was sent
    update.message.reply_text("Reply sent successfully!")

    # Clear the selected user data
    _.user_data.clear()
    return ConversationHandler.END

def cancel(update: Update, _: CallbackContext) -> int:
    # Clear the selected user data
    _.user_data.clear()
    update.message.reply_text("Reply canceled.")
    return ConversationHandler.END

# Main function to run the bot
def main():
    # Create the Updater and pass it your bot's token.
    updater = Updater(TOKEN, use_context=True)

    # Get the dispatcher to register handlers
    dp = updater.dispatcher

    # Register command and message handlers
    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(ConversationHandler(
        entry_points=[CommandHandler("reply", reply)],
        states={
            SELECT_USER: [MessageHandler(Filters.text & ~Filters.command, select_user)],
            SELECT_MESSAGE: [MessageHandler(Filters.text & ~Filters.command, select_message)],
        },
        fallbacks=[CommandHandler("cancel", cancel)],
    ))
    dp.add_handler(MessageHandler(Filters.text & ~Filters.command, reply_online))

    # Start the Bot
    updater.start_polling()

    # Run the bot until the user presses Ctrl-C
    updater.idle()

if __name__ == '__main__':
    main()
