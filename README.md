# Shexsi-destek
Ai Shexsi-Destek Bot
import logging
import asyncio
from aiogram import Bot, Dispatcher, types
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton
from aiogram.fsm.storage.memory import MemoryStorage
from openai import AsyncOpenAI
import os

BOT_TOKEN = os.getenv("BOT_TOKEN")
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")

device_lang = {}
user_state = {}

openai = AsyncOpenAI(api_key=OPENAI_API_KEY)
bot = Bot(token=BOT_TOKEN)
dp = Dispatcher(storage=MemoryStorage())

lang_keyboard = ReplyKeyboardMarkup(
    keyboard=[[KeyboardButton(text="🇷🇺 Русский"), KeyboardButton(text="🇦🇿 Azərbaycan")]],
    resize_keyboard=True
)

def get_response_template(lang: str) -> str:
    if lang == "ru":
        return "Ты можешь поделиться, что тебя тревожит? Я рядом."
    if lang == "az":
        return "Nə narahat edir səni? Mən burdayam."
    return "Choose a language first."

@dp.message(commands=["start"])
async def cmd_start(message: types.Message):
    await message.answer("Выберите язык / Dil seçin:", reply_markup=lang_keyboard)

@dp.message(lambda message: message.text in ["🇷🇺 Русский", "🇦🇿 Azərbaycan"])
async def set_language(message: types.Message):
    user_id = message.from_user.id
    lang = "ru" if "Русский" in message.text else "az"
    device_lang[user_id] = lang
    await message.answer("Напишите, что вас беспокоит..." if lang == "ru" else "Sizi narahat edən nədir...")

@dp.message()
async def handle_message(message: types.Message):
    user_id = message.from_user.id
    lang = device_lang.get(user_id)
    if not lang:
        await cmd_start(message)
        return

    prompt = f"Пользователь пишет на {lang}:
{message.text}
Ответь как психолог поддерживающе."
    try:
        response = await openai.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
            max_tokens=500,
        )
        reply = response.choices[0].message.content
        await message.answer(reply)
    except Exception as e:
        logging.exception(e)
        await message.answer(get_response_template(lang))

async def main():
    logging.basicConfig(level=logging.INFO)
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
