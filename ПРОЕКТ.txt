import asyncio
import logging
from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import CommandStart, Command
from aiogram.methods import DeleteWebhook
from aiogram.types import Message, ReplyKeyboardMarkup, KeyboardButton
from aiogram.utils.keyboard import ReplyKeyboardBuilder
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
import requests
from collections import defaultdict
from datetime import datetime, time
import pytz

TOKEN = '7826734862:AAFkDgHXtQKgoFY8FV2An2wti46GoRZp2GA'

logging.basicConfig(level=logging.INFO)
bot = Bot(TOKEN)
dp = Dispatcher()

# Храним историю диалогов для каждого пользователя (user_id: list[messages])
user_dialogs = defaultdict(list)

# Храним настройки погоды для пользователей (user_id: {'city': str, 'time': time})
weather_subscriptions = defaultdict(dict)


# Состояния для FSM
class WeatherStates(StatesGroup):
    waiting_for_city = State()
    waiting_for_time = State()


# Команда /start — главное меню
@dp.message(Command("start"))
async def cmd_start(message: types.Message):
    user_id = message.from_user.id
    user_dialogs[user_id] = [  # Начинаем новый диалог
        {"role": "system", "content": "You are a helpful assistant."}
    ]

    builder = ReplyKeyboardBuilder()
    builder.add(KeyboardButton(text="💬 Чат с ассистентом"))
    builder.add(KeyboardButton(text="⛅ Настроить рассылку погоды"))
    builder.add(KeyboardButton(text="❌ Отменить рассылку погоды"))

    await message.answer(
        'Привет! Я умный бот с несколькими функциями.\n'
        'Выбери действие:',
        reply_markup=builder.as_markup(resize_keyboard=True),
        parse_mode='HTML'
    )


# Обработка кнопки чата
@dp.message(F.text == "💬 Чат с ассистентом")
async def handle_chat(message: types.Message):
    user_id = message.from_user.id
    if user_id not in user_dialogs:
        user_dialogs[user_id] = [
            {"role": "system", "content": "You are a helpful assistant."}
        ]
    await message.answer("Режим чата активирован. Отправьте ваш вопрос.")


# Обработка кнопки настройки погоды
@dp.message(F.text == "⛅ Настроить рассылку погоды")
async def setup_weather_start(message: types.Message, state: FSMContext):
    await message.answer("Введите название города для рассылки погоды:")
    await state.set_state(WeatherStates.waiting_for_city)


# Обработка ввода города
@dp.message(WeatherStates.waiting_for_city)
async def process_city(message: types.Message, state: FSMContext):
    await state.update_data(city=message.text)
    await message.answer("Теперь введите время в формате ЧЧ:ММ (например, 08:00), когда вам присылать погоду:")
    await state.set_state(WeatherStates.waiting_for_time)


# Обработка ввода времени
@dp.message(WeatherStates.waiting_for_time)
async def process_time(message: types.Message, state: FSMContext):
    try:
        # Пытаемся распарсить время
        time_str = message.text
        hours, minutes = map(int, time_str.split(':'))
        if hours < 0 or hours > 23 or minutes < 0 or minutes > 59:
            raise ValueError

        user_data = await state.get_data()
        city = user_data['city']
        user_id = message.from_user.id

        # Сохраняем настройки
        weather_subscriptions[user_id] = {
            'city': city,
            'time': time(hour=hours, minute=minutes)
        }

        await message.answer(f"Рассылка погоды настроена!\nГород: {city}\nВремя: {time_str}")
        await state.clear()

    except (ValueError, IndexError):
        await message.answer(
            "Некорректный формат времени. Пожалуйста, введите время в формате ЧЧ:ММ (например, 08:00):")


# Отмена рассылки
@dp.message(F.text == "❌ Отменить рассылку погоды")
async def cancel_weather_subscription(message: types.Message):
    user_id = message.from_user.id
    if user_id in weather_subscriptions:
        del weather_subscriptions[user_id]
        await message.answer("Рассылка погоды отменена.")
    else:
        await message.answer("У вас нет активных подписок на погоду.")


