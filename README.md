import asyncio
import logging
import os
from datetime import date

from aiogram import Bot, Dispatcher, F
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.filters import CommandStart
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import Message, ReplyKeyboardMarkup, KeyboardButton, ReplyKeyboardRemove

from google import genai
from google.genai import types as genai_types

# =========================================================================
# CONFIGURATION
# =========================================================================
# Токены НИКОГДА не хранятся в коде. Задайте их как переменные окружения:
#   export BOT_TOKEN="..."
#   export GEMINI_API_KEY="..."
# либо положите их в файл .env и подключите python-dotenv.
BOT_TOKEN = os.getenv("BOT_TOKEN")
GEMINI_API_KEY = os.getenv("GEMINI_API_KEY")

if not BOT_TOKEN:
    raise RuntimeError("Переменная окружения BOT_TOKEN не задана.")
if not GEMINI_API_KEY:
    raise RuntimeError("Переменная окружения GEMINI_API_KEY не задана.")

GEMINI_TIMEOUT_SECONDS = 60  # генерация меню с БЖУ по каждому приёму пищи занимает больше времени
DAILY_CALORIE_FLOOR = 1200  # не опускаемся ниже безопасного минимума

gemini_client = genai.Client(api_key=GEMINI_API_KEY)
GEMINI_MODEL = "gemini-3.6-flash"  # Google отключила все модели 1.x и 2.5 в 2026 году

bot = Bot(
    token=BOT_TOKEN,
    default=DefaultBotProperties(parse_mode=ParseMode.MARKDOWN),
)
dp = Dispatcher(storage=MemoryStorage())

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(name)s | %(message)s",
)
logger = logging.getLogger("bytebite")

# NOTE: MemoryStorage хранит всё в оперативной памяти процесса — при
# перезапуске бота прогресс всех пользователей обнулится. Для продакшена
# замените MemoryStorage на RedisStorage (aiogram.fsm.storage.redis) или
# сохраняйте data в свою БД (например, через middleware).


# =========================================================================
# FSM STATES
# =========================================================================
class ProfileSetup(StatesGroup):
    language = State()
    phase = State()
    weight = State()
    height = State()
    activity = State()
    exclusions = State()
    preferences = State()


class EditProfile(StatesGroup):
    waiting_activity = State()
    waiting_exclusions = State()
    waiting_preferences = State()


class MealLog(StatesGroup):
    waiting_text = State()


class GoldenTicketFlow(StatesGroup):
    waiting_text = State()


