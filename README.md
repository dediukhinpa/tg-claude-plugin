# tg-channel — Telegram plugin for Claude Code

MCP server that connects Claude Code to Telegram. Send a message → Claude thinks → replies, sets reactions, handles voice, photos, and files.

```
You ──── Telegram ────► Claude Code ──► your codebase, tools, APIs
         ◄────────────
```

**What Claude can do via this plugin:**
- `reply` — send text back (Markdown supported)
- `react` — set emoji reaction on any message
- `edit_message` — update a previously sent reply
- `download_attachment` — read photos and documents you send
- Voice messages: transcribed via Groq Whisper before Claude sees them

---

## Установка одним промптом

Скопируй блок ниже в **новую сессию Claude Code**, подставь свои значения и отправь. Claude сам склонирует репо, установит зависимости и запустит бота.

```
Настрой tg-channel — Telegram-плагин для Claude Code.

Мои данные:
  TELEGRAM_BOT_TOKEN = <токен от @BotFather>      # формат: 1234567890:AAH...
  MY_TELEGRAM_USER_ID = <мой Telegram user ID>    # открой @userinfobot → нажми Start → скопируй Id
  GROQ_API_KEY = <ключ Groq>                       # опционально, для голоса — пропусти если не нужно

Шаги:
1. Склонируй https://github.com/dediukhinpa/tg-claude-plugin в ~/.claude/plugins/tg-channel/
2. Выполни `bun install` в этой директории (если bun не установлен: curl -fsSL https://bun.sh/install | bash)
3. Извлеки bot_id из TELEGRAM_BOT_TOKEN — это число до двоеточия (например "1234567890:AAH..." → 1234567890)
4. Создай ~/.claude/plugins/tg-channel/config.json:
   {
     "bot_id": <извлечённый_bot_id>,
     "dm_only": true,
     "allowed_user_ids": [<MY_TELEGRAM_USER_ID>],
     "allowed_chat_ids": [<MY_TELEGRAM_USER_ID>]
   }
5. Создай директорию для секретов: mkdir -p ~/.claude/channels/tg-channel-default/secrets
6. Запиши токен в файл:
     echo '<TELEGRAM_BOT_TOKEN>' > ~/.claude/channels/tg-channel-default/secrets/telegram-token
     chmod 600 ~/.claude/channels/tg-channel-default/secrets/telegram-token
7. Если указан GROQ_API_KEY:
     echo '<GROQ_API_KEY>' > ~/.claude/channels/tg-channel-default/secrets/groq-api-key
     chmod 600 ~/.claude/channels/tg-channel-default/secrets/groq-api-key
8. Выстави переменные и запусти бота:
     export TELEGRAM_BOT_TOKEN=<TELEGRAM_BOT_TOKEN>
     export TELEGRAM_STATE_DIR=~/.claude/channels/tg-channel-default
     cd ~/.claude/plugins/tg-channel
     claude --dangerously-skip-permissions --dangerously-load-development-channels server:tg-channel

Подтверждай каждый шаг по мере выполнения. В конце покажи последние 10 строк вывода бота.
```

> **Важно:** `--dangerously-skip-permissions` отключает запросы разрешений у Claude. Используй только в отдельной сессии для бота, не в основном рабочем пространстве.

---

## Manual setup

### Prerequisites