# Функция для получения погоды
async def get_weather(city: str) -> str:
    try:
        # Здесь используем OpenWeatherMap API (нужен API ключ)
        API_KEY = "58eea590dd60f4c8a4930a9803ece65f"
        url = f"http://api.openweathermap.org/data/2.5/weather?q={city}&appid={API_KEY}&units=metric&lang=ru"

        response = requests.get(url)
        data = response.json()

        if data.get('cod') != 200:
            return f"Не удалось получить погоду для города {city}. Проверьте название города."

        weather = data['weather'][0]['description']
        temp = data['main']['temp']
        feels_like = data['main']['feels_like']
        humidity = data['main']['humidity']

        return (f"Погода в {city}:\n"
                f"🌡 Температура: {temp}°C (ощущается как {feels_like}°C)\n"
                f"☁ Состояние: {weather}\n"
                f"💧 Влажность: {humidity}%")

    except Exception as e:
        logging.error(f"Error getting weather: {e}")
        return f"Произошла ошибка при получении погоды для {city}"


# Проверка времени и отправка погоды
async def check_weather_schedule():
    while True:
        await asyncio.sleep(30)  # Проверяем каждые 30 секунд

        try:
            moscow_tz = pytz.timezone('Europe/Moscow')
            now = datetime.now(moscow_tz)
            current_time = now.time()

            for user_id, sub in weather_subscriptions.items():
                if 'time' in sub:
                    # Сравниваем часы и минуты
                    if current_time.hour == sub['time'].hour and current_time.minute == sub['time'].minute:
                        city = sub['city']
                        weather = await get_weather(city)
                        await bot.send_message(user_id, weather)
                        # Добавляем задержку, чтобы не отправить повторно в ту же минуту
                        await asyncio.sleep(60)

        except Exception as e:
            logging.error(f"Error in weather scheduler: {e}")
            await asyncio.sleep(60)  # Подождем перед следующей попыткой


# Обработка всех сообщений (чат с ассистентом)
@dp.message()
async def handle_message(message: Message):
    user_id = message.from_user.id

    # Если это не текст или пользователь не в режиме чата
    if not message.text or message.text in ["⛅ Настроить рассылку погоды", "❌ Отменить рассылку погоды"]:
        return

    # Если пользователь новый, инициализируем историю
    if user_id not in user_dialogs:
        user_dialogs[user_id] = [
            {"role": "system", "content": "You are a helpful assistant."}
        ]

    # Добавляем запрос пользователя в историю
    user_dialogs[user_id].append({"role": "user", "content": message.text})

    url = "https://api.intelligence.io.solutions/api/v1/chat/completions"
    headers = {
        "Content-Type": "application/json",
        "Authorization": "Bearer io-v2-eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9."
                         "eyJvd25lciI6IjlkYmNiZjJkLWJjODAtNDA3Ni04YzUwLTkwNGEwYThhM2QwMyIsImV4cCI6NDkwMDQwNjM5MH0."
                         "Jc1hn20jAAijksR3i6Fj9BXhlKompahEAoSMkwMumZMsLZacs5iU0_LBnT19StlOKXEDFbctyygz5RK_9Agmyg",
    }

    data = {
        "model": "deepseek-ai/DeepSeek-R1",
        "messages": user_dialogs[user_id],  # Отправляем ВСЮ историю сообщений
    }

    response = requests.post(url, headers=headers, json=data)
    data = response.json()

    if 'choices' in data and data['choices']:
        bot_response = data['choices'][0]['message']['content']
        # Убираем лишнее, если API возвращает <think>...</think>
        if '</think>' in bot_response:
            bot_response = bot_response.split('</think>\n\n')[1]

        # Добавляем ответ бота в историю
        user_dialogs[user_id].append({"role": "assistant", "content": bot_response})
        await message.answer(bot_response, parse_mode="Markdown")
    else:
        await message.answer("Ошибка при обработке запроса. Попробуйте позже.")


async def on_startup():
    # Запускаем фоновую задачу при старте бота
    asyncio.create_task(check_weather_schedule())


async def main():
    await bot(DeleteWebhook(drop_pending_updates=True))
    await on_startup()  # Запускаем фоновые задачи
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())