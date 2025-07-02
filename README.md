import logging
import os
from aiogram import Bot, Dispatcher, types
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton, ReplyKeyboardMarkup, KeyboardButton
from aiogram.utils.executor import start_webhook
from dotenv import load_dotenv

from data.hangeul import hangeul_letters_data
from data.grammar import grammar_1A, grammar_1B, grammar_2A, grammar_2B, grammar_3A, grammar_3B, grammar_4A, grammar_4B, grammar_5A, grammar_5B, grammar_6A, grammar_6B

load_dotenv()

API_TOKEN = os.getenv("BOT_TOKEN")
ADMIN_ID = int(os.getenv("ADMIN_ID"))
PREMIUM_GROUP_LINK = os.getenv("PREMIUM_LINK")
TOPIK_LINK = os.getenv("TOPIK_LINK")
TOPIK2_LINK = os.getenv("TOPIK2_LINK")

WEBHOOK_HOST = f"https://{os.getenv('RENDER_EXTERNAL_HOSTNAME')}"
WEBHOOK_PATH = "/webhook"
WEBHOOK_URL = f"{WEBHOOK_HOST}{WEBHOOK_PATH}"
WEBAPP_HOST = "0.0.0.0"
WEBAPP_PORT = int(os.getenv("PORT", 8000))

logging.basicConfig(level=logging.INFO)

bot = Bot(token=API_TOKEN)
dp = Dispatcher(bot)

# === Asosiy menyu ===
main_menu = ReplyKeyboardMarkup(resize_keyboard=True)
main_menu.add(
    KeyboardButton("📘 TOPIK 1"),
    KeyboardButton("📙 TOPIK 2"),
    KeyboardButton("📚 서울대 한국어 kitoblar"),
    KeyboardButton("☀️ Harflar"),
    KeyboardButton("💎 Premium darslar")
)

@dp.message_handler(commands=["start"])
async def start_handler(message: types.Message):
    await message.answer(
        "👋 Assalomu alaykum!\n🇰🇷 Koreys tilini o‘rgatadigan botga xush kelibsiz.\n👇 Quyidagi menydan foydalaning:",
        reply_markup=main_menu
    )

# === Harflar ===
@dp.message_handler(lambda message: message.text == "☀️ Harflar")
async def show_letter_menu(message: types.Message):
    markup = InlineKeyboardMarkup(row_width=4)
    for harf in hangeul_letters_data.keys():
        markup.insert(InlineKeyboardButton(harf, callback_data=f"harf_{harf}"))
    markup.add(InlineKeyboardButton("⬅️ Orqaga", callback_data="back_to_main"))
    await message.answer("🔤 Quyidagi harflardan birini tanlang:", reply_markup=markup)

@dp.callback_query_handler(lambda c: c.data.startswith("harf_"))
async def show_letter_info(callback: types.CallbackQuery):
    harf = callback.data.replace("harf_", "")
    matn = hangeul_letters_data.get(harf, "ℹ️ Ma’lumot topilmadi")
    markup = InlineKeyboardMarkup().add(InlineKeyboardButton("⬅️ Orqaga", callback_data="back_to_letters"))
    await callback.message.edit_text(f"☀️ {harf}\n{matn}", reply_markup=markup)
    await callback.answer()

@dp.callback_query_handler(lambda c: c.data == "back_to_letters")
async def back_to_letters(callback: types.CallbackQuery):
    markup = InlineKeyboardMarkup(row_width=4)
    for harf in hangeul_letters_data.keys():
        markup.insert(InlineKeyboardButton(harf, callback_data=f"harf_{harf}"))
    markup.add(InlineKeyboardButton("⬅️ Orqaga", callback_data="back_to_main"))
    await callback.message.edit_text("🔤 Quyidagi harflardan birini tanlang:", reply_markup=markup)
    await callback.answer()

# === Grammatikalar ===
grammar_all = {
    "1A": grammar_1A,
    "1B": grammar_1B,
    "2A": grammar_2A,
    "2B": grammar_2B,
    "3A": grammar_3A,
    "3B": grammar_3B,
    "4A": grammar_4A,
    "4B": grammar_4B,
    "5A": grammar_5A,
    "5B": grammar_5B,
    "6A": grammar_6A,
    "6B": grammar_6B
}

# Barcha grammatik tugmalarni yig‘ish
all_grammar_keys = set().union(*[g.keys() for g in grammar_all.values()])

@dp.callback_query_handler(lambda c: c.data in all_grammar_keys)
async def handle_grammar(callback: types.CallbackQuery):
    key = callback.data
    for level, grammars in grammar_all.items():
        if key in grammars:
            text = f"📘 <b>{level}</b> - <b>{key}</b>\n\n{grammars[key]}"
            markup = InlineKeyboardMarkup().add(InlineKeyboardButton("⬅️ Orqaga", callback_data=f"back_to_{level}"))
            await callback.message.edit_text(text, parse_mode="HTML", reply_markup=markup)
            await callback.answer()
            break

# Har bir darajaga kirish tugmalari va orqaga tugmasi
def grammar_menu(level):
    markup = InlineKeyboardMarkup(row_width=2)
    for title in grammar_all[level].keys():
        markup.insert(InlineKeyboardButton(title, callback_data=title))
    markup.add(InlineKeyboardButton("⬅️ Orqaga", callback_data="back_to_main"))
    return markup

@dp.message_handler(lambda message: message.text == "📚 서울대 한국어 kitoblar")
async def show_grammar_levels(message: types.Message):
    markup = InlineKeyboardMarkup(row_width=3)
    for level in grammar_all.keys():
        markup.insert(InlineKeyboardButton(level, callback_data=f"level_{level}"))
    markup.add(InlineKeyboardButton("⬅️ Orqaga", callback_data="back_to_main"))
    await message.answer("📚 Qaysi darajadagi grammatikani tanlaysiz?", reply_markup=markup)

@dp.callback_query_handler(lambda c: c.data.startswith("level_"))
async def handle_level(callback: types.CallbackQuery):
    level = callback.data.replace("level_", "")
    markup = grammar_menu(level)
    await callback.message.edit_text(f"📘 {level} darajadagi grammatikalar:", reply_markup=markup)
    await callback.answer()

@dp.callback_query_handler(lambda c: c.data.startswith("back_to_"))
async def back_to_grammar_level(callback: types.CallbackQuery):
    level = callback.data.replace("back_to_", "")
    markup = grammar_menu(level)
    await callback.message.edit_text(f"📘 {level} darajadagi grammatikalar:", reply_markup=markup)
    await callback.answer()

@dp.callback_query_handler(lambda c: c.data == "back_to_main")
async def back_to_main(callback: types.CallbackQuery):
    await callback.message.answer("🏠 Asosiy menyuga qaytdingiz.", reply_markup=main_menu)
    await callback.answer()

# === Webhook sozlamalari ===
async def on_startup(dp):
    await bot.set_webhook(WEBHOOK_URL)

async def on_shutdown(dp):
    logging.warning("Bot to‘xtatildi!")

if __name__ == '__main__':
    from aiogram import executor
    executor.start_webhook(
        dispatcher=dp,
        webhook_path=WEBHOOK_PATH,
        on_startup=on_startup,
        on_shutdown=on_shutdown,
        skip_updates=True,
        host=WEBAPP_HOST,
        port=WEBAPP_PORT
    )
