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
    KeyboardButton("\ud83d\udcda TOPIK 1"),
    KeyboardButton("\ud83d\udcda TOPIK 2"),
    KeyboardButton("\ud83d\udcd6 \uc11c\uc6b8\ub300 \ud55c\uad6d\uc5b4 kitoblar"),
    KeyboardButton("\u2600\ufe0f Harflar"),
    KeyboardButton("\ud83d\udc8e Premium darslar")
)

@dp.message_handler(commands=["start"])
async def start_handler(message: types.Message):
    await message.answer(
        "Assalomu alaykum!\n\ud55c\uad6d\uc5b4 o\u2018rgatadigan botga xush kelibsiz.\nQuyidagi menydan foydalaning:",
        reply_markup=main_menu
    )

# === Harflar ===
@dp.message_handler(lambda message: message.text == "\u2600\ufe0f Harflar")
async def show_letter_menu(message: types.Message):
    markup = InlineKeyboardMarkup(row_width=4)
    for harf in hangeul_letters_data.keys():
        markup.insert(InlineKeyboardButton(harf, callback_data=f"harf_{harf}"))
    markup.add(InlineKeyboardButton("\u2b05\ufe0f Orqaga", callback_data="back_to_main"))
    await message.answer("Quyidagi harflardan birini tanlang:", reply_markup=markup)

@dp.callback_query_handler(lambda c: c.data.startswith("harf_"))
async def show_letter_info(callback: types.CallbackQuery):
    harf = callback.data.replace("harf_", "")
    matn = hangeul_letters_data.get(harf, "Ma’lumot topilmadi")
    markup = InlineKeyboardMarkup().add(InlineKeyboardButton("\u2b05\ufe0f Orqaga", callback_data="back_to_letters"))
    await callback.message.edit_text(f"\u2600\ufe0f {harf}\n{matn}", reply_markup=markup)
    await callback.answer()

@dp.callback_query_handler(lambda c: c.data == "back_to_letters")
async def back_to_letters(callback: types.CallbackQuery):
    markup = InlineKeyboardMarkup(row_width=4)
    for harf in hangeul_letters_data.keys():
        markup.insert(InlineKeyboardButton(harf, callback_data=f"harf_{harf}"))
    markup.add(InlineKeyboardButton("\u2b05\ufe0f Orqaga", callback_data="back_to_main"))
    await callback.message.edit_text("Quyidagi harflardan birini tanlang:", reply_markup=markup)
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

@dp.message_handler(lambda message: message.text.startswith("\ud83d\udcd6 \uc11c\uc6b8\ub300 \ud55c\uad6d\uc5b4"))
async def show_books(message: types.Message):
    markup = InlineKeyboardMarkup(row_width=2)
    for level in grammar_all:
        markup.insert(InlineKeyboardButton(level, callback_data=f"book_{level}"))
    markup.add(InlineKeyboardButton("\u2b05\ufe0f Orqaga", callback_data="back_to_main"))
    await message.answer("Qaysi kitobni tanlaysiz?", reply_markup=markup)

@dp.callback_query_handler(lambda c: c.data.startswith("book_"))
async def show_grammar_menu(callback: types.CallbackQuery):
    book = callback.data.replace("book_", "")
    grammars = grammar_all.get(book, {})
    markup = InlineKeyboardMarkup(row_width=1)
    for key in grammars:
        markup.add(InlineKeyboardButton(key, callback_data=key))
    markup.add(InlineKeyboardButton("\u2b05\ufe0f Orqaga", callback_data="show_books_menu"))
    await callback.message.edit_text(f"{book} grammatikalaridan birini tanlang:", reply_markup=markup)
    await callback.answer()

@dp.callback_query_handler(lambda c: c.data in sum([list(g.keys()) for g in grammar_all.values()], []))
async def show_grammar(callback: types.CallbackQuery):
    key = callback.data
    all_grammars = {k: v for d in grammar_all.values() for k, v in d.items()}
    text = all_grammars.get(key, "Ma’lumot topilmadi")
    back_map = {k: f"book_{level}" for level, g in grammar_all.items() for k in g}
    book_code = back_map.get(key, "show_books_menu")
    markup = InlineKeyboardMarkup().add(InlineKeyboardButton("\u2b05\ufe0f Orqaga", callback_data=book_code))
    await callback.message.edit_text(text, reply_markup=markup)
    await callback.answer()

