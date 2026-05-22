# Schedule Bot — Development Process Documentation

## Project Overview

**Name:** Schedule Bot  
**Type:** Telegram chatbot  
**Stack:** Python · Django · python-telegram-bot · SQLite  
**Goal:** Allow users to add, view, and delete weekly schedule events via Telegram

---

## Step 1 — Choosing the Project Type

The first decision was which type of chatbot to build. The options were Desktop (Tkinter/PyQt), Flask, Django, Telegram Bot, or Hybrid.

I chose **Telegram Bot** because:
- Telegram is already installed on most devices — no extra UI needed
- The `python-telegram-bot` library provides a clean async API
- It pairs naturally with Django's ORM for data persistence

---

## Step 2 — Setting Up the Django Project

**Commands run:**
```bash
django-admin startproject scheduler_project
cd scheduler_project
python manage.py startapp schedulerpython
```

A separate app `schedulerpython` was created to hold the `Event` model. This keeps the data layer isolated from the bot logic.

**Why Django instead of just a plain Python script?**  
Django gives us a free ORM, migrations, and admin panel. The bot logic stays in a custom management command (`runbot.py`), so we run it with `python manage.py runbot` — no separate scripts.

---

## Step 3 — Designing the Data Model

File: `schedulerpython/models.py`

```python
from django.db import models

class Event(models.Model):
    day  = models.CharField(max_length=20)
    text = models.TextField()
```

**Why this structure?**
- `day` stores the weekday as a string (e.g. `"monday"`)
- `text` stores the full event description including time

This is the simplest model that satisfies the requirements. It can be extended later (add `user_id`, `time`, `created_at`).

**Running migrations:**
```bash
python manage.py makemigrations
python manage.py migrate
```

---

## Step 4 — The async/sync Problem

Django ORM is **synchronous**. The Telegram bot uses **async** handlers (Python's `asyncio`). Calling ORM methods directly inside async functions raises a `SynchronousOnlyOperation` error.

**Solution:** `asgiref.sync.sync_to_async`

```python
from asgiref.sync import sync_to_async

@sync_to_async
def create_event(day, text):
    return Event.objects.create(day=day, text=text)

@sync_to_async
def get_all_events():
    return list(Event.objects.all())
```

Each ORM function is wrapped with `@sync_to_async`, converting it into a coroutine that can be safely `await`-ed inside async handlers.

**Important:** `.all()` and `.filter()` return lazy QuerySets. The `list()` call forces evaluation inside the wrapper, before we leave the synchronous context.

---

## Step 5 — Building the Bot Handlers

File: `bot/management/commands/runbot.py`

### `/start` handler

```python
async def start(update, context):
    await update.message.reply_text(
        "📅 Таңдаңыз әрекет",
        reply_markup=main_keyboard()
    )
```

Sends a persistent keyboard with 4 buttons using `ReplyKeyboardMarkup`. `resize_keyboard=True` makes the buttons smaller and more natural-looking.

### `/add` handler

```python
async def add(update, context):
    if len(context.args) < 2:
        await update.message.reply_text("Мысал: /add monday 9:00 Math")
        return

    day = context.args[0].lower()
    text = " ".join(context.args[1:])

    if day not in valid_days:
        await update.message.reply_text("Қате күн")
        return

    await create_event(day, text)
    await update.message.reply_text("✅ Қосылды")
```

`context.args` gives a list of words after the command. Joining `args[1:]` allows the event text to contain spaces.

### `/list` handler

Fetches all events and formats them as a single message:
```
📅 Кесте:
monday: 9:00 Math
friday: 14:00 Meeting
```

### `/delete` handler

Takes the day and 1-based index. Fetches events for that day, validates the index, then deletes.

```python
events = await get_day_events(day)
await delete_event(events[index])
```

### Text / keyboard button handler

`MessageHandler(filters.TEXT & ~filters.COMMAND, text_handler)` catches all plain text messages that are not commands.

```python
async def text_handler(update, context):
    text = update.message.text
    if text == "📅 Меню":
        await update.message.reply_text("📌 Командалар:\n/add\n/delete\n/list")
    elif text == "📋 Кесте":
        await list_events(update, context)
    ...
```

This makes the keyboard buttons functional without any extra setup.

---

## Step 6 — Registering Handlers and Running

```python
def run_bot():
    app = ApplicationBuilder().token(TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("add", add))
    app.add_handler(CommandHandler("list", list_events))
    app.add_handler(CommandHandler("delete", delete))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, text_handler))

    app.run_polling()
```

`run_polling()` starts a loop that keeps asking Telegram for new updates. It runs until the process is interrupted (Ctrl+C).

---

## Step 7 — Error Handling Summary

| Situation | Handler | Response |
|---|---|---|
| `/add` with no args | `add()` | Shows usage example |
| Invalid day name | `add()` | "Қате күн" |
| `/delete` with invalid index | `delete()` | "Қате индекс" |
| Any exception in `/delete` | `except Exception` | Shows usage example |
| Empty schedule | `list_events()` | "мұнда ештеңе жоқ.." |
| Unknown text input | `text_handler()` — no match | (silently ignored / could add a fallback message) |

---

## Step 8 — Project Structure Finalization

```
scheduler_project/
├── manage.py
├── requirements.txt
├── README.md
├── schedulerpython/
│   ├── models.py
│   ├── admin.py
│   └── migrations/
│       └── 0001_initial.py
└── bot/
    └── management/
        └── commands/
            └── runbot.py
```

The `requirements.txt` was generated with:
```bash
pip freeze > requirements.txt
```

Minimum contents:
```
django>=4.0
python-telegram-bot>=20.0
asgiref>=3.6
```

---

## Challenges Faced

### 1. sync_to_async confusion
Initially calling `Event.objects.create(...)` directly inside an async handler caused a crash. The fix was wrapping every ORM call in its own `@sync_to_async` function and evaluating QuerySets inside those wrappers (not outside).

### 2. Lazy QuerySet evaluation
`Event.objects.filter(day=day)` doesn't run the SQL until the results are accessed. Since Django blocks this access outside its sync context, the `list()` call inside the wrapper is essential.

### 3. Running the bot as a Django management command
Instead of a standalone Python script, the bot runs as `python manage.py runbot`. This ensures Django's app registry is loaded (so models work) without needing to call `django.setup()` manually.

---

## What I Would Add With More Time

- **Per-user storage** — add a `user_id` field to `Event` so each Telegram user has their own schedule
- **Reminders** — use APScheduler or Celery to send a morning message with today's events
- **Inline keyboard** — let users tap to delete instead of typing `/delete monday 1`
- **Django Admin** — already available; could be used to manage all events from a browser
- **Time parsing** — validate that the time format is correct (e.g. `HH:MM`) before saving