# =========================================================================
# I18N TEXTS
# =========================================================================
TEXTS = {
    "ru": {
        "choose_lang": "👋 **Welcome to Byte&Bite!**\nПожалуйста, выберите язык / Please choose your language:",
        "invalid_lang": "Пожалуйста, выберите язык кнопкой ниже 👇",
        "intro": (
            "🎮 **Добро пожаловать в Byte&Bite!**\n\n"
            "⚠️ **Дисклеймер:** *Этот бот — игровой помощник для мотивации и трекинга. "
            "Он не заменяет консультацию врача, диетолога или тренера.*\n\n"
            "✨ **Система Золотого билета:**\n"
            "За каждые 3 дня честного трекинга вы получаете 🎫 **Золотой билет** "
            "для гибкого питания без чувства вины.\n\n"
            "👉 **Выберите вашу цель:**"
        ),
        "ask_weight": "Введите ваш текущий вес в кг (например: 53):",
        "invalid_weight": "Похоже, это не число. Введите вес в кг цифрами (например: 53):",
        "ask_height": "Введите ваш рост в см (например: 158):",
        "invalid_height": "Похоже, это не число. Введите рост в см цифрами (например: 158):",
        "ask_activity": "🏃 **Уровень активности:**\n\nВыберите, что больше всего похоже на ваш обычный образ жизни:",
        "ask_exclusions": (
            "🚫 **Что исключить из питания?**\n\n"
            "Напишите продукты, которые нельзя или не хочется есть "
            "(например: *без свинины, аллергия на орехи*).\n"
            "Эти продукты никогда не появятся в меню.\n"
            "Если исключений нет, отправьте: *нет*"
        ),
        "ask_preferences": (
            "❤️ **Что вы любите?**\n\n"
            "Напишите продукты или блюда, которые предпочитаете "
            "(например: *овсянка, куриная грудка, молоко*).\n"
            "Это НЕ будет исключено — наоборот, бот постарается это учитывать.\n"
            "Если предпочтений нет, отправьте: *нет*"
        ),
        "profile_ready": (
            "✅ **Профиль успешно настроен!**\n\n"
            "🎯 Цель: **{phase}**\n\n"
            "😴 Базовый обмен (тратится сам по себе): **~{bmr} ккал/день**\n"
            "🔥 Полный расход с учётом активности: **~{tdee} ккал/день**\n"
            "🍽️ Ориентир по питанию под вашу цель: **~{calories} ккал/день**\n"
            "🥩 Белки: **~{protein} г** | 🥑 Жиры: **~{fat} г** | 🍞 Углеводы: **~{carbs} г**\n"
            "🎫 Доступно золотых билетов: **1**\n\n"
            "Нажмите **'📝 Записать еду (+XP)'**, чтобы начать!"
        ),
        "need_profile": "Сначала настройте профиль командой /start!",
        "settings_menu_exclusions": (
            "⚙️ **Настройки**\n\n"
            "Обновим профиль без потери прогресса (XP, серия, билеты сохранятся).\n\n"
            "🚫 Напишите, что исключить из питания (или *нет*):"
        ),
        "settings_menu_preferences": "❤️ Теперь напишите, что вы любите (или *нет*):",
        "restrictions_updated": "✅ Профиль обновлён!",
        "stats": (
            "🏆 **Игровая панель**\n\n"
            "📊 День трекинга: {audit_day}\n"
            "⭐ Опыт: {xp} XP\n"
            "🔥 Серия дней подряд: {streak}\n"
            "🎫 Золотых билетов: {tickets} 🎟️\n\n"
            "😴 Базовый обмен: {bmr} ккал/день\n"
            "🔥 Полный расход (с активностью): {tdee} ккал/день\n"
            "🍽️ Ориентир по питанию: {calories} ккал | Б {protein}г / Ж {fat}г / У {carbs}г"
        ),
        "no_tickets": (
            "🔒 **У вас не осталось золотых билетов!**\n"
            "Продолжайте регулярно записывать приемы пищи, чтобы заработать новый билет."
        ),
        "ticket_activated": (
            "🎟️ **Золотой билет активирован!**\n"
            "Что хотите съесть без чувства вины прямо сейчас? "
            "(например: *'кусочек пиццы'*).\n\nНапишите мне 👇"
        ),
        "ask_meal_log": (
            "🍽️ **Запись еды:**\n"
            "Опишите, что вы сегодня съели и взвесили. "
            "Первая запись за день даёт **+20 XP** и продвигает серию дней!"
        ),
        "meal_logged_again_today": (
            "📝 Записал! Но XP и серия дней уже учтены сегодня — "
            "следующий бонус будет завтра. Так честнее 😉"
        ),
        "milestone_bonus": "\n\n🎉 **Бонус! Вы заработали +1 Золотой билет!** 🎫",
        "gen_menu_loading": "⏳ Генерирую меню и список покупок...",
        "gen_menu_timeout": "⌛ Gemini не ответил вовремя. Попробуйте ещё раз чуть позже.",
        "gen_menu_error": "❌ Не получилось сгенерировать меню. Попробуйте позже.",
        "ai_error": "❌ Не получилось получить ответ от ИИ. Попробуйте ещё раз чуть позже.",
    },
    "en": {
        "choose_lang": "👋 **Welcome to Byte&Bite!**\nPlease choose your language / Пожалуйста, выберите язык:",
        "invalid_lang": "Please pick a language using the button below 👇",
        "intro": (
            "🎮 **Welcome to Byte&Bite!**\n\n"
            "⚠️ **Disclaimer:** *This bot is a gamified assistant for motivation and tracking. "
            "It does not replace professional medical or nutritional advice.*\n\n"
            "✨ **The Golden Ticket System:**\n"
            "For every 3 days of tracking, you earn a 🎫 **Golden Ticket** for a guilt-free flexible meal.\n\n"
            "👉 **Choose your goal:**"
        ),
        "ask_weight": "Enter your current weight in kg (e.g., 53):",
        "invalid_weight": "That doesn't look like a number. Enter weight in kg (e.g., 53):",
        "ask_height": "Enter your height in cm (e.g., 158):",
        "invalid_height": "That doesn't look like a number. Enter height in cm (e.g., 158):",
        "ask_activity": "🏃 **Activity level:**\n\nPick what best matches your usual lifestyle:",
        "ask_exclusions": (
            "🚫 **What should we exclude?**\n\n"
            "Write foods you can't or don't want to eat (e.g., *no pork, nut allergy*).\n"
            "These will never appear in your menu.\n"
            "If none, send: *none*"
        ),
        "ask_preferences": (
            "❤️ **What do you like?**\n\n"
            "Write foods or dishes you prefer (e.g., *oatmeal, chicken breast, milk*).\n"
            "This will NOT be excluded — the bot will try to favor it instead.\n"
            "If none, send: *none*"
        ),
        "profile_ready": (
            "✅ **Profile Initialized!**\n\n"
            "🎯 Goal: **{phase}**\n\n"
            "😴 Base metabolism (burned at rest): **~{bmr} kcal/day**\n"
            "🔥 Total daily burn (with activity): **~{tdee} kcal/day**\n"
            "🍽️ Eating target for your goal: **~{calories} kcal/day**\n"
            "🥩 Protein: **~{protein} g** | 🥑 Fat: **~{fat} g** | 🍞 Carbs: **~{carbs} g**\n"
            "🎫 Golden Tickets available: **1**\n\n"
            "Tap **'📝 Log Meal (+XP)'** to start!"
        ),
        "need_profile": "Please set up your profile via /start first!",
        "settings_menu_exclusions": (
            "⚙️ **Settings**\n\n"
            "Let's update your profile without losing progress (XP, streak, tickets stay).\n\n"
            "🚫 Write what to exclude from your diet (or *none*):"
        ),
        "settings_menu_preferences": "❤️ Now write what you like (or *none*):",
        "restrictions_updated": "✅ Profile updated!",
        "stats": (
            "🏆 **Gamification Dashboard**\n\n"
            "📊 Tracking day: {audit_day}\n"
            "⭐ Experience Points: {xp} XP\n"
            "🔥 Daily Streak: {streak} days\n"
            "🎫 Golden Tickets in stock: {tickets} 🎟️\n\n"
            "😴 Base metabolism: {bmr} kcal/day\n"
            "🔥 Total burn (with activity): {tdee} kcal/day\n"
            "🍽️ Eating target: {calories} kcal | P {protein}g / F {fat}g / C {carbs}g"
        ),
        "no_tickets": (
            "🔒 **No Golden Tickets left!**\n"
            "Keep logging your meals consistently to earn a new ticket."
        ),
        "ticket_activated": (
            "🎟️ **The Golden Ticket is activated!**\n"
            "What do you want to eat right now without guilt? (e.g., *'a slice of pizza'*).\n\n"
            "Send me your craving 👇"
        ),
        "ask_meal_log": (
            "🍽️ **Log your food:**\n"
            "Send a message describing what you ate today. "
            "Your first log of the day earns **+20 XP** and advances your streak!"
        ),
        "meal_logged_again_today": (
            "📝 Logged! But XP and streak are already counted for today — "
            "next bonus is tomorrow. Keeping it fair 😉"
        ),
        "milestone_bonus": "\n\n🎉 **Milestone Bonus! You earned +1 Golden Ticket!** 🎫",
        "gen_menu_loading": "⏳ Generating your 1-day sample menu and grocery list...",
        "gen_menu_timeout": "⌛ Gemini didn't respond in time. Please try again shortly.",
        "gen_menu_error": "❌ Couldn't generate the menu. Please try again later.",
        "ai_error": "❌ Couldn't get a response from the AI. Please try again shortly.",
    },
}

