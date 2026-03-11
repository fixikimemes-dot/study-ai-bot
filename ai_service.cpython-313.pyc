from __future__ import annotations

import csv
import logging
import re
import zipfile
from datetime import datetime, time
from pathlib import Path
from typing import Any
from zoneinfo import ZoneInfo

from dotenv import load_dotenv
from telegram import BotCommand, InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.constants import ParseMode
from telegram.ext import (
    Application,
    ApplicationBuilder,
    CallbackQueryHandler,
    CommandHandler,
    ContextTypes,
    ConversationHandler,
    Defaults,
    MessageHandler,
    filters,
)

from .ai_service import AIService
from .config import ConfigError, Settings, load_settings
from .db import Database
from .formatters import format_hobbies, format_plan_table, progress_summary
from .keyboards import (
    BTN_ADD,
    BTN_AI,
    BTN_CANCEL,
    BTN_COACH,
    BTN_DONE,
    BTN_EXPORT,
    BTN_PLAN,
    BTN_SETTINGS,
    PRIORITY_HIGH,
    PRIORITY_LOW,
    PRIORITY_MED,
    SCHEDULE_EVENING,
    SCHEDULE_MIDDAY,
    SCHEDULE_MORNING,
    SCHEDULE_TZ,
    cancel_menu,
    main_menu,
    priority_menu,
    quick_plan_keyboard,
    settings_keyboard,
    task_action_keyboard,
    timezone_keyboard,
)

logging.basicConfig(
    format="%(asctime)s | %(name)s | %(levelname)s | %(message)s",
    level=logging.INFO,
)
logger = logging.getLogger(__name__)

ONBOARD_BIO, ONBOARD_HOBBIES, ONBOARD_TIMEZONE, ONBOARD_MORNING, ONBOARD_MIDDAY, ONBOARD_EVENING = range(6)
ADD_TITLE, ADD_PRIORITY, ADD_DURATION, ADD_NOTE = range(10, 14)
COACH_QUESTION = 20
CHECKIN_SUMMARY = 30

TIME_RE = re.compile(r"^(?:[01]?\d|2[0-3]):[0-5]\d$")


def services(context: ContextTypes.DEFAULT_TYPE) -> tuple[Database, AIService, Settings]:
    return (
        context.application.bot_data["db"],
        context.application.bot_data["ai"],
        context.application.bot_data["settings"],
    )


def parse_time_text(raw: str) -> str | None:
    value = raw.strip()
    if TIME_RE.fullmatch(value):
        hour, minute = value.split(":")
        return f"{int(hour):02d}:{minute}"
    return None


def parse_duration(raw: str) -> int | None:
    value = raw.strip().lower()
    if value in {"-", "нет", "skip", "пропуск"}:
        return None
    if value.isdigit() and 1 <= int(value) <= 600:
        return int(value)
    return None


def parse_priority(raw: str) -> int | None:
    mapping = {
        PRIORITY_LOW: 1,
        PRIORITY_MED: 2,
        PRIORITY_HIGH: 3,
        "1": 1,
        "2": 2,
        "3": 3,
    }
    return mapping.get(raw.strip())


def ensure_timezone(zone_name: str) -> str | None:
    try:
        ZoneInfo(zone_name)
        return zone_name
    except Exception:
        return None


def today_str(user: dict[str, Any]) -> str:
    tz = ZoneInfo(user["timezone"])
    return datetime.now(tz).date().isoformat()


def now_hhmm(user: dict[str, Any]) -> str:
    tz = ZoneInfo(user["timezone"])
    return datetime.now(tz).strftime("%H:%M")


def reminder_job_name(kind: str, telegram_id: int) -> str:
    return f"reminder:{kind}:{telegram_id}"


def _remove_jobs(application: Application, name: str) -> None:
    job_queue = application.job_queue
    if not job_queue:
        return
    for job in job_queue.get_jobs_by_name(name):
        job.schedule_removal()


async def schedule_jobs_for_user(application: Application, user: dict[str, Any]) -> None:
    if not user.get("onboarding_complete"):
        return
    job_queue = application.job_queue
    if not job_queue:
        return

    tz = ZoneInfo(user["timezone"])
    telegram_id = int(user["telegram_id"])

    jobs = {
        "morning": (user["morning_time"], morning_reminder),
        "midday": (user["midday_time"], midday_reminder),
        "evening": (user["evening_time"], evening_reminder),
    }

    for kind, (hhmm, callback) in jobs.items():
        _remove_jobs(application, reminder_job_name(kind, telegram_id))
        hour, minute = map(int, hhmm.split(":"))
        job_queue.run_daily(
            callback,
            time=time(hour=hour, minute=minute, tzinfo=tz),
            data={"telegram_id": telegram_id, "kind": kind},
            name=reminder_job_name(kind, telegram_id),
        )


