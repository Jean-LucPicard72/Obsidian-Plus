# MTProto Proxy сервер

> **Цель:** Поднять личный MTProto-прокси для Telegram на VPS/LXC, чтобы Telegram работал без VPN даже при блокировках.

---

## Что такое MTProto Proxy

MTProto — транспортный протокол Telegram. Прокси на его основе:
- встроен в официальный клиент Telegram (не нужно ставить отдельное приложение)
- выглядит как обычный HTTPS-трафик (TLS-обфускация)
- не раскрывает содержимое сообщений (E2E остаётся)
- подходит для раздачи ссылки друзьям/семье

Есть несколько реализаций. Здесь используем **[MTG](https://github.com/9seconds/mtg)** — Go-бинарник, без зависимостей, активно поддерживается.

---

## Архитектура

```
Telegram клиент (телефон/ноутбук)
         │
         │  TLS-обфусцированный MTProto
         ▼
┌─────────────────────────────┐
│   VPS / LXC (публичный IP)  │
│                             │
│   mtg  :443  ──────────────►│──► Telegram серверы
│                             │
└─────────────────────────────┘
```

---

## Требования

| Параметр | Минимум |
|----------|---------|
| ОС | Debian 12 / Ubuntu 22.04 |
| CPU | 1 vCPU |
| RAM | 128 МБ |
| Трафик | ~1–5 ГБ/мес на пользователя |
| Открытые порты | `443/tcp` (или любой другой) |

---

## Установка MTG

### 1. Скачать бинарник

```bash
# Определить архитектуру
dpkg --print-architecture
# amd64 / arm64 / arm

# Последняя версия (проверь на https://github.com/9seconds/mtg/releases)
MTG_VERSION="2.1.7"
ARCH="amd64"   # поменяй если нужно

curl -Lo /usr/local/bin/mtg \
  "https://github.com/9seconds/mtg/releases/download/v${MTG_VERSION}/mtg-linux-${ARCH}"

chmod +x /usr/local/bin/mtg
mtg --version
```

### 2. Сгенерировать секрет

```bash
mtg generate-secret --hex tls my.domain.com
```

- `tls` — включает TLS-обфускацию (рекомендуется)
- `my.domain.com` — домен, под который маскируется трафик (например `www.amazon.com`)

Вывод будет примерно такой:
```
ee3c3aface4d....  ← это твой SECRET, сохрани
tg://proxy?server=X.X.X.X&port=443&secret=ee3c3a...
```

### 3. Создать конфиг

```bash
mkdir -p /etc/mtg
cat > /etc/mtg/config.toml << 'EOF'
secret = "ee3c3aface4d...."   # вставь свой секрет

bind-to = "0.0.0.0:443"

# Опционально — ограничить число соединений
# concurrency = 8192

# Статистика (локально, не светится наружу)
[stats]
bind-to = "127.0.0.1:3129"
EOF
```

### 4. Systemd-юнит

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

# Hardening
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/log

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now mtg
systemctl status mtg
```

---

## Firewall

### Если используется `ufw`

```bash
ufw allow 443/tcp comment "MTProto Proxy"
ufw reload
```

### Если используется `nftables`

```nft
table inet filter {
    chain input {
        tcp dport 443 accept comment "MTProto"
    }
}
```

### Если LXC за Proxmox — прокинуть порт

На хосте Proxmox:

```bash
# В /etc/network/interfaces или через GUI
# DNAT 443 -> LXC IP
iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 443 -j DNAT --to-destination 192.168.100.10:443
iptables -A FORWARD -p tcp -d 192.168.100.10 --dport 443 -j ACCEPT
```

Или добавить в `/etc/pve/lxc/<CTID>.conf`:

```
lxc.hook.autodev: sh -c "iptables -t nat -A PREROUTING ..."
```

---

## Подключение в Telegram

### Вариант А — ссылка (самое простое)

MTG при запуске выводит ссылку вида:
```
tg://proxy?server=1.2.3.4&port=443&secret=ee3c3a...
```

Открой её на телефоне — Telegram сразу предложит подключиться.

Или сгенерировать ссылку вручную:
```
https://t.me/proxy?server=1.2.3.4&port=443&secret=ee3c3a...
```

### Вариант Б — вручную в настройках

`Настройки → Данные и память → Тип прокси → MTProto`

| Поле | Значение |
|------|----------|
| Сервер | IP или домен сервера |
| Порт | `443` |
| Секрет | `ee3c3a...` |

---

## Мониторинг

MTG открывает метрики на `127.0.0.1:3129`:

```bash
curl -s http://127.0.0.1:3129/stats | python3 -m json.tool
```

Пример полезных полей:
- `connections.active` — текущие соединения
- `traffic.ingress` / `traffic.egress` — трафик с начала запуска

Для Prometheus — MTG поддерживает `/metrics` endpoint (формат Prometheus).

Простой просмотр логов:
```bash
journalctl -u mtg -f
```

---

## Обновление

```bash
MTG_VERSION="X.Y.Z"
ARCH="amd64"

systemctl stop mtg

curl -Lo /usr/local/bin/mtg \
  "https://github.com/9seconds/mtg/releases/download/v${MTG_VERSION}/mtg-linux-${ARCH}"

chmod +x /usr/local/bin/mtg
systemctl start mtg
```

---

## Troubleshooting

### Telegram не подключается

```bash
# Проверь что mtg слушает порт
ss -tlnp | grep 443

# Проверь логи
journalctl -u mtg --since "5 min ago"

# Проверь доступность снаружи (с другой машины)
nc -zv <server_ip> 443
```

### Ошибка `bind: permission denied` на порту 443

```bash
# Вариант 1 — capability
setcap cap_net_bind_service=+ep /usr/local/bin/mtg

# Вариант 2 — поменять порт на 8443 в конфиге и прокинуть через iptables
iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port 8443
```

### Высокая нагрузка

По умолчанию MTG не ограничивает число соединений. Добавь в конфиг:
```toml
concurrency = 512   # макс. одновременных соединений
```

---

## Мультипользовательская схема

> **Важно:** протокол MTProto не поддерживает привязку ключа к устройству. Любой, кто получил ссылку, может ею пользоваться и переслать её дальше. Единственная защита — давать каждому пользователю **свой** секрет.

### Как это работает

Один пользователь = один секрет = один порт = один systemd-сервис.  
Тогда можно видеть трафик каждого отдельно и отозвать конкретного человека, не затрагивая остальных.

```
:443  → mtg@alice  (секрет alice)
:444  → mtg@bob    (секрет bob)
:445  → mtg@carol  (секрет carol)
```

### Шаблонный systemd-юнит (`/etc/systemd/system/mtg@.service`)

```bash
cat > /etc/systemd/system/mtg@.service << 'EOF'
[Unit]
Description=MTProto Proxy — %i
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=nobody
ExecStart=/usr/local/bin/mtg run /etc/mtg/%i.toml
Restart=always
RestartSec=5
LimitNOFILE=65536
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes

[Install]
WantedBy=multi-user.target
EOF
```

### Добавить пользователя (пример — Alice, порт 443)

```bash
# 1. Сгенерировать секрет
mtg generate-secret --hex tls www.amazon.com

# 2. Создать конфиг
cat > /etc/mtg/alice.toml << 'EOF'
secret = "ВАШ_СЕКРЕТ_ALICE"
bind-to = "0.0.0.0:443"

[stats]
bind-to = "127.0.0.1:3443"   # порт метрик = порт прокси + 3000
EOF

# 3. Включить и запустить
systemctl enable --now mtg@alice
systemctl status mtg@alice
```

Для Bob — аналогично, но порт `444`, метрики `127.0.0.1:3444`, конфиг `/etc/mtg/bob.toml`.

### Посмотреть трафик конкретного пользователя

```bash
# Alice
curl -s http://127.0.0.1:3443/stats | python3 -c "
import json,sys
s = json.load(sys.stdin)
print('Active:', s['connections']['active'])
print('In:    ', s['traffic']['ingress'] // 1024 // 1024, 'MB')
print('Out:   ', s['traffic']['egress'] // 1024 // 1024, 'MB')
"
```

### Отозвать пользователя

```bash
# Мгновенно — все его соединения рвутся, ссылка перестаёт работать
systemctl disable --now mtg@bob
```

Чтобы потом выдать Bob новый ключ — создаёшь новый секрет, обновляешь `/etc/mtg/bob.toml`, запускаешь снова.

### Firewall — открыть все пользовательские порты

```bash
# ufw
for port in 443 444 445; do
  ufw allow $port/tcp comment "MTProto"
done
ufw reload
```

---

## Безопасность

- Секрет — это одновременно и аутентификация, и ключ обфускации. **Не публикуй его.**
- Один секрет на одного пользователя — если ссылка утечёт, отзываешь только его инстанс.
- Ссылку **можно переслать**: технически ничто не мешает человеку поделиться ею. Лимит `concurrency` в конфиге защитит сервер от перегрузки, но не от "лишних" пользователей.
- Используй TLS-обфускацию (`--hex tls`) — без неё трафик легче детектировать.
- Сервисы запускаются под `nobody` — дополнительная изоляция.

---

## Связанные заметки

- [[Tailscale + Mihomo на Proxmox]]
- [[Mihomo — расширенные правила]]