PHASE_KEY_MAP = {
    "📉 Похудение": "weight_loss", "📉 Weight Loss": "weight_loss",
    "⚖️ Рекомпозиция": "recomposition", "⚖️ Body Recomposition": "recomposition",
    "🛡️ Поддержание": "maintenance", "🛡️ Maintenance": "maintenance",
    "📈 Набор массы": "muscle_gain", "📈 Muscle Gain": "muscle_gain",
}

ACTIVITY_KEY_MAP = {
    "🛋️ Малоподвижный": "sedentary", "🛋️ Sedentary": "sedentary",
    "🚶 Лёгкая (1-3 р/нед)": "light", "🚶 Light (1-3x/week)": "light",
    "🏃 Средняя (3-5 р/нед)": "moderate", "🏃 Moderate (3-5x/week)": "moderate",
    "🏋️ Высокая (6-7 р/нед)": "active", "🏋️ Active (6-7x/week)": "active",
    "🔥 Очень высокая": "very_active", "🔥 Very active": "very_active",
}

# Стандартные коэффициенты активности (Harris-Benedict / Mifflin приближение).
ACTIVITY_FACTORS = {
    "sedentary": 1.2,
    "light": 1.375,
    "moderate": 1.55,
    "active": 1.725,
    "very_active": 1.9,
}

# Множитель к базовой оценке калорий (BMR-приближение) в зависимости от цели.
# Это ГРУБАЯ прикидка, а не медицинский расчёт — бот честно предупреждает об этом.
PHASE_CALORIE_ADJUSTMENT = {
    "weight_loss": -400,
    "recomposition": -150,
    "maintenance": 0,
    "muscle_gain": 300,
}


def t(lang: str, key: str, **kwargs) -> str:
    template = TEXTS.get(lang, TEXTS["en"])[key]
    return template.format(**kwargs) if kwargs else template


