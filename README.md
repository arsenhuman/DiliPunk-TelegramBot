# 🦆🤘 PunkDuck

A Telegram bot for the **DiliRock** summer music festival organizing chat. PunkDuck quietly
logs every message in the group and, on command, produces an AI-generated digest of what
happened — key topics, decisions made, and open questions — so nobody has to scroll through
hundreds of messages to catch up.

It also has a personality: every so often it randomly asks the chat for a cigarette, with an
inline button for someone to hand one over. Because organizing a festival is stressful and a
punk duck deserves a smoke break.

## Features

- **`/summary`** — generates a digest of the chat for a given period using OpenAI's API
  - `/summary` — since the last summary (or last 24h if none exist yet)
  - `/summary 6h` — last 6 hours
  - `/summary 3d` — last 3 days
- **Reply-aware context** — when summarizing, the bot reconstructs who replied to whom, so the
  AI has the conversational thread, not just a flat list of messages
- **🚬 Cigarette requests** — a small chance on every message that the bot asks the chat for a
  cigarette, with a first-come-first-served inline button
- Fully containerized with **Docker Compose** (bot + PostgreSQL)
- SQL migrations tracked and applied explicitly, no auto-magic on startup

## Tech stack

| Layer | Choice |
|---|---|
| Runtime | Node.js + [Telegraf](https://telegraf.js.org/) (long polling, no webhook/public IP needed) |
| Database | PostgreSQL |
| AI | OpenAI API |
| Containerization | Docker / Docker Compose |
| Infra (planned) | Terraform (GCP) + Ansible |

## Project structure

```
PunkDuck-TelegramBot/
├── bot/
│   ├── src/
│   │   ├── index.js        # entrypoint, starts Telegraf
│   │   ├── settings.js     # centralized env var loading
│   │   ├── db.js           # PostgreSQL connection & queries
│   │   ├── handlers.js     # message/command handlers
│   │   ├── summarize.js    # OpenAI summary generation
│   │   ├── persona.js      # system prompts for the AI
│   │   ├── messages.js     # all user-facing bot text in one place
│   │   └── cigarette.js    # the cigarette-request feature
│   ├── migrations/         # versioned SQL migrations
│   ├── Dockerfile
│   └── package.json
├── docker-compose.yaml
└── .env.example
```

## Getting started

### 0. Create the bot on Telegram (one-time)

1. Open [@BotFather](https://t.me/BotFather), send `/newbot`, follow the prompts. Save the
   token — that's your `BOT_TOKEN`.
2. **Important:** send BotFather `/setprivacy`, select your bot, then **Disable**.

   By default, Telegram bots in groups only see messages that are commands or that explicitly
   mention the bot — regular chat messages stay invisible to it. Without disabling this, the
   bot will never see the conversation it's supposed to be summarizing. This is the most common
   reason a freshly deployed bot looks like it "isn't saving anything."

3. Add the bot to the target group as a regular member.

### 1. Configure environment variables

```bash
cp .env.example .env
```

Fill in:

```env
BOT_TOKEN=your_botfather_token
OPENAI_API_KEY=your_openai_key
OPENAI_MODEL=gpt-4o-mini

PGUSER=punkduck
PGPASSWORD=choose_a_password
PGDATABASE=punkduck
```

> Don't wrap values in quotes — `BOT_TOKEN=abc123`, not `BOT_TOKEN="abc123"`. Also don't add
> spaces around `=`.

### 2. Build the images

```bash
docker compose build
```

### 3. Start PostgreSQL and wait for it to be healthy

```bash
docker compose up -d postgres
docker compose ps   # wait for STATUS to show (healthy)
```

### 4. Apply database migrations

Migrations are **not** applied automatically — this is intentional, so you always know exactly
when the schema changes.

```bash
docker compose run --rm migrate
```

### 5. Start the bot

```bash
docker compose up -d bot
docker compose logs -f bot
```

You should see the bot report it's running. Send `/start` to it in Telegram to confirm.

## Updating a running deployment

```bash
git pull
docker compose build bot
docker compose run --rm migrate   # only if new migration files were added
docker compose up -d bot
```

## Database backups

Postgres data lives in a Docker named volume, which survives container restarts but not a lost
disk. A manual dump:

```bash
docker compose exec postgres pg_dump -U $PGUSER $PGDATABASE > backup_$(date +%F).sql
```

Automating this (e.g. via cron + upload to a cloud bucket) is on the roadmap.

## Roadmap

- [ ] Terraform module to provision the GCP VM
- [ ] Ansible playbook for hands-off deployment
- [ ] Secret management beyond plain `.env` files
- [ ] More bot personality features

## License

MIT