@dp.callback_query_handler(lambda c: c.data == "show_books_menu")
async def show_books_menu(callback: types.CallbackQuery):
    await show_books(callback.message)
    await callback.answer()

# === TOPIK 1 ===
@dp.message_handler(lambda message: message.text == "\ud83d\udcda TOPIK 1")
async def topik1_handler(message: types.Message):
    await message.reply(
        f"\ud83d\udcd8 TOPIK 1 sayohatiga xush kelibsiz!\n"
        f"Bu yerda asoslar mustahkamlanadi, kelajakdagi yutuqlaringiz shu yerda boshlanadi! \ud83d\udcaa\n\n"
        f"\ud83d\ude80 Boshlash: {TOPIK_LINK}",
        disable_web_page_preview=True
    )

# === TOPIK 2 ===
@dp.message_handler(lambda message: message.text == "\ud83d\udcda TOPIK 2")
async def topik2_handler(message: types.Message):
    await message.reply(
        f"\ud83d\udcda Siz endi TOPIK 2 \"jang maydoni\"dasiz!\n"
        f"Tayyor bo‘ling — bilimlar hujumi boshlanmoqda \ud83d\ude04\n\n"
        f"\ud83d\ude80 Qo‘shiling: {TOPIK2_LINK}",
        disable_web_page_preview=True
    )

# === Premium ===
@dp.message_handler(lambda message: message.text == "\ud83d\udc8e Premium darslar")
async def premium_info(message: types.Message):
    text = (
        "\ud83d\udc8e PREMIUM DARS TARIFI\n\n"
        "\ud83d\udccc Imkoniyatlar:\n"
        "\ud83d\udd39 Har ikki kunda jonli dars\n"
        "\ud83d\udd39 Yopiq premium materiallar\n"
        "\ud83d\udd39 0 dan koreys tilini o‘rganish\n"
        "\ud83d\udd39 Savol-javoblar uchun guruh\n\n"
        "\ud83d\udcb0 Narxi: 30 000 so‘m / oy\n"
        "\ud83d\udcb3 To‘lov karta:\n5614 6818 1030 9850\n\n"
        "\ud83d\uddd3 To‘lov cheki bilan 'PREMIUM' deb yuboring!"
    )
    await message.answer(text)

@dp.message_handler(content_types=types.ContentType.PHOTO)
async def handle_check(message: types.Message):
    if message.caption and "premium" in message.caption.lower():
        await bot.send_message(ADMIN_ID, f"\ud83d\udcb3 Yangi premium foydalanuvchi:\n\ud83d\udc64 {message.from_user.full_name}\n\ud83c\udd94 {message.from_user.id}")
        await bot.send_photo(ADMIN_ID, message.photo[-1].file_id, caption=message.caption)
        await message.reply(f"\u2705 Chek qabul qilindi!\nGuruh: {PREMIUM_GROUP_LINK}")
    else:
        await message.reply("\u2757 Iltimos, captionda 'PREMIUM' deb yozing.")

# === Orqaga ===
@dp.callback_query_handler(lambda c: c.data == "back_to_main")
async def back_to_main(callback: types.CallbackQuery):
    await bot.send_message(callback.from_user.id, "\u2b05\ufe0f Asosiy menyu:", reply_markup=main_menu)
    await callback.message.delete()
    await callback.answer()

# === Webhook ===
async def on_startup(dp):
    await bot.set_webhook(WEBHOOK_URL)
    print("\u2705 Webhook o‘rnatildi:", WEBHOOK_URL)

async def on_shutdown(dp):
    print("\u274c Webhook o‘chirildi")

if __name__ == '__main__':
    start_webhook(
        dispatcher=dp,
        webhook_path=WEBHOOK_PATH,
        on_startup=on_startup,
        on_shutdown=on_shutdown,
        skip_updates=True,
        host=WEBAPP_HOST,
        port=WEBAPP_PORT,
    )