- [Bun](https://bun.sh) >= 1.0
- Claude Code CLI (`npm i -g @anthropic/claude-code`)
- A Telegram bot token from [@BotFather](https://t.me/BotFather)
- Твой Telegram user ID — открой [@userinfobot](https://t.me/userinfobot), нажми **Start**, скопируй `Id`

### Install

```bash
git clone https://github.com/dediukhinpa/tg-claude-plugin ~/.claude/plugins/tg-channel
cd ~/.claude/plugins/tg-channel
bun install
```

### Configure

```bash
# 1. Create config (replace values)
BOT_TOKEN="1234567890:AAH_your_token_here"
BOT_ID="${BOT_TOKEN%%:*}"    # extracts numeric part before the colon
USER_ID="your_telegram_user_id"

cat > config.json <<EOF
{
  "bot_id": $BOT_ID,
  "dm_only": true,
  "allowed_user_ids": [$USER_ID],
  "allowed_chat_ids": [$USER_ID]
}
EOF

# 2. Store token securely
STATE_DIR="$HOME/.claude/channels/tg-channel-default"
mkdir -p "$STATE_DIR/secrets"
echo "$BOT_TOKEN" > "$STATE_DIR/secrets/telegram-token"
chmod 600 "$STATE_DIR/secrets/telegram-token"

# 3. Optional: Groq for voice transcription
# echo "gsk_your_groq_key" > "$STATE_DIR/secrets/groq-api-key"
# chmod 600 "$STATE_DIR/secrets/groq-api-key"
```

### Launch

```bash
export TELEGRAM_BOT_TOKEN="$(cat $HOME/.claude/channels/tg-channel-default/secrets/telegram-token)"
export TELEGRAM_STATE_DIR="$HOME/.claude/channels/tg-channel-default"

claude --dangerously-skip-permissions \
       --dangerously-load-development-channels \
       server:tg-channel
```

The bot starts polling. Send it a message on Telegram.

---

## Configuration reference

`config.json` (full options):

```jsonc
{
  "bot_id": 1234567890,           // required: numeric bot ID (before colon in token)
  "dm_only": true,                // only accept DMs (recommended)
  "allowed_user_ids": [111222],   // allowlist of Telegram user IDs
  "allowed_chat_ids": [111222],   // allowlist of chat IDs (usually same as user IDs for DMs)

  "status": {
    "enabled": false,             // show typing indicator while Claude thinks
    "interval_ms": 300,
    "delete_on_complete": true    // delete typing message after reply
  },

  "voice": {
    "provider": "groq",           // groq is the only supported provider
    "language": "ru",             // BCP-47 language code for Whisper
    "model": "whisper-large-v3-turbo"
  },

  "webhook": {
    "enabled": true,              // internal webhook for inter-agent signals
    "host": "127.0.0.1",
    "port": 8089
  }
}
```

Environment variables (override config.json):

| Variable | Description |
|----------|-------------|
| `TELEGRAM_BOT_TOKEN` | Bot token from BotFather |
| `TELEGRAM_STATE_DIR` | Directory for state files and secrets |
| `GROQ_API_KEY` | Groq API key for voice transcription |

---

## How it works

```
Telegram update
     |
     v
  gate.ts         <- checks allowed_user_ids / dm_only
     |
     v
  handlers.ts     <- text / voice / photo / document handlers
     |  voice -> Groq Whisper -> transcript
     |  photo -> downloads file, adds <media> tag
     |
     v
  MCP channel     <- sends structured message to Claude Code
     |
     v
  Claude thinks
     |
     +-- reply(chat_id, text)         -> Telegram message
     +-- react(chat_id, msg_id, "👍") -> emoji reaction
     +-- edit_message(...)            -> update sent message
     +-- download_attachment(...)     -> read file bytes
```

The plugin is an MCP server. Claude Code loads it via `--dangerously-load-development-channels server:tg-channel` and receives inbound Telegram messages as channel events. Claude calls the tools above to respond.

---

## Persistent background agent

To keep the bot running after you close the terminal, use tmux + systemd. See [lesson-05 of claude-ops-course](https://github.com/dediukhinpa/claude-ops-course/tree/main/lesson-05) for production-grade setup with watchdog, auto-restart, and orphan process cleanup.

Quick tmux version:
```bash
tmux new-session -d -s tg-bot \
  -e "TELEGRAM_BOT_TOKEN=$(cat ~/.claude/channels/tg-channel-default/secrets/telegram-token)" \
  -e "TELEGRAM_STATE_DIR=$HOME/.claude/channels/tg-channel-default" \
  "claude --dangerously-skip-permissions --dangerously-load-development-channels server:tg-channel"

tmux attach -t tg-bot   # detach with Ctrl+B, D
```

---

## License

Apache 2.0