def estimate_calorie_breakdown(weight_kg: float, height_cm: float, phase_key: str, activity_key: str) -> dict:
    """Очень грубая оценка (упрощённый Mifflin-St Jeor без возраста/пола, средние
    допущения). Используется только как стартовый ориентир для бота-игры,
    не как медицинская рекомендация.

    Возвращает:
    - bmr: базовый обмен веществ — калории, которые тратятся "сами по себе"
      (в состоянии полного покоя, даже если весь день лежать).
    - tdee: общий дневной расход с учётом уровня активности (BMR × коэффициент).
    - target: TDEE, скорректированный под цель (похудение/набор массы/поддержание).
    """
    bmr = 10 * weight_kg + 6.25 * height_cm - 5 * 30 + 5  # усреднённые допущения по полу/возрасту
    activity_factor = ACTIVITY_FACTORS.get(activity_key, 1.375)
    tdee = bmr * activity_factor
    target = tdee + PHASE_CALORIE_ADJUSTMENT.get(phase_key, 0)
    return {
        "bmr": round(bmr / 10) * 10,
        "tdee": round(tdee / 10) * 10,
        "target": max(DAILY_CALORIE_FLOOR, round(target / 10) * 10),
    }


def estimate_macros(weight_kg: float, target_calories: int) -> dict:
    """Грубая раскладка БЖУ: белок и жир считаются от веса тела (стандартный
    подход для фитнес-целей), углеводы добирают оставшиеся калории."""
    protein_g = round(weight_kg * 2.0)
    fat_g = round(weight_kg * 0.9)
    remaining_kcal = max(0, target_calories - protein_g * 4 - fat_g * 9)
    carbs_g = round(remaining_kcal / 4)
    return {"protein": protein_g, "fat": fat_g, "carbs": carbs_g}


def split_text_safely(text: str, limit: int = 4000):
    """Режет длинный текст по границам абзацев/строк, а не посреди слов и
    Markdown-разметки."""
    if len(text) <= limit:
        return [text]

    chunks = []
    current = ""
    for paragraph in text.split("\n"):
        candidate = f"{current}\n{paragraph}" if current else paragraph
        if len(candidate) > limit:
            if current:
                chunks.append(current)
            current = paragraph
        else:
            current = candidate
    if current:
        chunks.append(current)
    return chunks


def _generate_content_sync(prompt: str) -> str:
    response = gemini_client.models.generate_content(
        model=GEMINI_MODEL,
        contents=prompt,
    )
    return response.text


async def call_gemini(prompt: str) -> str:
    """Единая точка вызова Gemini с таймаутом. Бросает исключение наверх —
    вызывающий код решает, как сообщить об ошибке пользователю."""
    return await asyncio.wait_for(
        asyncio.to_thread(_generate_content_sync, prompt),
        timeout=GEMINI_TIMEOUT_SECONDS,
    )


# =========================================================================
# KEYBOARDS
# =========================================================================
def get_language_keyboard():
    return ReplyKeyboardMarkup(
        keyboard=[[KeyboardButton(text="🇷🇺 Русский"), KeyboardButton(text="🇬🇧 English")]],
        resize_keyboard=True,
    )


def get_phase_keyboard(lang):
    if lang == "ru":
        rows = [["📉 Похудение", "⚖️ Рекомпозиция"], ["🛡️ Поддержание", "📈 Набор массы"]]
    else:
        rows = [["📉 Weight Loss", "⚖️ Body Recomposition"], ["🛡️ Maintenance", "📈 Muscle Gain"]]
    return ReplyKeyboardMarkup(
        keyboard=[[KeyboardButton(text=b) for b in row] for row in rows],
        resize_keyboard=True,
    )


def get_activity_keyboard(lang):
    if lang == "ru":
        rows = [
            ["🛋️ Малоподвижный"],
            ["🚶 Лёгкая (1-3 р/нед)"],
            ["🏃 Средняя (3-5 р/нед)"],
            ["🏋️ Высокая (6-7 р/нед)"],
            ["🔥 Очень высокая"],
        ]
    else:
        rows = [
            ["🛋️ Sedentary"],
            ["🚶 Light (1-3x/week)"],
            ["🏃 Moderate (3-5x/week)"],
            ["🏋️ Active (6-7x/week)"],
            ["🔥 Very active"],
        ]
    return ReplyKeyboardMarkup(
        keyboard=[[KeyboardButton(text=b) for b in row] for row in rows],
        resize_keyboard=True,
    )


def get_main_keyboard(lang):
    if lang == "ru":
        rows = [
            ["📝 Записать еду (+XP)", "🏆 Статистика"],
            ["🎫 Золотой билет", "📅 Меню (1 день)"],
            ["⚙️ Настройки"],
        ]
    else:
        rows = [
            ["📝 Log Meal (+XP)", "🏆 Stats"],
            ["🎫 Golden Ticket", "📅 Menu (1 Day)"],
            ["⚙️ Settings"],
        ]
    return ReplyKeyboardMarkup(
        keyboard=[[KeyboardButton(text=b) for b in row] for row in rows],
        resize_keyboard=True,
    )