async def post_init(application: Application) -> None:
    db: Database = application.bot_data["db"]
    await db.init()
    await application.bot.set_my_commands(
        [
            BotCommand("start", "запустить или перезапустить бота"),
            BotCommand("menu", "показать меню"),
            BotCommand("plan", "план на сегодня"),
            BotCommand("analysis", "AI-анализ дня"),
            BotCommand("coach", "задать вопрос AI-коучу"),
            BotCommand("report", "написать промежуточный отчёт"),
            BotCommand("export", "экспорт данных за 30 дней"),
            BotCommand("settings", "настройки напоминаний"),
        ]
    )

    for user in await db.list_users():
        await schedule_jobs_for_user(application, user)


async def send_main_menu_message(update: Update, text: str) -> None:
    if update.message:
        await update.message.reply_text(text, reply_markup=main_menu(), parse_mode=ParseMode.HTML)
    elif update.callback_query and update.callback_query.message:
        await update.callback_query.message.reply_text(text, reply_markup=main_menu(), parse_mode=ParseMode.HTML)


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    db, _, settings = services(context)
    user_obj = update.effective_user
    if not user_obj or not update.message:
        return ConversationHandler.END

    user = await db.ensure_user(
        telegram_id=user_obj.id,
        full_name=user_obj.full_name,
        username=user_obj.username,
        timezone=settings.default_timezone,
        morning_time=settings.default_morning_time,
        midday_time=settings.default_midday_time,
        evening_time=settings.default_evening_time,
    )

    context.user_data.pop("settings_target", None)

    if user.get("onboarding_complete"):
        await update.message.reply_text(
            "Я уже на месте. Напомню про план, помогу отмечать прогресс и дам AI-разбор дня.",
            reply_markup=main_menu(),
        )
        return ConversationHandler.END

    await update.message.reply_text(
        "Привет. Я соберу тебе умного Telegram-бота для контроля успеваемости за день.\n\n"
        "Сначала коротко расскажи о себе: чем занимаешься, в каком ты ритме, что сейчас важно. "
        "Это будет твоя биография для AI-анализа.",
        reply_markup=cancel_menu(),
    )
    return ONBOARD_BIO


