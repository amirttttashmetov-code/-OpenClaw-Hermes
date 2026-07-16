---
name: openclaw-to-hermes-migration
description: Полная миграция с OpenClaw на Hermes Agent — удаление OpenClaw с сохранением всех данных и ботов, установка Hermes, настройка Telegram и DeepSeek.
version: 1.1.0
author: Рамирес / Pixel Bot
platforms: [linux]
---

# Миграция OpenClaw → Hermes Agent

Полный перенос AI-агента с OpenClaw на Hermes Agent на Linux-сервере (Ubuntu 24.04).
OpenClaw удаляется полностью, **все боты сохраняются и продолжают работать**.

## Когда применять

- OpenClaw жрёт 350+ MB RAM (Node.js), а Hermes — 80 MB (Python)
- Нужен self-improving агент со skills и memory
- Нужна поддержка 20+ мессенджеров
- Хочется избавиться от Node.js на сервере

## Порядок действий

### 1. Установка Hermes

```bash
apt install -y pipx
pipx install hermes-agent
export PATH="/root/.local/bin:$PATH"
hermes --version  # должно быть 0.18+
```

### 2. Конфигурация

```bash
mkdir -p ~/.hermes

# .env — ВАЖНО: скопируй ключи из старого .env ботов!
cat > ~/.hermes/.env << 'EOF'
DEEPSEEK_API_KEY=sk-ВАШ_КЛЮЧ_ИЗ_OPENCLAW_ENV
TELEGRAM_BOT_TOKEN=НОМЕР:ТОКЕН_ОТ_OPENCLAW
GATEWAY_ALLOW_ALL_USERS=true
TELEGRAM_HOME_CHANNEL=ВАШ_CHAT_ID
EOF

# Конфиг
cat > ~/.hermes/config.yaml << 'EOF'
model:
  default: deepseek/deepseek-v4-pro
  provider: deepseek
  base_url: https://api.deepseek.com/v1
gateway:
  platforms:
    telegram:
      enabled: true
EOF
```

### 3. Сохранение ВСЕХ данных и скиллов

```bash
# === ШАГ 3.1: Сохраняем workspace ===
cp -r /root/.openclaw/workspace /root/pixel-workspace
echo "✅ Workspace сохранён"

# === ШАГ 3.2: Сохраняем ВСЕ скиллы (КРИТИЧЕСКИ!) ===
# Скиллы могут быть в 4 местах. Сохраняем КАЖДОЕ:

# 1. Скиллы в workspace
if [ -d /root/.openclaw/workspace/skills ]; then
    cp -r /root/.openclaw/workspace/skills /root/pixel-workspace/skills 2>/dev/null
    echo "✅ workspace/skills сохранён"
fi

# 2. Скиллы в plugin-skills (установленные через хаб)
if [ -d /root/.openclaw/plugin-skills ]; then
    cp -r /root/.openclaw/plugin-skills /root/pixel-workspace/plugin-skills-backup 2>/dev/null
    echo "✅ plugin-skills сохранён ($(ls /root/.openclaw/plugin-skills/ | wc -l) шт.)"
fi

# 3. Скиллы в skill-workshop (созданные агентом)
if [ -d /root/.openclaw/skill-workshop ]; then
    cp -r /root/.openclaw/skill-workshop /root/pixel-workspace/skill-workshop-backup 2>/dev/null
    echo "✅ skill-workshop сохранён"
fi

# 4. Любые SKILL.md файлы в /root/.openclaw (кроме venv и node_modules)
find /root/.openclaw -name "SKILL.md" -not -path "*/node_modules/*" -not -path "*/.venv/*" | while read f; do
    dest="/root/pixel-workspace/skills-backup/$(echo $f | md5sum | cut -c1-8)"
    mkdir -p "$dest"
    cp "$f" "$dest/SKILL.md"
done
echo "✅ Все найденные SKILL.md скопированы"

# === ШАГ 3.3: Сохраняем ВСЕ API-ключи ===
mkdir -p /root/pixel-workspace/secrets-backup

# 1. ВСЕ .env файлы (содержат DEEPSEEK_API_KEY, TELEGRAM_BOT_TOKEN и др.)
find /root/.openclaw -name ".env" -not -path "*/.venv/*" -not -path "*/node_modules/*" | while read f; do
    cp "$f" "/root/pixel-workspace/secrets-backup/$(echo $f | tr '/' '_').env"
    echo "  ✅ $f"
done

# 2. Systemd EnvironmentFile= (указывает на .env — сохраняем сам факт)
grep -h "EnvironmentFile=" /etc/systemd/system/*.service /root/.config/systemd/user/*.service 2>/dev/null \
    > /root/pixel-workspace/secrets-backup/environment-files.txt

# 3. Environment= с ключами напрямую в сервисах (если есть)
grep -h "Environment=" /etc/systemd/system/*.service /root/.config/systemd/user/*.service 2>/dev/null \
    | grep -iE "KEY|TOKEN|SECRET|PASS" \
    > /root/pixel-workspace/secrets-backup/environment-vars.txt

# 4. Весь openclaw.json (там botToken и auth-профили)
cp /root/.openclaw/openclaw.json /root/pixel-workspace/secrets-backup/openclaw.json 2>/dev/null

# 5. Папка credentials (Telegram pairing, OAuth-токены)
cp -r /root/.openclaw/credentials /root/pixel-workspace/secrets-backup/credentials 2>/dev/null

echo "✅ Все ключи сохранены в /root/pixel-workspace/secrets-backup/"
ls -la /root/pixel-workspace/secrets-backup/

# === ШАГ 3.4: Найти ВСЕ боты, привязанные к OpenClaw ===
grep -rl "/root/.openclaw" /etc/systemd/system/ 2>/dev/null
grep -rl "/root/.openclaw" /root/.config/systemd/user/ 2>/dev/null

# Сохраняем список сервисов
grep -rl "/root/.openclaw" /etc/systemd/system/ /root/.config/systemd/user/ 2>/dev/null > /tmp/openclaw_services.txt
echo "✅ Сервисы сохранены в /tmp/openclaw_services.txt"
```