# =========================================================================
# ONBOARDING (полный сброс профиля — только через /start)
# =========================================================================
@dp.message(CommandStart())
async def start_cmd(message: Message, state: FSMContext):
    await state.clear()
    await state.set_state(ProfileSetup.language)
    await message.answer(TEXTS["ru"]["choose_lang"], reply_markup=get_language_keyboard())


@dp.message(ProfileSetup.language)
async def process_language(message: Message, state: FSMContext):
    text = message.text or ""
    if "Русский" in text:
        lang = "ru"
    elif "English" in text:
        lang = "en"
    else:
        await message.answer(TEXTS["ru"]["invalid_lang"], reply_markup=get_language_keyboard())
        return

    await state.update_data(language=lang)
    await state.set_state(ProfileSetup.phase)
    await message.answer(t(lang, "intro"), reply_markup=get_phase_keyboard(lang))


@dp.message(ProfileSetup.phase)
async def process_phase(message: Message, state: FSMContext):
    data = await state.get_data()
    lang = data.get("language", "en")

    phase_key = PHASE_KEY_MAP.get(message.text)
    if not phase_key:
        await message.answer(t(lang, "invalid_lang"), reply_markup=get_phase_keyboard(lang))
        return

    await state.update_data(
        phase=message.text,
        phase_key=phase_key,
        audit_day=0,
        streak=0,
        xp=0,
        golden_tickets=1,
        active_ticket=False,
        last_log_date=None,
    )
    await state.set_state(ProfileSetup.weight)
    await message.answer(t(lang, "ask_weight"), reply_markup=ReplyKeyboardRemove())


@dp.message(ProfileSetup.weight)
async def process_weight(message: Message, state: FSMContext):
    data = await state.get_data()
    lang = data.get("language", "en")

    try:
        weight = float((message.text or "").replace(",", "."))
        if not (20 <= weight <= 400):
            raise ValueError
    except ValueError:
        await message.answer(t(lang, "invalid_weight"))
        return

    await state.update_data(weight=weight)
    await state.set_state(ProfileSetup.height)
    await message.answer(t(lang, "ask_height"))


@dp.message(ProfileSetup.height)
async def process_height(message: Message, state: FSMContext):
    data = await state.get_data()
    lang = data.get("language", "en")

    try:
        height = float((message.text or "").replace(",", "."))
        if not (100 <= height <= 250):
            raise ValueError
    except ValueError:
        await message.answer(t(lang, "invalid_height"))
        return

    await state.update_data(height=height)
    await state.set_state(ProfileSetup.activity)
    await message.answer(t(lang, "ask_activity"), reply_markup=get_activity_keyboard(lang))


@dp.message(ProfileSetup.activity)
async def process_activity(message: Message, state: FSMContext):
    data = await state.get_data()
    lang = data.get("language", "en")

    activity_key = ACTIVITY_KEY_MAP.get(message.text)
    if not activity_key:
        await message.answer(t(lang, "invalid_lang"), reply_markup=get_activity_keyboard(lang))
        return

    await state.update_data(activity_key=activity_key)
    await state.set_state(ProfileSetup.exclusions)
    await message.answer(t(lang, "ask_exclusions"), reply_markup=ReplyKeyboardRemove())


@dp.message(ProfileSetup.exclusions)
async def process_exclusions(message: Message, state: FSMContext):
    data = await state.get_data()
    lang = data.get("language", "en")

    await state.update_data(exclusions=message.text)
    await state.set_state(ProfileSetup.preferences)
    await message.answer(t(lang, "ask_preferences"))


@dp.message(ProfileSetup.preferences)
async def process_preferences(message: Message, state: FSMContext):
    data = await state.get_data()
    lang = data.get("language", "en")

    await state.update_data(preferences=message.text)
    data = await state.get_data()

    weight = data.get("weight", 60.0)
    height = data.get("height", 165.0)
    phase_key = data.get("phase_key", "maintenance")
    activity_key = data.get("activity_key", "light")
    cal = estimate_calorie_breakdown(weight, height, phase_key, activity_key)
    macros = estimate_macros(weight, cal["target"])

    await state.update_data(
        bmr=cal["bmr"],
        tdee=cal["tdee"],
        target_calories=cal["target"],
        target_protein=macros["protein"],
        target_fat=macros["fat"],
        target_carbs=macros["carbs"],
    )
    await state.set_state(None)  # выходим из FSM онбординга — дальше обычные сообщения

    await message.answer(
        t(
            lang, "profile_ready", phase=data.get("phase"),
            bmr=cal["bmr"], tdee=cal["tdee"], calories=cal["target"], **macros,
        ),
        reply_markup=get_main_keyboard(lang),
    )


