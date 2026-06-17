# tg-channel — Telegram-плагин для Claude Code

MCP-сервер, который подключает Claude Code к Telegram. Отправляешь сообщение → Claude думает → отвечает, ставит реакции, обрабатывает голос, фото и файлы.

```
Ты ──── Telegram ────► Claude Code ──► твой код, инструменты, API
        ◄────────────
```

**Что Claude умеет через этот плагин:**
- `reply` — отправить ответ (поддерживается Markdown)
- `react` — поставить эмодзи-реакцию на сообщение
- `edit_message` — отредактировать уже отправленный ответ
- `download_attachment` — прочитать фото или документ
- Голосовые сообщения: расшифровываются через Groq Whisper до того, как Claude их видит

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

**Совместимость:** промпт работает на Linux и macOS. На Windows — только через [WSL](https://learn.microsoft.com/ru-ru/windows/wsl/install) или Git Bash (команды `chmod`, `export`, `curl | bash` недоступны в PowerShell/CMD).

---

## Ручная установка

### Что нужно

- [Bun](https://bun.sh) >= 1.0
- Claude Code CLI (`npm i -g @anthropic/claude-code`)
- Токен Telegram-бота от [@BotFather](https://t.me/BotFather)
- Твой Telegram user ID — открой [@userinfobot](https://t.me/userinfobot), нажми **Start**, скопируй `Id`

### Установка

```bash
git clone https://github.com/dediukhinpa/tg-claude-plugin ~/.claude/plugins/tg-channel
cd ~/.claude/plugins/tg-channel
bun install
```

### Конфигурация

```bash
# 1. Создаём config.json (замени значения)
BOT_TOKEN="1234567890:AAH_your_token_here"
BOT_ID="${BOT_TOKEN%%:*}"    # число до двоеточия
USER_ID="твой_telegram_user_id"

cat > config.json <<EOF
{
  "bot_id": $BOT_ID,
  "dm_only": true,
  "allowed_user_ids": [$USER_ID],
  "allowed_chat_ids": [$USER_ID]
}
EOF

# 2. Сохраняем токен в файл
STATE_DIR="$HOME/.claude/channels/tg-channel-default"
mkdir -p "$STATE_DIR/secrets"
echo "$BOT_TOKEN" > "$STATE_DIR/secrets/telegram-token"
chmod 600 "$STATE_DIR/secrets/telegram-token"

# 3. Опционально: Groq для расшифровки голоса
# echo "gsk_your_groq_key" > "$STATE_DIR/secrets/groq-api-key"
# chmod 600 "$STATE_DIR/secrets/groq-api-key"
```

### Запуск

```bash
export TELEGRAM_BOT_TOKEN="$(cat $HOME/.claude/channels/tg-channel-default/secrets/telegram-token)"
export TELEGRAM_STATE_DIR="$HOME/.claude/channels/tg-channel-default"

claude --dangerously-skip-permissions \
       --dangerously-load-development-channels \
       server:tg-channel
```

Бот начинает поллинг. Напиши ему в Telegram.

---

## Справочник по конфигурации

`config.json` (все параметры):

```jsonc
{
  "bot_id": 1234567890,           // обязательно: числовой ID бота (часть токена до двоеточия)
  "dm_only": true,                // принимать только личные сообщения (рекомендуется)
  "allowed_user_ids": [111222],   // разрешённые Telegram user ID
  "allowed_chat_ids": [111222],   // разрешённые chat ID (обычно совпадает с user ID для ЛС)

  "status": {
    "enabled": false,             // показывать индикатор печати пока Claude думает
    "interval_ms": 300,
    "delete_on_complete": true    // удалить сообщение-индикатор после ответа
  },

  "voice": {
    "provider": "groq",           // единственный поддерживаемый провайдер
    "language": "ru",             // язык для Whisper (BCP-47)
    "model": "whisper-large-v3-turbo"
  },

  "webhook": {
    "enabled": true,              // внутренний вебхук для межагентных сигналов
    "host": "127.0.0.1",
    "port": 8089
  }
}
```

Переменные окружения (перекрывают config.json):

| Переменная | Описание |
|------------|----------|
| `TELEGRAM_BOT_TOKEN` | Токен бота от BotFather |
| `TELEGRAM_STATE_DIR` | Директория для состояния и секретов |
| `GROQ_API_KEY` | Groq API ключ для расшифровки голоса |

---

## Как это работает

```
Сообщение в Telegram
     |
     v
  gate.ts         <- проверяет allowed_user_ids / dm_only
     |
     v
  handlers.ts     <- обработчики текста / голоса / фото / документов
     |  голос -> Groq Whisper -> транскрипт
     |  фото  -> скачивает файл, добавляет тег <media>
     |
     v
  MCP channel     <- передаёт структурированное сообщение в Claude Code
     |
     v
  Claude думает
     |
     +-- reply(chat_id, text)         -> сообщение в Telegram
     +-- react(chat_id, msg_id, "👍") -> эмодзи-реакция
     +-- edit_message(...)            -> редактирование ответа
     +-- download_attachment(...)     -> чтение байт файла
```

Плагин — MCP-сервер. Claude Code загружает его через `--dangerously-load-development-channels server:tg-channel` и получает входящие сообщения из Telegram как channel events. Для ответа Claude вызывает инструменты выше.

---

## Постоянный фоновый агент

Чтобы бот продолжал работать после закрытия терминала, используй tmux + systemd. Для продакшена рекомендуется добавить watchdog-скрипт, который следит за состоянием сессии и перезапускает бота при зависании.

Быстрый вариант с tmux:
```bash
tmux new-session -d -s tg-bot \
  -e "TELEGRAM_BOT_TOKEN=$(cat ~/.claude/channels/tg-channel-default/secrets/telegram-token)" \
  -e "TELEGRAM_STATE_DIR=$HOME/.claude/channels/tg-channel-default" \
  "claude --dangerously-skip-permissions --dangerously-load-development-channels server:tg-channel"

tmux attach -t tg-bot   # отсоединиться: Ctrl+B, затем D
```

---

## Лицензия

Apache 2.0
