import asyncio
from aiogram import Bot, Dispatcher, types
from aiogram.filters import Command
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.exceptions import TelegramForbiddenError

TOKEN = "7915419040:AAHT2RApfMdfwQCJL9fKrLLiaXBUP4qr1m4"
CHANNEL_USERNAME = "@kosmin_inc"
ARTICLE_URL = "https://dzen.ru/a/Z7sfm1cR20lEV2zA"  # Замените на реальную ссылку

bot = Bot(token=TOKEN)
dp = Dispatcher()

# Кнопки
subscribe_button = InlineKeyboardButton(
    text="Подписаться на канал",
    url=f"https://t.me/{CHANNEL_USERNAME[1:]}"
)

check_button = InlineKeyboardButton(
    text="Проверить подписку",
    callback_data="check_subscription"
)

article_button = InlineKeyboardButton(
    text="📖 Перейти на статью",
    url=ARTICLE_URL
)


async def is_subscribed(user_id: int) -> bool:
    """Проверяет подписку на канал"""
    try:
        member = await bot.get_chat_member(chat_id=CHANNEL_USERNAME, user_id=user_id)
        return member.status in ["member", "administrator", "creator"]
    except TelegramForbiddenError:
        return False


async def send_article_message(chat_id: int):
    """Отправляет сообщение со статьей через 5 сек после проверки"""
    await asyncio.sleep(5)
    keyboard = InlineKeyboardMarkup(inline_keyboard=[[article_button]])
    await bot.send_message(
        chat_id=chat_id,
        text="Ещё у меня есть статья, в которой я рассказываю, как легко можно зарабатывать на монтаже видео",
        reply_markup=keyboard
    )


async def handle_subscription(chat_id: int, user_id: int):
    """Основная логика проверки подписки"""
    await asyncio.sleep(3)
    if await is_subscribed(user_id):
        await bot.send_document(
            chat_id=chat_id,
            document=types.FSInputFile("kosmin.inc.pdf"),
            caption="Спасибо за подписку! Вот ваш PDF файл:"
        )
        # Запускаем отложенное сообщение со статьей
        asyncio.create_task(send_article_message(chat_id))
    else:
        keyboard = InlineKeyboardMarkup(inline_keyboard=[
            [subscribe_button],
            [check_button]
        ])
        await bot.send_message(
            chat_id=chat_id,
            text="Хочешь забрать файл с визуальными хуками, которые пробивают баннерную слепоту? \n\nПодпишись, пожалуйста, и я тебе его отправлю!",
            reply_markup=keyboard
        )


@dp.message(Command("start"))
async def start_command(message: types.Message):
    """Приветственное сообщение без кнопок"""
    await message.answer_photo(
        photo=types.FSInputFile("image.png"),
        caption="Привет! 👋\n\nЯ Игорь, видеомонтажер с 7-летним опытом работы с топовыми блогерами и крупными компаниями."
    )

    # Запускаем проверку подписки
    asyncio.create_task(handle_subscription(
        chat_id=message.chat.id,
        user_id=message.from_user.id
    ))


@dp.callback_query(lambda call: call.data == "check_subscription")
async def check_subscription(call: types.CallbackQuery):
    """Проверка подписки по кнопке"""
    if await is_subscribed(call.from_user.id):
        await call.message.delete()
        await call.message.answer_document(
            document=types.FSInputFile("kosmin.inc.pdf"),
            caption="Спасибо за подписку! Вот твой PDF"
        )
        # Запускаем отложенное сообщение со статьей
        asyncio.create_task(send_article_message(call.message.chat.id))
    else:
        await call.answer("Наебать не получится, подписывайся", show_alert=True)


async def main():
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
