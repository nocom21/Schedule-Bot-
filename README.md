# 📅 Schedule Bot

A Telegram chatbot for managing your weekly schedule, built with Python, Django, and the Telegram Bot API.

---

## Description

Schedule Bot lets you add, view, and delete personal events for any day of the week — all from within Telegram. No external app needed. Events are stored persistently in a SQLite database via Django ORM.

---

## Technologies

| Technology | Purpose |
|---|---|
| Python 3.11 | Core language |
| Django 4.x | ORM and data models |
| python-telegram-bot | Telegram Bot API integration |
| SQLite | Local persistent database |
| asgiref | `sync_to_async` bridge for Django ORM |

---

## Installation

**1. Clone the repository**
```bash
git clone https://github.com/your-username/schedule-bot.git
cd schedule-bot
```

**2. Create and activate a virtual environment**
```bash
python -m venv venv
source venv/bin/activate        # Linux / macOS
venv\Scripts\activate           # Windows
```

**3. Install dependencies**
```bash
pip install -r requirements.txt
```

**4. Run Django migrations**
```bash
python manage.py makemigrations
python manage.py migrate
```

---

## Configuration

Open `bot/management/commands/runbot.py` and replace the token value:

```python
TOKEN = "your-telegram-bot-token-here"
```

To get a token, message [@BotFather](https://t.me/BotFather) on Telegram and create a new bot.

---

## Running the Bot

```bash
python manage.py runbot
```

The bot will start polling for messages. Keep this terminal window open.

---

## Commands

| Command | Description |
|---|---|
| `/start` | Show the main keyboard menu |
| `/add <day> <text>` | Add an event to a weekday |
| `/list` | Show all saved events |
| `/delete <day> <index>` | Delete event #index on that day |

**Keyboard buttons:**

| Button | Action |
|---|---|
| 📅 Меню | Show available commands |
| 📋 Кесте | Display all events (same as `/list`) |
| ➕ Қосу | Show hint for adding events |
| ❌ Жою | Show hint for deleting events |

**Valid days:** `monday`, `tuesday`, `wednesday`, `thursday`, `friday`, `saturday`, `sunday`

---

## Usage Examples

```
/add monday 9:00 Math lecture
✅ Қосылды

/add friday 14:00 Team meeting
✅ Қосылды

/list
📅 Кесте:
monday: 9:00 Math lecture
friday: 14:00 Team meeting

/delete monday 1
🗑 Жойылды
```

---

## Project Structure

```
schedule-bot/
├── manage.py
├── requirements.txt
├── README.md
├── schedulerpython/
│   ├── __init__.py
│   ├── models.py          # Event model (day, text)
│   ├── admin.py
│   └── migrations/
│       └── 0001_initial.py
└── bot/
    ├── __init__.py
    └── management/
        └── commands/
            └── runbot.py  # All bot logic
```

---

## Data Model

```python
class Event(models.Model):
    day  = models.CharField(max_length=20)   # e.g. "monday"
    text = models.TextField()                # e.g. "9:00 Math"
```

---

## Error Handling

The bot handles:
- **Unknown commands** — replies with a hint
- **Missing arguments** — shows usage example
- **Invalid day names** — returns "Қате күн" (Wrong day)
- **Out-of-range index** — returns "Қате индекс" (Wrong index)
- **Empty schedule** — returns "мұнда ештеңе жоқ.." (Nothing here)

---

## Requirements

```
django>=4.0
python-telegram-bot>=20.0
asgiref>=3.6
```

---

## Screenshots

> Add screenshots of your bot in action here after running it.

---

## License

MIT