# =========================================================================
# SETTINGS (не сбрасывает прогресс — только меняет исключения/предпочтения)
# =========================================================================
@dp.message(F.text.in_(["⚙️ Настройки", "⚙️ Settings"]))
async def open_settings(message: Message, state: FSMContext):
    data = await state.get_data()
    lang = data.get("language", "en")

    if not data.get("target_calories"):
        await message.answer(t(lang, "need_profile"))
        return

    await state.set_state(EditProfile.waiting_activity)
    await message.answer(t(lang, "ask_activity"), reply_markup=get_activity_keyboard(lang))


@dp.message(EditProfile.waiting_activity)
async def save_new_activity(message: Message, state: FSMContext):
    data = await state.get_data()
    lang = data.get("language", "en")

    activity_key = ACTIVITY_KEY_MAP.get(message.text)
    if not activity_key:
        await message.answer(t(lang, "invalid_lang"), reply_markup=get_activity_keyboard(lang))
        return

    await state.update_data(activity_key=activity_key)

    # Пересчитываем калории/БЖУ сразу же с учётом новой активности.
    weight = data.get("weight", 60.0)
    height = data.get("height", 165.0)
    phase_key = data.get("phase_key", "maintenance")
    cal = estimate_calorie_breakdown(weight, height, phase_key, activity_key)
    macros = estimate_macros(weight, cal["target"])
    await state.update_data(
        bmr=cal["bmr"],
        tdee=cal["tdee"],
        target_calories=cal["target"],
        target_protein=macros["protein"],
        target_fat=macros["fat"],
        target_carbs=macros["carbs"],
    )

    await state.set_state(EditProfile.waiting_exclusions)
    await message.answer(t(lang, "settings_menu_exclusions"), reply_markup=ReplyKeyboardRemove())


@dp.message(EditProfile.waiting_exclusions)
async def save_new_exclusions(message: Message, state: FSMContext):
    data = await state.get_data()
    lang = data.get("language", "en")

    await state.update_data(exclusions=message.text)
    await state.set_state(EditProfile.waiting_preferences)
    await message.answer(t(lang, "settings_menu_preferences"))


@dp.message(EditProfile.waiting_preferences)
async def save_new_preferences(message: Message, state: FSMContext):
    data = await state.get_data()
    lang = data.get("language", "en")

    await state.update_data(preferences=message.text)
    await state.set_state(None)
    await message.answer(t(lang, "restrictions_updated"), reply_markup=get_main_keyboard(lang))


# =========================================================================
# STATS
# =========================================================================
@dp.message(F.text.in_(["🏆 Статистика", "🏆 Stats"]))
async def show_stats(message: Message, state: FSMContext):
    data = await state.get_data()
    lang = data.get("language", "en")

    if not data.get("target_calories"):
        await message.answer(t(lang, "need_profile"))
        return

    await message.answer(
        t(
            lang, "stats",
            audit_day=data.get("audit_day", 0),
            xp=data.get("xp", 0),
            streak=data.get("streak", 0),
            tickets=data.get("golden_tickets", 0),
            bmr=data.get("bmr", 0),
            tdee=data.get("tdee", 0),
            calories=data.get("target_calories", 0),
            protein=data.get("target_protein", 0),
            fat=data.get("target_fat", 0),
            carbs=data.get("target_carbs", 0),
        ),
        reply_markup=get_main_keyboard(lang),
    )


# =========================================================================
# GOLDEN TICKET
# =========================================================================
@dp.message(F.text.in_(["🎫 Золотой билет", "🎫 Golden Ticket"]))
async def use_golden_ticket(message: Message, state: FSMContext):
    data = await state.get_data()
    lang = data.get("language", "en")

    if not data.get("target_calories"):
        await message.answer(t(lang, "need_profile"))
        return

    tickets = data.get("golden_tickets", 0)
    if tickets <= 0:
        await message.answer(t(lang, "no_tickets"), reply_markup=get_main_keyboard(lang))
        return

    await state.set_state(GoldenTicketFlow.waiting_text)
    await message.answer(t(lang, "ticket_activated"), reply_markup=get_main_keyboard(lang))