async def onboarding_bio(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if not update.message:
        return ONBOARD_BIO
    context.user_data["bio"] = update.message.text.strip()
    await update.message.reply_text(
        "Теперь напиши свои увлечения через запятую.\n"
        "Например: спорт, программирование, игры, музыка.",
        reply_markup=cancel_menu(),
    )
    return ONBOARD_HOBBIES


async def onboarding_hobbies(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if not update.message:
        return ONBOARD_HOBBIES
    hobbies = [part.strip() for part in update.message.text.split(",") if part.strip()]
    context.user_data["hobbies"] = hobbies
    await update.message.reply_text(
        "Выбери часовой пояс или отправь свой вручную строкой вроде <code>Europe/Amsterdam</code>.",
        reply_markup=timezone_keyboard(),
        parse_mode=ParseMode.HTML,
    )
    return ONBOARD_TIMEZONE


async def onboarding_timezone_callback(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query
    if not query:
        return ONBOARD_TIMEZONE
    await query.answer()
    timezone_name = query.data.split(":", 1)[1]
    context.user_data["timezone"] = timezone_name
    await query.message.reply_text(
        "Во сколько писать тебе утром? Формат <code>HH:MM</code>, например <code>08:00</code>.",
        reply_markup=cancel_menu(),
        parse_mode=ParseMode.HTML,
    )
    return ONBOARD_MORNING


async def onboarding_timezone_text(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if not update.message:
        return ONBOARD_TIMEZONE
    timezone_name = ensure_timezone(update.message.text.strip())
    if not timezone_name:
        await update.message.reply_text(
            "Не смог распознать этот часовой пояс. Попробуй что-то вроде <code>Europe/Amsterdam</code> или нажми кнопку.",
            parse_mode=ParseMode.HTML,
        )
        return ONBOARD_TIMEZONE
    context.user_data["timezone"] = timezone_name
    await update.message.reply_text(
        "Во сколько писать тебе утром? Формат <code>HH:MM</code>, например <code>08:00</code>.",
        reply_markup=cancel_menu(),
        parse_mode=ParseMode.HTML,
    )
    return ONBOARD_MORNING


async def onboarding_morning(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if not update.message:
        return ONBOARD_MORNING
    value = parse_time_text(update.message.text)
    if not value:
        await update.message.reply_text("Нужно время в формате HH:MM. Например: 08:00")
        return ONBOARD_MORNING
    context.user_data["morning_time"] = value
    await update.message.reply_text("Во сколько писать днём для проверки прогресса? Формат HH:MM")
    return ONBOARD_MIDDAY


async def onboarding_midday(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if not update.message:
        return ONBOARD_MIDDAY
    value = parse_time_text(update.message.text)
    if not value:
        await update.message.reply_text("Нужно время в формате HH:MM. Например: 14:00")
        return ONBOARD_MIDDAY
    context.user_data["midday_time"] = value
    await update.message.reply_text("Во сколько писать вечером для подведения итогов? Формат HH:MM")
    return ONBOARD_EVENING


async def onboarding_evening(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if not update.message or not update.effective_user:
        return ONBOARD_EVENING
    value = parse_time_text(update.message.text)
    if not value:
        await update.message.reply_text("Нужно время в формате HH:MM. Например: 20:30")
        return ONBOARD_EVENING

    db, _, _ = services(context)
    await db.update_profile(
        telegram_id=update.effective_user.id,
        bio=context.user_data.get("bio", ""),
        hobbies=context.user_data.get("hobbies", []),
        timezone=context.user_data.get("timezone", "Europe/Amsterdam"),
        morning_time=context.user_data.get("morning_time", "08:00"),
        midday_time=context.user_data.get("midday_time", "14:00"),
        evening_time=value,
    )
    user = await db.get_user(update.effective_user.id)
    if user:
        await schedule_jobs_for_user(context.application, user)

    context.user_data.pop("bio", None)
    context.user_data.pop("hobbies", None)
    context.user_data.pop("timezone", None)
    context.user_data.pop("morning_time", None)
    context.user_data.pop("midday_time", None)

    await update.message.reply_text(
        "Готово. Теперь я:\n"
        "• сам пишу тебе утром, днём и вечером;\n"
        "• собираю задачи кнопками;\n"
        "• показываю план в виде таблицы;\n"
        "• делаю AI-анализ по биографии, интересам и твоим результатам.\n\n"
        "Начнём с первой задачи?",
        reply_markup=main_menu(),
    )
    return ConversationHandler.END


async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data.pop("settings_target", None)
    if update.message:
        await update.message.reply_text("Окей, остановились. Меню ниже.", reply_markup=main_menu())
    elif update.callback_query and update.callback_query.message:
        await update.callback_query.answer()
        await update.callback_query.message.reply_text("Окей, остановились. Меню ниже.", reply_markup=main_menu())
    return ConversationHandler.END


async def add_task_entry(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if update.callback_query:
        await update.callback_query.answer()
        await update.callback_query.message.reply_text(
            "Напиши название задачи на сегодня.",
            reply_markup=cancel_menu(),
        )
    elif update.message:
        await update.message.reply_text("Напиши название задачи на сегодня.", reply_markup=cancel_menu())
    return ADD_TITLE


async def add_task_title(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if not update.message:
        return ADD_TITLE
    context.user_data["task_title"] = update.message.text.strip()
    await update.message.reply_text(
        "Выбери приоритет.",
        reply_markup=priority_menu(),
    )
    return ADD_PRIORITY


async def add_task_priority(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if not update.message:
        return ADD_PRIORITY
    priority = parse_priority(update.message.text)
    if priority is None:
        await update.message.reply_text("Нажми одну из кнопок приоритета или пришли 1, 2 или 3.")
        return ADD_PRIORITY
    context.user_data["task_priority"] = priority
    await update.message.reply_text(
        "Сколько минут займёт задача? Можно написать число, например 45.\n"
        "Если не хочешь указывать — отправь <code>-</code>.",
        reply_markup=cancel_menu(),
        parse_mode=ParseMode.HTML,
    )
    return ADD_DURATION


async def add_task_duration(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if not update.message:
        return ADD_DURATION
    duration = parse_duration(update.message.text)
    if duration is None and update.message.text.strip() not in {"-", "нет", "skip", "пропуск"}:
        await update.message.reply_text("Напиши число минут или символ <code>-</code>.", parse_mode=ParseMode.HTML)
        return ADD_DURATION
    context.user_data["task_duration"] = duration
    await update.message.reply_text(
        "Добавить комментарий к задаче? Например: «сделать до 18:00» или «после работы».\n"
        "Если без комментария — отправь <code>-</code>.",
        reply_markup=cancel_menu(),
        parse_mode=ParseMode.HTML,
    )
    return ADD_NOTE


async def add_task_note(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if not update.message or not update.effective_user:
        return ADD_NOTE

    note = update.message.text.strip()
    if note == "-":
        note = ""

    db, _, _ = services(context)
    user = await db.get_user(update.effective_user.id)
    if not user:
        await update.message.reply_text("Сначала нажми /start.")
        return ConversationHandler.END

    await db.add_task(
        telegram_id=update.effective_user.id,
        day=today_str(user),
        title=context.user_data.get("task_title", "Без названия"),
        priority=int(context.user_data.get("task_priority", 2)),
        duration_minutes=context.user_data.get("task_duration"),
        note=note,
    )
    context.user_data.pop("task_title", None)
    context.user_data.pop("task_priority", None)
    context.user_data.pop("task_duration", None)

    tasks = await db.get_tasks_for_day(update.effective_user.id, today_str(user))
    await update.message.reply_text(
        "Задача добавлена. Вот твоя таблица на сегодня:",
        reply_markup=main_menu(),
    )
    await update.message.reply_text(
        format_plan_table(tasks),
        parse_mode=ParseMode.HTML,
        reply_markup=task_action_keyboard(tasks),
    )
    return ConversationHandler.END


async def show_plan(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    db, _, _ = services(context)
    user_obj = update.effective_user
    if not user_obj:
        return
    user = await db.get_user(user_obj.id)
    if not user:
        if update.message:
            await update.message.reply_text("Сначала нажми /start.")
        return

    tasks = await db.get_tasks_for_day(user_obj.id, today_str(user))
    text = format_plan_table(tasks) + "\n\n" + progress_summary(tasks)

    if update.callback_query and update.callback_query.message:
        await update.callback_query.answer()
        await update.callback_query.message.reply_text(
            text,
            parse_mode=ParseMode.HTML,
            reply_markup=task_action_keyboard(tasks) if tasks else None,
        )
    elif update.message:
        await update.message.reply_text(
            text,
            parse_mode=ParseMode.HTML,
            reply_markup=task_action_keyboard(tasks) if tasks else None,
        )


async def mark_done_prompt(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    db, _, _ = services(context)
    user_obj = update.effective_user
    if not user_obj or not update.message:
        return
    user = await db.get_user(user_obj.id)
    if not user:
        await update.message.reply_text("Сначала нажми /start.")
        return
    tasks = await db.get_tasks_for_day(user_obj.id, today_str(user))
    if not tasks:
        await update.message.reply_text("У тебя пока нет задач на сегодня.", reply_markup=main_menu())
        return
    await update.message.reply_text(
        "Нажми кнопку под нужной задачей: ✅ — сделал, ⏭️ — перенёс, 🗑️ — удалить.",
        reply_markup=main_menu(),
    )
    await update.message.reply_text(
        format_plan_table(tasks),
        parse_mode=ParseMode.HTML,
        reply_markup=task_action_keyboard(tasks),
    )


async def task_callback(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    query = update.callback_query
    if not query or not update.effective_user:
        return
    await query.answer()
    db, _, _ = services(context)

    action, task_id_raw = query.data.split(":", 1)
    task = await db.get_task(int(task_id_raw))
    if not task:
        await query.message.reply_text("Эта задача уже не найдена.")
        return

    if action == "done":
        await db.update_task_status(int(task_id_raw), "done")
        status_text = "Задача отмечена как выполненная ✅"
    elif action == "skip":
        await db.update_task_status(int(task_id_raw), "skipped")
        status_text = "Задача перенесена ⏭️"
    elif action == "delete":
        await db.delete_task(int(task_id_raw))
        status_text = "Задача удалена 🗑️"
    else:
        status_text = "Неизвестное действие"

    user = await db.get_user(update.effective_user.id)
    if not user:
        await query.message.reply_text(status_text)
        return
    tasks = await db.get_tasks_for_day(update.effective_user.id, today_str(user))
    await query.message.reply_text(status_text)
    await query.message.reply_text(
        format_plan_table(tasks) + "\n\n" + progress_summary(tasks),
        parse_mode=ParseMode.HTML,
        reply_markup=task_action_keyboard(tasks) if tasks else None,
    )


async def analysis_today(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    db, ai, _ = services(context)
    user_obj = update.effective_user
    if not user_obj:
        return
    user = await db.get_user(user_obj.id)
    if not user:
        if update.message:
            await update.message.reply_text("Сначала нажми /start.")
        elif update.callback_query and update.callback_query.message:
            await update.callback_query.answer()
            await update.callback_query.message.reply_text("Сначала нажми /start.")
        return

    day = today_str(user)
    tasks = await db.get_tasks_for_day(user_obj.id, day)
    checkins = await db.get_checkins_for_day(user_obj.id, day)
    history = await db.get_recent_history(user_obj.id, days=7)

    if update.callback_query and update.callback_query.message:
        await update.callback_query.answer("Думаю...")
        message = await update.callback_query.message.reply_text("Смотрю на твой день, привычки и прогресс...")
    elif update.message:
        message = await update.message.reply_text("Смотрю на твой день, привычки и прогресс...")
    else:
        return

    analysis = await ai.daily_analysis(user, tasks, checkins, history)
    await db.save_ai_analysis(user_obj.id, day, analysis)
    await message.reply_text(
        f"<b>AI-анализ</b>\n{analysis}",
        parse_mode=ParseMode.HTML,
        reply_markup=main_menu(),
    )


async def coach_entry(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if update.message:
        if context.args:
            context.user_data["coach_direct_question"] = " ".join(context.args)
            return await coach_question(update, context)
        await update.message.reply_text(
            "Напиши вопрос AI-коучу. Например: «Как не сорваться с плана после работы?»",
            reply_markup=cancel_menu(),
        )
        return COACH_QUESTION
    return ConversationHandler.END


async def coach_question(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    db, ai, _ = services(context)
    user_obj = update.effective_user
    if not user_obj or not update.message:
        return ConversationHandler.END
    user = await db.get_user(user_obj.id)
    if not user:
        await update.message.reply_text("Сначала нажми /start.")
        return ConversationHandler.END

    question = context.user_data.pop("coach_direct_question", None) or update.message.text.strip()
    tasks = await db.get_tasks_for_day(user_obj.id, today_str(user))
    history = await db.get_recent_history(user_obj.id, days=7)

    wait_message = await update.message.reply_text("Формулирую ответ...", reply_markup=main_menu())
    reply = await ai.coach_reply(user, tasks, history, question)
    await wait_message.reply_text(f"<b>AI-коуч</b>\n{reply}", parse_mode=ParseMode.HTML)
    return ConversationHandler.END


async def checkin_entry(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    kind = "manual"
    if update.callback_query:
        await update.callback_query.answer()
        kind = update.callback_query.data.split(":", 1)[1]
        context.user_data["checkin_kind"] = kind
        await update.callback_query.message.reply_text(
            "Напиши коротко, как проходит день: что сделал, где застрял, что мешает.",
            reply_markup=cancel_menu(),
        )
        return CHECKIN_SUMMARY
    if update.message:
        await update.message.reply_text(
            "Напиши коротко, как проходит день: что сделал, где застрял, что мешает.",
            reply_markup=cancel_menu(),
        )
        context.user_data["checkin_kind"] = kind
        return CHECKIN_SUMMARY
    return ConversationHandler.END


async def checkin_summary(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    db, _, _ = services(context)
    user_obj = update.effective_user
    if not user_obj or not update.message:
        return ConversationHandler.END
    user = await db.get_user(user_obj.id)
    if not user:
        await update.message.reply_text("Сначала нажми /start.")
        return ConversationHandler.END

    await db.save_checkin(
        telegram_id=user_obj.id,
        day=today_str(user),
        kind=context.user_data.pop("checkin_kind", "manual"),
        summary=update.message.text.strip(),
    )
    await update.message.reply_text(
        "Принято. Я запомнил твой отчёт и учту его в следующем AI-анализе.",
        reply_markup=main_menu(),
    )
    return ConversationHandler.END


async def settings_entry(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if update.message:
        await update.message.reply_text(
            "Что изменить?",
            reply_markup=settings_keyboard(),
        )


async def settings_callback(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    query = update.callback_query
    if not query or not update.effective_user:
        return
    await query.answer()
    data = query.data
    context.user_data.pop("settings_target", None)

    if data == SCHEDULE_MORNING:
        context.user_data["settings_target"] = "morning_time"
        await query.message.reply_text("Напиши новое утреннее время в формате HH:MM", reply_markup=cancel_menu())
        return
    if data == SCHEDULE_MIDDAY:
        context.user_data["settings_target"] = "midday_time"
        await query.message.reply_text("Напиши новое дневное время в формате HH:MM", reply_markup=cancel_menu())
        return
    if data == SCHEDULE_EVENING:
        context.user_data["settings_target"] = "evening_time"
        await query.message.reply_text("Напиши новое вечернее время в формате HH:MM", reply_markup=cancel_menu())
        return
    if data == SCHEDULE_TZ:
        await query.message.reply_text(
            "Выбери часовой пояс или отправь его вручную строкой вроде Europe/Amsterdam",
            reply_markup=timezone_keyboard(),
        )
        context.user_data["settings_target"] = "timezone"
        return
    if data.startswith("tz:"):
        timezone_name = data.split(":", 1)[1]
        db, _, _ = services(context)
        await db.update_schedule(update.effective_user.id, timezone=timezone_name)
        user = await db.get_user(update.effective_user.id)
        if user:
            await schedule_jobs_for_user(context.application, user)
        context.user_data.pop("settings_target", None)
        await query.message.reply_text(
            f"Часовой пояс обновлён: <b>{timezone_name}</b>",
            parse_mode=ParseMode.HTML,
            reply_markup=main_menu(),
        )


async def maybe_handle_settings_value(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if not update.message or not update.effective_user:
        return
    target = context.user_data.get("settings_target")
    if not target:
        return

    db, _, _ = services(context)
    text = update.message.text.strip()

    if text == BTN_CANCEL:
        context.user_data.pop("settings_target", None)
        await update.message.reply_text("Настройка отменена.", reply_markup=main_menu())
        return

    if target == "timezone":
        timezone_name = ensure_timezone(text)
        if not timezone_name:
            await update.message.reply_text("Не смог распознать часовой пояс. Попробуй ещё раз.")
            return
        await db.update_schedule(update.effective_user.id, timezone=timezone_name)
    else:
        hhmm = parse_time_text(text)
        if not hhmm:
            await update.message.reply_text("Нужно время в формате HH:MM.")
            return
        await db.update_schedule(update.effective_user.id, **{target: hhmm})

    user = await db.get_user(update.effective_user.id)
    if user:
        await schedule_jobs_for_user(context.application, user)

    context.user_data.pop("settings_target", None)
    await update.message.reply_text("Настройки обновлены.", reply_markup=main_menu())


async def export_data(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    db, _, settings = services(context)
    user_obj = update.effective_user
    if not user_obj or not update.message:
        return
    user = await db.get_user(user_obj.id)
    if not user:
        await update.message.reply_text("Сначала нажми /start.")
        return

    dataset = await db.export_dataset(user_obj.id, days=30)
    export_path = create_export_zip(settings.export_dir, user_obj.id, dataset)

    caption = (
        "Экспорт за последние 30 дней.\n"
        f"Профиль: {user.get('full_name') or 'без имени'}\n"
        f"Увлечения: {format_hobbies(user.get('hobbies', []))}"
    )
    with export_path.open("rb") as file_obj:
        await update.message.reply_document(document=file_obj, filename=export_path.name, caption=caption)
    export_path.unlink(missing_ok=True)


def create_export_zip(export_dir: Path, telegram_id: int, dataset: dict[str, list[dict[str, Any]]]) -> Path:
    timestamp = datetime.utcnow().strftime("%Y%m%d_%H%M%S")
    zip_path = export_dir / f"study_tracker_{telegram_id}_{timestamp}.zip"

    with zipfile.ZipFile(zip_path, mode="w", compression=zipfile.ZIP_DEFLATED) as archive:
        for name, rows in dataset.items():
            csv_path = export_dir / f"{name}_{telegram_id}_{timestamp}.csv"
            write_csv(csv_path, rows)
            archive.write(csv_path, arcname=csv_path.name)
            csv_path.unlink(missing_ok=True)
    return zip_path


def write_csv(path: Path, rows: list[dict[str, Any]]) -> None:
    fieldnames: list[str] = []
    if rows:
        fieldnames = list(rows[0].keys())
    else:
        fieldnames = ["empty"]
        rows = [{"empty": ""}]

    with path.open("w", encoding="utf-8", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(rows)


async def menu(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if update.message:
        await update.message.reply_text("Главное меню ниже.", reply_markup=main_menu())


async def reminder_action_router(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    query = update.callback_query
    if not query:
        return
    data = query.data
    if data == "open:plan":
        await show_plan(update, context)
    elif data == "open:analysis":
        await analysis_today(update, context)


async def morning_reminder(context: ContextTypes.DEFAULT_TYPE) -> None:
    db, _, _ = services(context)
    telegram_id = int(context.job.data["telegram_id"])
    user = await db.get_user(telegram_id)
    if not user:
        return
    tasks = await db.get_tasks_for_day(telegram_id, today_str(user))
    text = (
        f"Доброе утро. Сейчас у тебя <b>{now_hhmm(user)}</b>.\n"
        "Готов составить или обновить план на день."
    )
    if tasks:
        text += "\n\n" + progress_summary(tasks)
    await context.bot.send_message(
        chat_id=telegram_id,
        text=text,
        parse_mode=ParseMode.HTML,
        reply_markup=InlineKeyboardMarkup(
            [
                [InlineKeyboardButton("➕ Добавить задачу", callback_data="open:add")],
                [InlineKeyboardButton("📋 Открыть план", callback_data="open:plan")],
                [InlineKeyboardButton("🧠 AI-анализ", callback_data="open:analysis")],
            ]
        ),
    )


async def midday_reminder(context: ContextTypes.DEFAULT_TYPE) -> None:
    db, _, _ = services(context)
    telegram_id = int(context.job.data["telegram_id"])
    user = await db.get_user(telegram_id)
    if not user:
        return
    tasks = await db.get_tasks_for_day(telegram_id, today_str(user))
    text = (
        f"Дневная проверка. Сейчас <b>{now_hhmm(user)}</b>.\n"
        "Посмотри, что уже закрыто, и напиши короткий отчёт.\n\n"
        + progress_summary(tasks)
    )
    await context.bot.send_message(
        chat_id=telegram_id,
        text=text,
        parse_mode=ParseMode.HTML,
        reply_markup=InlineKeyboardMarkup(
            [
                [InlineKeyboardButton("📋 План", callback_data="open:plan")],
                [InlineKeyboardButton("✍️ Отчёт", callback_data="checkin:midday")],
                [InlineKeyboardButton("🧠 AI-анализ", callback_data="open:analysis")],
            ]
        ),
    )


async def evening_reminder(context: ContextTypes.DEFAULT_TYPE) -> None:
    db, _, _ = services(context)
    telegram_id = int(context.job.data["telegram_id"])
    user = await db.get_user(telegram_id)
    if not user:
        return
    tasks = await db.get_tasks_for_day(telegram_id, today_str(user))
    text = (
        f"Вечерний итог. Сейчас <b>{now_hhmm(user)}</b>.\n"
        "Давай подведём день, сохраним отчёт и сделаем AI-разбор.\n\n"
        + progress_summary(tasks)
    )
    await context.bot.send_message(
        chat_id=telegram_id,
        text=text,
        parse_mode=ParseMode.HTML,
        reply_markup=InlineKeyboardMarkup(
            [
                [InlineKeyboardButton("✍️ Вечерний отчёт", callback_data="checkin:evening")],
                [InlineKeyboardButton("🧠 AI-анализ", callback_data="open:analysis")],
                [InlineKeyboardButton("📋 План", callback_data="open:plan")],
            ]
        ),
    )


async def unknown_text(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if not update.message:
        return
    if context.user_data.get("settings_target"):
        return
    await update.message.reply_text(
        "Я понял сообщение, но лучше используй кнопки ниже: так я точнее сохраню план и прогресс.",
        reply_markup=main_menu(),
    )


async def error_handler(update: object, context: ContextTypes.DEFAULT_TYPE) -> None:
    logger.exception("Unhandled error", exc_info=context.error)
    if isinstance(update, Update):
        target_message = update.effective_message
        if target_message:
            await target_message.reply_text(
                "Что-то пошло не так. Попробуй ещё раз или нажми /menu.",
                reply_markup=main_menu(),
            )


def build_application(settings: Settings) -> Application:
    db = Database(settings.db_path)
    ai = AIService(settings.openai_api_key, settings.openai_model)

    application = (
        ApplicationBuilder()
        .token(settings.telegram_token)
        .defaults(Defaults(parse_mode=ParseMode.HTML))
        .concurrent_updates(False)
        .post_init(post_init)
        .build()
    )

    application.bot_data["db"] = db
    application.bot_data["ai"] = ai
    application.bot_data["settings"] = settings

    onboarding_conv = ConversationHandler(
        entry_points=[CommandHandler("start", start)],
        states={
            ONBOARD_BIO: [MessageHandler(filters.TEXT & ~filters.COMMAND & ~filters.Regex(f"^{re.escape(BTN_CANCEL)}$"), onboarding_bio)],
            ONBOARD_HOBBIES: [MessageHandler(filters.TEXT & ~filters.COMMAND & ~filters.Regex(f"^{re.escape(BTN_CANCEL)}$"), onboarding_hobbies)],
            ONBOARD_TIMEZONE: [
                CallbackQueryHandler(onboarding_timezone_callback, pattern=r"^tz:"),
                MessageHandler(filters.TEXT & ~filters.COMMAND & ~filters.Regex(f"^{re.escape(BTN_CANCEL)}$"), onboarding_timezone_text),
            ],
            ONBOARD_MORNING: [MessageHandler(filters.TEXT & ~filters.COMMAND & ~filters.Regex(f"^{re.escape(BTN_CANCEL)}$"), onboarding_morning)],
            ONBOARD_MIDDAY: [MessageHandler(filters.TEXT & ~filters.COMMAND & ~filters.Regex(f"^{re.escape(BTN_CANCEL)}$"), onboarding_midday)],
            ONBOARD_EVENING: [MessageHandler(filters.TEXT & ~filters.COMMAND & ~filters.Regex(f"^{re.escape(BTN_CANCEL)}$"), onboarding_evening)],
        },
        fallbacks=[
            MessageHandler(filters.Regex(f"^{re.escape(BTN_CANCEL)}$"), cancel),
            CommandHandler("cancel", cancel),
        ],
        allow_reentry=True,
    )

    add_task_conv = ConversationHandler(
        entry_points=[
            CommandHandler("add", add_task_entry),
            MessageHandler(filters.Regex(f"^{re.escape(BTN_ADD)}$"), add_task_entry),
            CallbackQueryHandler(add_task_entry, pattern=r"^open:add$"),
        ],
        states={
            ADD_TITLE: [MessageHandler(filters.TEXT & ~filters.COMMAND & ~filters.Regex(f"^{re.escape(BTN_CANCEL)}$"), add_task_title)],
            ADD_PRIORITY: [MessageHandler(filters.TEXT & ~filters.COMMAND & ~filters.Regex(f"^{re.escape(BTN_CANCEL)}$"), add_task_priority)],
            ADD_DURATION: [MessageHandler(filters.TEXT & ~filters.COMMAND & ~filters.Regex(f"^{re.escape(BTN_CANCEL)}$"), add_task_duration)],
            ADD_NOTE: [MessageHandler(filters.TEXT & ~filters.COMMAND & ~filters.Regex(f"^{re.escape(BTN_CANCEL)}$"), add_task_note)],
        },
        fallbacks=[
            MessageHandler(filters.Regex(f"^{re.escape(BTN_CANCEL)}$"), cancel),
            CommandHandler("cancel", cancel),
        ],
        allow_reentry=True,
    )

    coach_conv = ConversationHandler(
        entry_points=[
            CommandHandler("coach", coach_entry),
            MessageHandler(filters.Regex(f"^{re.escape(BTN_COACH)}$"), coach_entry),
        ],
        states={
            COACH_QUESTION: [MessageHandler(filters.TEXT & ~filters.COMMAND & ~filters.Regex(f"^{re.escape(BTN_CANCEL)}$"), coach_question)],
        },
        fallbacks=[
            MessageHandler(filters.Regex(f"^{re.escape(BTN_CANCEL)}$"), cancel),
            CommandHandler("cancel", cancel),
        ],
        allow_reentry=True,
    )

    checkin_conv = ConversationHandler(
        entry_points=[
            CommandHandler("report", checkin_entry),
            CallbackQueryHandler(checkin_entry, pattern=r"^checkin:(midday|evening)$"),
        ],
        states={
            CHECKIN_SUMMARY: [MessageHandler(filters.TEXT & ~filters.COMMAND & ~filters.Regex(f"^{re.escape(BTN_CANCEL)}$"), checkin_summary)],
        },
        fallbacks=[
            MessageHandler(filters.Regex(f"^{re.escape(BTN_CANCEL)}$"), cancel),
            CommandHandler("cancel", cancel),
        ],
        allow_reentry=True,
    )

    application.add_handler(onboarding_conv)
    application.add_handler(add_task_conv)
    application.add_handler(coach_conv)
    application.add_handler(checkin_conv)

    application.add_handler(CommandHandler("menu", menu))
    application.add_handler(CommandHandler("plan", show_plan))
    application.add_handler(CommandHandler("analysis", analysis_today))
    application.add_handler(CommandHandler("export", export_data))
    application.add_handler(CommandHandler("settings", settings_entry))

    application.add_handler(MessageHandler(filters.Regex(f"^{re.escape(BTN_PLAN)}$"), show_plan))
    application.add_handler(MessageHandler(filters.Regex(f"^{re.escape(BTN_DONE)}$"), mark_done_prompt))
    application.add_handler(MessageHandler(filters.Regex(f"^{re.escape(BTN_AI)}$"), analysis_today))
    application.add_handler(MessageHandler(filters.Regex(f"^{re.escape(BTN_EXPORT)}$"), export_data))
    application.add_handler(MessageHandler(filters.Regex(f"^{re.escape(BTN_SETTINGS)}$"), settings_entry))

    application.add_handler(CallbackQueryHandler(task_callback, pattern=r"^(done|skip|delete):\d+$"))
    application.add_handler(CallbackQueryHandler(settings_callback, pattern=r"^(schedule:|tz:)"))
    application.add_handler(CallbackQueryHandler(reminder_action_router, pattern=r"^open:(plan|analysis)$"))

    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, maybe_handle_settings_value), group=1)

    application.add_error_handler(error_handler)
    return application


def main() -> None:
    load_dotenv()
    try:
        settings = load_settings()
    except ConfigError as exc:
        raise SystemExit(str(exc)) from exc

    application = build_application(settings)
    application.run_polling(allowed_updates=Update.ALL_TYPES)


if __name__ == "__main__":
    main()
