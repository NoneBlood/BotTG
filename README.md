from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application, CommandHandler, MessageHandler, CallbackQueryHandler,
    ConversationHandler, ContextTypes, filters
)
from telegram import Update
from telegram.ext import Application, MessageHandler, filters, ContextTypes

async def get_file_id(update: Update, context: ContextTypes.DEFAULT_TYPE):
    print(update.message)

# Этапы анкеты
NAME, YEARS, ORDERS, TURNOVER, CONFIRM = range(5)
SUPPORT_MESSAGE = range(1)

ADMIN_CHAT_ID = 8030522072

# Главное меню
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [
        [InlineKeyboardButton("💼 Услуги", callback_data='services'),
         InlineKeyboardButton("📊 Заполнить анкету", callback_data='survey')],
        [InlineKeyboardButton("📞 Поддержка", callback_data='support')],
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    if update.message:
        await update.message.reply_text("👋 Добро пожаловать!\n\nВыберите раздел:", reply_markup=reply_markup)
    elif update.callback_query:
        await update.callback_query.edit_message_text("👋 Добро пожаловать!\n\nВыберите раздел:", reply_markup=reply_markup)

# Главное меню обработчик
async def main_menu_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    if query.data == 'services':
        await query.edit_message_text("📦 Ведение учета Wildberries, Ozon, УСН.", reply_markup=back_to_menu())
    elif query.data == 'survey':
        await query.edit_message_text("👤 Введите ваше ФИО:", reply_markup=back_to_menu())
        return NAME
    elif query.data == 'support':
        await query.edit_message_text("🛠 Опишите проблему и приложите файлы:", reply_markup=back_to_menu())
        return SUPPORT_MESSAGE
    elif query.data == 'back_to_main':
        await start(update, context)

# Кнопка назад
def back_to_menu():
    return InlineKeyboardMarkup([[InlineKeyboardButton("🔙 Назад", callback_data='back_to_main')]])

# Анкета — шаг 1: ФИО
async def get_name(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['name'] = update.message.text
    await update.message.reply_text("📅 Сколько лет ведете деятельность?", reply_markup=back_to_menu())
    return YEARS

# Анкета — шаг 2: Стаж
async def get_years(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['years'] = update.message.text
    await update.message.reply_text("📦 Сколько заказов в среднем в месяц?", reply_markup=back_to_menu())
    return ORDERS

# Анкета — шаг 3: Заказы
async def get_orders(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['orders'] = update.message.text
    await update.message.reply_text("💰 Какой средний оборот в месяц?", reply_markup=back_to_menu())
    return TURNOVER

# Анкета — шаг 4: Оборот
async def get_turnover(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['turnover'] = update.message.text

    summary = (
        f"📋 Ваша анкета:\n"
        f"👤 ФИО: {context.user_data['name']}\n"
        f"📅 Стаж: {context.user_data['years']} лет\n"
        f"📦 Заказов: {context.user_data['orders']}\n"
        f"💰 Оборот: {context.user_data['turnover']}"
    )

    keyboard = [
        [InlineKeyboardButton("✅ Всё верно — отправить", callback_data='confirm_send')],
        [InlineKeyboardButton("🔙 Назад в меню", callback_data='back_to_main')]
    ]
    await update.message.reply_text(summary + "\n\nОтправить анкету?", reply_markup=InlineKeyboardMarkup(keyboard))
    return CONFIRM

# Финальное подтверждение
async def confirm_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    user = query.from_user
    full_summary = (
        f"📥 Новая анкета от пользователя @{user.username or 'нет username'} (ID: {user.id}):\n\n"
        f"👤 ФИО: {context.user_data['name']}\n"
        f"📅 Стаж: {context.user_data['years']} лет\n"
        f"📦 Заказов: {context.user_data['orders']}\n"
        f"💰 Оборот: {context.user_data['turnover']}"
    )

    await query.edit_message_text("✅ Анкета отправлена!")
    await context.bot.send_message(chat_id=ADMIN_CHAT_ID, text=full_summary)
    return ConversationHandler.END

# Поддержка
async def support_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.message.from_user
    text = f"🛠 Поддержка от @{user.username or 'нет username'} (ID: {user.id}):\n\n{update.message.text}"

    await context.bot.send_message(chat_id=ADMIN_CHAT_ID, text=text)

    if update.message.photo:
        photo_file = update.message.photo[-1].file_id
        await context.bot.send_photo(chat_id=ADMIN_CHAT_ID, photo=photo_file)

    await update.message.reply_text("✅ Ваше сообщение передано.", reply_markup=back_to_menu())
    return ConversationHandler.END

# Полный запуск
def run_bot():
    TOKEN = "7969450778:AAErcio6CDxgebmGBxhHpXGRfaw1USmYnNs"

    app = Application.builder().token(TOKEN).build()

    survey_conv = ConversationHandler(
        entry_points=[CallbackQueryHandler(main_menu_handler, pattern='^survey$')],
        states={
            NAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_name)],
            YEARS: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_years)],
            ORDERS: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_orders)],
            TURNOVER: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_turnover)],
            CONFIRM: [CallbackQueryHandler(confirm_handler, pattern='^confirm_send$')]
        },
        fallbacks=[]
    )

    support_conv = ConversationHandler(
        entry_points=[CallbackQueryHandler(main_menu_handler, pattern='^support$')],
        states={
            SUPPORT_MESSAGE: [MessageHandler(filters.ALL, support_handler)],
        },
        fallbacks=[]
    )

    app.add_handler(CommandHandler("start", start))
    app.add_handler(survey_conv)
    app.add_handler(support_conv)
    app.add_handler(CallbackQueryHandler(main_menu_handler))

    app.run_polling()

if __name__ == '__main__':
    run_bot()