@dp.message(GoldenTicketFlow.waiting_text)
async def process_golden_ticket_meal(message: Message, state: FSMContext):
    data = await state.get_data()
    lang = data.get("language", "en")
    exclusions = data.get("exclusions", "-")
    preferences = data.get("preferences", "-")

    prompt = (
        f"The user has activated their Golden Ticket to eat: \"{message.text}\".\n"
        f"User profile target: ~{data.get('target_calories', 1800)} kcal, "
        f"~{data.get('target_protein', 0)}g protein, ~{data.get('target_fat', 0)}g fat, "
        f"~{data.get('target_carbs', 0)}g carbs.\n"
        f"Strict exclusions (never suggest these, but this specific meal is already chosen by the user "
        f"so just note if it conflicts): {exclusions}.\n"
        f"User's usual food preferences (for context only): {preferences}.\n\n"
        f"INSTRUCTIONS:\n"
        f"1. Reply ENTIRELY in {'Russian' if lang == 'ru' else 'English'}.\n"
        f"2. Enthusiastically validate and approve their choice (no guilt, total support!).\n"
        f"3. Estimate the calories AND a rough protein/fat/carb breakdown in grams for this meal.\n"
        f"4. Give a smart anti-crisis plan (lighter dinner, short walk, etc.).\n"
        f"5. End with a short reassuring encouragement about staying on track this week."
    )

    try:
        reply_text = await call_gemini(prompt)
    except asyncio.TimeoutError:
        logger.warning("Gemini timeout on golden ticket flow")
        await message.answer(t(lang, "ai_error"), reply_markup=get_main_keyboard(lang))
        await state.set_state(None)
        return
    except Exception:
        logger.exception("Gemini error on golden ticket flow")
        await message.answer(t(lang, "ai_error"), reply_markup=get_main_keyboard(lang))
        await state.set_state(None)
        return

    # Билет списываем только при успешном ответе от ИИ.
    tickets = max(0, data.get("golden_tickets", 1) - 1)
    await state.update_data(golden_tickets=tickets)
    await state.set_state(None)

    for chunk in split_text_safely(reply_text):
        await message.answer(chunk)
    await message.answer(f"🎫 {tickets}", reply_markup=get_main_keyboard(lang))


# =========================================================================
# LOG MEAL (единственный способ получить XP/streak)
# =========================================================================
@dp.message(F.text.in_(["📝 Записать еду (+XP)", "📝 Log Meal (+XP)"]))
async def ask_meal_log(message: Message, state: FSMContext):
    data = await state.get_data()
    lang = data.get("language", "en")

    if not data.get("target_calories"):
        await message.answer(t(lang, "need_profile"))
        return

    await state.set_state(MealLog.waiting_text)
    await message.answer(t(lang, "ask_meal_log"), reply_markup=get_main_keyboard(lang))


@dp.message(MealLog.waiting_text)
async def process_meal_log(message: Message, state: FSMContext):
    data = await state.get_data()
    lang = data.get("language", "en")

    today = date.today().isoformat()
    last_log_date = data.get("last_log_date")
    already_logged_today = last_log_date == today

    xp = data.get("xp", 0)
    streak = data.get("streak", 0)
    audit_day = data.get("audit_day", 0)
    tickets = data.get("golden_tickets", 0)
    milestone_msg = ""

    if not already_logged_today:
        xp += 20
        streak += 1
        audit_day += 1
        if audit_day % 3 == 0:
            tickets += 1
            milestone_msg = t(lang, "milestone_bonus")
        await state.update_data(
            xp=xp, streak=streak, audit_day=audit_day,
            golden_tickets=tickets, last_log_date=today,
        )

    prompt = (
        f"You are an encouraging, gamified personal nutritionist and fitness coach.\n"
        f"User status: Day {audit_day}. XP: {xp}, Streak: {streak} days, Golden Tickets: {tickets}.\n"
        f"User's exclusions (allergies/dislikes, never recommend these): {data.get('exclusions', '-')}\n"
        f"User's preferences (foods they like, NOT to be excluded): {data.get('preferences', '-')}\n\n"
        f"User's meal log: \"{message.text}\"\n\n"
        f"INSTRUCTIONS:\n"
        f"1. Reply ENTIRELY in {'Russian' if lang == 'ru' else 'English'}.\n"
        f"2. Acknowledge and praise their input like a fun video-game coach.\n"
        f"3. Estimate the approximate calories and protein/fat/carbs (in grams) for what they logged. "
        f"If the description is too vague to estimate, say so briefly and ask for grams next time.\n"
        f"4. Give one short practical tip, respecting their exclusions."
    )

    try:
        ai_text = await call_gemini(prompt)
    except asyncio.TimeoutError:
        logger.warning("Gemini timeout on meal log")
        ai_text = None
    except Exception:
        logger.exception("Gemini error on meal log")
        ai_text = None

    await state.set_state(None)

    if already_logged_today:
        prefix = t(lang, "meal_logged_again_today") + "\n\n"
    else:
        prefix = (
            f"✨ **+20 XP!** (Серия: {streak} дн.) | 🎫 Билетов: {tickets}\n\n"
            if lang == "ru"
            else f"✨ **+20 XP!** (Streak: {streak} days) | 🎫 Tickets: {tickets}\n\n"
        )

    if ai_text is None:
        await message.answer(prefix + t(lang, "ai_error") + milestone_msg, reply_markup=get_main_keyboard(lang))
        return

    full_text = prefix + ai_text + milestone_msg
    for chunk in split_text_safely(full_text):
        await message.answer(chunk, reply_markup=get_main_keyboard(lang))