### 4. Обновление путей во ВСЕХ ботах

```bash
# Для каждого сервиса из списка — обновить /root/.openclaw → /root/pixel-workspace
while read service_file; do
    sed -i 's|/root/.openclaw|/root/pixel-workspace|g' "$service_file"
    echo "Обновлён: $service_file"
done < /tmp/openclaw_services.txt

# Обновить пути в скриптах внутри workspace
find /root/pixel-workspace -type f -name "*.sh" -exec sed -i 's|/root/.openclaw|/root/pixel-workspace|g' {} \;

# Перезагрузить systemd
systemctl daemon-reload
```

### 5. Перенос скиллов OpenClaw → Hermes

```bash
# Найти все скиллы в workspace
find /root/pixel-workspace -name "SKILL.md" | while read skill; do
    skill_dir=$(dirname "$skill")
    skill_name=$(basename "$skill_dir")
    cp -r "$skill_dir" "/root/.hermes/skills/$skill_name"
    echo "Skill: $skill_name"
done

# Проверить plugin-skills (если ещё не удалены)
if [ -d /root/.openclaw/plugin-skills ]; then
    cp -r /root/.openclaw/plugin-skills/* /root/.hermes/skills/
fi
```

### 6. Исправление бага Event loop в Python-ботах

```bash
# Для каждого bot.py в workspace — чиним краш Event loop is closed
find /root/pixel-workspace -name "bot.py" | while read bot; do
    python3 -c "
import sys
with open('$bot') as f:
    content = f.read()
old = '''        except Exception as e:
            logger.error(f\"💥 Бот упал: {e}\", exc_info=True)
            logger.info(\"🔄 Перезапуск через 10 секунд...\")
            # Принудительно создаём новый event loop для следующей попытки
            try:
                asyncio.set_event_loop(asyncio.new_event_loop())
            except:
                pass
            time.sleep(10)'''
new = '''        except Exception as e:
            logger.error(f\"💥 Бот упал: {e}\", exc_info=True)
            logger.info(\"🔄 Перезапуск через 10 секунд...\")
            # Корректно закрываем старый event loop и создаём новый
            try:
                loop = asyncio.get_event_loop()
                if not loop.is_closed():
                    loop.close()
            except Exception:
                pass
            try:
                new_loop = asyncio.new_event_loop()
                asyncio.set_event_loop(new_loop)
            except Exception:
                pass
            time.sleep(10)'''
if old in content:
    content = content.replace(old, new)
    with open('$bot', 'w') as f:
        f.write(content)
    print('Patched: $bot')
"
done
```

