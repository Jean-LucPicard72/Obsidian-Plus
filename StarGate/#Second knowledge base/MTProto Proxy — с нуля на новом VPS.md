# MTProto Proxy — с нуля на новом VPS

> **Цель:** Получить рабочий MTProto-прокси для Telegram за ~15 минут.  
> Используем **[MTG v2](https://github.com/9seconds/mtg)** — лёгкий Go-бинарник без зависимостей.

---

## Что нужно до начала

| Требование | Минимум |
|---|---|
| VPS / облако | Любой провайдер, чистый Debian 12 или Ubuntu 22.04/24.04 |
| CPU / RAM | 1 vCPU / 128 МБ |
| Открытый порт | `443/tcp` (или любой другой, 443 предпочтительнее — выглядит как HTTPS) |
| Доступ | Root или sudo-пользователь по SSH |

---

## Этап 0 — Первое подключение и минимальная безопасность

```bash
# Подключиться к серверу
ssh root@<IP_СЕРВЕРА>
```

### 0.1 Обновить систему

```bash
apt update && apt upgrade -y
```

### 0.2 Создать обычного пользователя (если зашли как root)

```bash
adduser deploy
usermod -aG sudo deploy
```

### 0.3 Настроить SSH-ключи (рекомендуется)

На **локальной машине** (если ключа ещё нет):
```bash
ssh-keygen -t ed25519 -C "mtproto-vps"
```

Скопировать ключ на сервер:
```bash
ssh-copy-id deploy@<IP_СЕРВЕРА>
```

После проверки что вход по ключу работает — отключить вход по паролю:
```bash
# /etc/ssh/sshd_config
PasswordAuthentication no

systemctl reload sshd
```

---

## Этап 1 — Установка MTG

### 1.1 Определить архитектуру сервера

```bash
dpkg --print-architecture
# Ответ: amd64, arm64 или armhf
```

### 1.2 Скачать бинарник

```bash
MTG_VERSION="2.1.7"
ARCH="amd64"   # ← замени если arm64 / armhf

curl -Lo /usr/local/bin/mtg \
  "https://github.com/9seconds/mtg/releases/download/v${MTG_VERSION}/mtg-linux-${ARCH}"

chmod +x /usr/local/bin/mtg
mtg --version
```

Ожидаемый вывод:
```
mtg/2.1.7 ...
```

> Актуальную версию смотри на https://github.com/9seconds/mtg/releases

### 1.3 Дать права на порт 443 без root

```bash
setcap cap_net_bind_service=+ep /usr/local/bin/mtg
```

---

## Этап 2 — Генерация секрета

```bash
mtg generate-secret --hex tls www.amazon.com
```

- `tls` — включает TLS-обфускацию: трафик замаскирован под HTTPS
- `www.amazon.com` — домен, под который маскируется рукопожатие. Можно любой крупный HTTPS-сайт (`www.google.com`, `cloudflare.com`)

Пример вывода:
```
ee3c3aface4d9e0000000000000000002e616d617a6f6e2e636f6d
tg://proxy?server=<IP>&port=443&secret=ee3c3a...
```

**Сохрани секрет** — он понадобится в конфиге и для раздачи ссылок.

---

## Этап 3 — Конфигурация

### 3.1 Создать файл конфига

```bash
mkdir -p /etc/mtg

cat > /etc/mtg/config.toml << 'EOF'
# Твой секрет из шага 2
secret = "ee3c3aface4d9e0000000000000000002e616d617a6f6e2e636f6d"

# Адрес и порт прослушивания
bind-to = "0.0.0.0:443"

# Максимум одновременных соединений (опционально, 0 = без ограничений)
# concurrency = 4096

# Локальная статистика
[stats]
bind-to = "127.0.0.1:3129"
EOF
```

### 3.2 Проверить конфиг

```bash
mtg run /etc/mtg/config.toml
# Должно запуститься без ошибок, Ctrl+C для остановки
```

---

## Этап 4 — Systemd-сервис

### 4.1 Создать юнит

```bash
cat > /etc/systemd/system/mtg.service << 'EOF'
[Unit]
Description=MTProto Proxy (mtg)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=nobody
ExecStart=/usr/local/bin/mtg run /etc/mtg/config.toml
Restart=always
RestartSec=5
LimitNOFILE=65536

# Изоляция процесса
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes

[Install]
WantedBy=multi-user.target
EOF
```

### 4.2 Запустить и включить автозапуск

```bash
systemctl daemon-reload
systemctl enable --now mtg
systemctl status mtg
```

Ожидаемый статус: `active (running)`

---

## Этап 5 — Firewall

### Если используется UFW (Ubuntu по умолчанию)

```bash
ufw allow 443/tcp comment "MTProto Proxy"
ufw reload
ufw status
```

### Если используется nftables (Debian по умолчанию)

```bash
# Проверить текущие правила
nft list ruleset

# Добавить разрешение для порта 443
nft add rule inet filter input tcp dport 443 accept comment \"MTProto\"

# Сохранить (Debian 12)
nft list ruleset > /etc/nftables.conf
systemctl restart nftables
```

### Если firewall не настроен

```bash
# Быстро проверить что порт слушается
ss -tlnp | grep 443
```

---

## Этап 6 — Подключение в Telegram

### Способ 1 — Ссылка (быстрее всего)

Сформируй ссылку:
```
https://t.me/proxy?server=<IP_СЕРВЕРА>&port=443&secret=<ТВОЙ_СЕКРЕТ>
```

Открой на телефоне → Telegram предложит «Подключиться».

Можно также использовать схему `tg://`:
```
tg://proxy?server=<IP_СЕРВЕРА>&port=443&secret=<ТВОЙ_СЕКРЕТ>
```

### Способ 2 — Вручную в настройках

`Настройки → Конфиденциальность и безопасность → Тип прокси → MTProto`

| Поле | Значение |
|------|----------|
| Сервер | IP-адрес VPS |
| Порт | `443` |
| Секрет | Из шага 2 |

---

## Этап 7 — Проверка работы

### Проверка порта с локальной машины

```bash
nc -zv <IP_СЕРВЕРА> 443
# Ожидаемо: Connection to ... succeeded
```

### Просмотр логов

```bash
journalctl -u mtg -f
```

### Статистика соединений

```bash
curl -s http://127.0.0.1:3129/stats | python3 -m json.tool
```

Полезные поля:
- `connections.active` — активные соединения прямо сейчас
- `traffic.ingress` / `traffic.egress` — суммарный трафик с запуска

---

## Несколько секретов для разных пользователей

Каждый пользователь может получить свой секрет — тогда их можно отзывать независимо:

```bash
# Генерируем отдельный секрет для каждого
mtg generate-secret --hex tls www.amazon.com  # пользователь 1
mtg generate-secret --hex tls www.amazon.com  # пользователь 2
```

В конфиге перечисляем все секреты через массив:

```toml
# /etc/mtg/config.toml
secret = [
  "ee3c3a...",   # Аня
  "ff4d4b...",   # Петя
]

bind-to = "0.0.0.0:443"

[stats]
bind-to = "127.0.0.1:3129"
```

Чтобы отозвать пользователя — убираем его секрет и перезапускаем:
```bash
systemctl restart mtg
```

---

## Обновление MTG

```bash
MTG_VERSION="X.Y.Z"   # новая версия
ARCH="amd64"

systemctl stop mtg

curl -Lo /usr/local/bin/mtg \
  "https://github.com/9seconds/mtg/releases/download/v${MTG_VERSION}/mtg-linux-${ARCH}"

chmod +x /usr/local/bin/mtg
setcap cap_net_bind_service=+ep /usr/local/bin/mtg

systemctl start mtg
mtg --version
```

---

## Решение частых проблем

### Telegram не подключается

```bash
# 1. Порт слушается?
ss -tlnp | grep 443

# 2. Логи MTG
journalctl -u mtg --since "5 min ago"

# 3. Доступность снаружи (выполнять НЕ на сервере)
nc -zv <IP> 443

# 4. Firewall не режет?
iptables -L -n | grep 443
```

### `bind: permission denied` на порту 443

```bash
# Вариант 1 — capability (уже делали в шаге 1.3)
setcap cap_net_bind_service=+ep /usr/local/bin/mtg

# Вариант 2 — сменить порт на 8443 и проксировать через iptables
# В конфиге: bind-to = "0.0.0.0:8443"
iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port 8443
```

### `failed to start` — ошибка в конфиге

```bash
# Проверить TOML вручную
mtg run /etc/mtg/config.toml
# Покажет подробную ошибку в stdout
```

### Высокое потребление CPU/памяти

Добавить ограничение в конфиг:
```toml
concurrency = 512
```

Перезапустить:
```bash
systemctl restart mtg
```

---

## Безопасность — кратко

- **Не публикуй секрет** — он и аутентифицирует, и шифрует обфускацию
- **Используй TLS-обфускацию** (`--hex tls`) — без неё трафик проще детектировать по сигнатуре
- Сервис запускается под `nobody` — минимальные привилегии
- Регулярно проверяй новые версии MTG на GitHub

---

## Связанные заметки

- [[MTProto Proxy сервер]] — справочник по конфигурации и мониторингу
- [[Tailscale + Mihomo на Proxmox]] — более сложная схема с умной маршрутизацией
- [[Mihomo — расширенные правила]]