# =========================================================================
# MENU GENERATION (1 DAY)
# =========================================================================
@dp.message(F.text.in_(["📅 Меню (1 день)", "📅 Menu (1 Day)"]))
async def generate_single_day_menu(message: Message, state: FSMContext):
    data = await state.get_data()
    lang = data.get("language", "en")

    if not data.get("target_calories"):
        await message.answer(t(lang, "need_profile"))
        return

    msg = await message.answer(t(lang, "gen_menu_loading"))

    target_calories = data.get("target_calories", 1800)
    target_protein = data.get("target_protein", 0)
    target_fat = data.get("target_fat", 0)
    target_carbs = data.get("target_carbs", 0)

    # Заранее считаем бюджет калорий по приёмам пищи — так модели проще
    # попасть в цель, чем при одной общей цифре на весь день.
    meal_split = {
        "breakfast": round(target_calories * 0.25),
        "lunch": round(target_calories * 0.35),
        "dinner": round(target_calories * 0.30),
        "snack": round(target_calories * 0.10),
    }

    prompt = (
        f"Create a detailed 1-day sample meal plan split into 4 meals: "
        f"Breakfast, Lunch, Dinner, Snack.\n\n"
        f"STRICT calorie budget per meal (stay within ±7% of each number):\n"
        f"- Breakfast: ~{meal_split['breakfast']} kcal\n"
        f"- Lunch: ~{meal_split['lunch']} kcal\n"
        f"- Dinner: ~{meal_split['dinner']} kcal\n"
        f"- Snack: ~{meal_split['snack']} kcal\n"
        f"Daily total target: ~{target_calories} kcal (must land within ±7% of this — "
        f"recompute portion sizes if your draft total drifts outside that range), "
        f"~{target_protein}g protein, ~{target_fat}g fat, ~{target_carbs}g carbs for the day.\n\n"
        f"HARD LIMIT: total fruit across the entire day must not exceed 250g combined "
        f"(sum of all fruit servings in all meals/snacks). Track this as you build the plan.\n\n"
        f"STRICTLY exclude these foods — never include them under any circumstance: "
        f"{data.get('exclusions', '-')}.\n"
        f"User preferences — favor these foods when it fits the plan, do NOT treat them as "
        f"exclusions: {data.get('preferences', '-')}.\n\n"
        f"INSTRUCTIONS:\n"
        f"1. Write the menu clearly with precise gram measurements for each meal.\n"
        f"2. For each meal, include an estimated calorie and protein/fat/carb (grams) breakdown, "
        f"and make sure it matches that meal's budget above.\n"
        f"3. Before finalizing, add up all 4 meals' calories yourself and check the sum is within "
        f"±7% of {target_calories} kcal. Also add up all fruit grams across the day and confirm "
        f"it does not exceed 250g total. If either check fails, adjust and recalculate.\n"
        f"4. At the end, show the daily total calories and protein/fat/carbs, and a clear "
        f"'Grocery List' with exact weights.\n"
        f"Language requirement: Write the ENTIRE response in {'Russian' if lang == 'ru' else 'English'}."
    )

    try:
        text = await call_gemini(prompt)
    except asyncio.TimeoutError:
        logger.warning("Gemini timeout on menu generation")
        await bot.edit_message_text(
            t(lang, "gen_menu_timeout"), chat_id=message.chat.id, message_id=msg.message_id,
        )
        return
    except Exception:
        logger.exception("Gemini error on menu generation")
        await bot.edit_message_text(
            t(lang, "gen_menu_error"), chat_id=message.chat.id, message_id=msg.message_id,
        )
        return

    # Удаляем "loading"-сообщение только после того, как ответ успешно получен.
    try:
        await bot.delete_message(chat_id=message.chat.id, message_id=msg.message_id)
    except Exception:
        pass  # не критично, если не удалось удалить

    for chunk in split_text_safely(text):
        await message.answer(chunk, reply_markup=get_main_keyboard(lang))


# =========================================================================
# FALLBACK
# =========================================================================
@dp.message()
async def fallback(message: Message, state: FSMContext):
    data = await state.get_data()
    lang = data.get("language", "en")

    if not data.get("target_calories"):
        await message.answer(
            "Нажмите /start для настройки! / Press /start to set up your profile!"
        )
        return

    await message.answer(t(lang, "need_profile") if False else (
        "Используйте меню кнопок ниже 👇" if lang == "ru" else "Please use the menu buttons below 👇"
    ), reply_markup=get_main_keyboard(lang))


# =========================================================================
# ENTRYPOINT
# =========================================================================
async def main():
    await bot.delete_webhook(drop_pending_updates=True)
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