### 6. Удаление OpenClaw

```bash
# Останавливаем все сервисы OpenClaw
systemctl --user stop openclaw-gateway 2>/dev/null
systemctl --user disable openclaw-gateway 2>/dev/null
systemctl stop openclaw 2>/dev/null
systemctl disable openclaw 2>/dev/null

# Убиваем процессы
pkill -9 -f openclaw

# Удаляем npm-пакет
npm uninstall -g openclaw

# Чистим сервисные файлы
rm -f /root/.config/systemd/user/openclaw-gateway.service*
rm -f /etc/systemd/system/openclaw.service

# Удаляем данные (workspace уже в /root/pixel-workspace!)
rm -rf /root/.openclaw

# Перезагружаем и перезапускаем обновлённые боты
systemctl daemon-reload

# Рестарт всех ботов, которые были привязаны к OpenClaw
while read service_file; do
    svc=$(basename "$service_file")
    systemctl restart "$svc" 2>/dev/null
    echo "Перезапущен: $svc"
done < /tmp/openclaw_services.txt
```

### 7. Запуск Hermes gateway

```bash
export PATH="/root/.local/bin:$PATH"

# Переносим API-ключи из бэкапа в Hermes .env
echo "=== Ключи из бэкапа ==="
cat /root/pixel-workspace/secrets-backup/openclaw.json | python3 -c "
import json,sys; c=json.load(sys.stdin)
# botToken
for p,ch in c.get('channels',{}).items():
    print(f'TELEGRAM_BOT_TOKEN={ch.get(\"botToken\",\"\")}')
# gateway token
t=c.get('gateway',{}).get('auth',{}).get('token','')
if t: print(f'GATEWAY_TOKEN={t}')
" >> /root/.hermes/.env 2>/dev/null

# Копируем DEEPSEEK_API_KEY из первого найденного .env
grep "DEEPSEEK_API_KEY" /root/pixel-workspace/secrets-backup/*.env 2>/dev/null | head -1 | cut -d: -f2 >> /root/.hermes/.env

echo "✅ Ключи добавлены в ~/.hermes/.env"

hermes gateway install
hermes gateway status
```

### 8. Проверка ВСЕХ ботов

```bash
# Проверить что все сервисы запущены
while read service_file; do
    svc=$(basename "$service_file")
    systemctl status "$svc" --no-pager | head -3
done < /tmp/openclaw_services.txt

# Проверить Hermes
hermes gateway status
```

Отправь каждому боту тестовое сообщение в Telegram.

## Частые ошибки

### Боты не работают после миграции
**Причина:** сервисы указывают на `/root/.openclaw/...` — удалённую папку.
**Решение:** шаги 3–4 выше. `grep -rl "/root/.openclaw" /etc/systemd/system/` найдёт все затронутые сервисы.

### SSH виснет на KexAlgorithms
```bash
ssh -o KexAlgorithms=curve25519-sha256 root@IP
```

### «No bot token configured»
Токен в `~/.hermes/.env` должен быть: `TELEGRAM_BOT_TOKEN=123:abc`

### «Provider authentication failed»
Ключ DeepSeek неверный. **Скопировать ключ из `.env` старого бота** — не придумывать новый!

### «Telegram polling conflict»
Старая сессия OpenClaw ещё висит. Убить процессы + подождать 30-60 сек.

### OpenClaw перезапускается сам
```bash
systemctl --user list-units | grep openclaw
systemctl list-units | grep openclaw
crontab -l | grep openclaw
```
Отключить всё найденное.

## Результат

| До (OpenClaw) | После (Hermes) |
|---|---|
| Node.js, 350 MB RAM | Python, 80 MB RAM |
| 3 провайдера | 20+ провайдеров |
| Только Telegram | Telegram + Discord + Slack + ... |
| Нет skills/memory | Self-improving skills + память |
| Нет cron/webhooks | Встроенный cron + webhooks |

---

*Скилл основан на реальной миграции: сервер 95.214.62.174, Pixel Bot v3.0 + Hermes 0.18.2, июль 2026.*
