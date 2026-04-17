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
nano /etc/ssh/sshd_config
# Найти и поменять: PasswordAuthentication no
```

Перезагрузить SSH:
```bash
# Debian
systemctl reload ssh

# Ubuntu
systemctl reload sshd
```

> На Debian сервис называется `ssh`, не `sshd` — не перепутай.

---

## Этап 1 — Установка MTG

### 1.1 Определить архитектуру сервера

```bash
dpkg --print-architecture
# Ответ: amd64, arm64 или armhf
```

### 1.2 Узнать актуальную версию

```bash
curl -s https://api.github.com/repos/9seconds/mtg/releases/latest | grep tag_name
# Пример: "tag_name": "v2.2.8"
```

### 1.3 Скачать и распаковать бинарник

```bash
# Задать переменные (подставь свою версию и архитектуру)
MTG_VERSION="2.2.8"
ARCH="amd64"

cd /tmp
curl -fsSL -Lo mtg.tar.gz \
  "https://github.com/9seconds/mtg/releases/download/v${MTG_VERSION}/mtg-${MTG_VERSION}-linux-${ARCH}.tar.gz"

tar -xzf mtg.tar.gz
sudo mv mtg-${MTG_VERSION}-linux-${ARCH}/mtg /usr/local/bin/mtg
sudo chmod +x /usr/local/bin/mtg

mtg --version
```

> Начиная с v2.2.x бинарник упакован в `.tar.gz` в подпапке — прямого файла больше нет.

---

## Этап 2 — Генерация секрета

```bash
mtg generate-secret -x www.yandex.ru
```

- `-x` — вывести в hex-формате (нужен для конфига)
- `www.yandex.ru` — домен, под который маскируется TLS-рукопожатие. Можно любой крупный HTTPS-сайт

Пример вывода:
```
ee3c3aface4d9e0000000000000000002e79616e6465782e7275
```

**Сохрани секрет** — он понадобится в конфиге и для раздачи ссылок.

> Синтаксис изменился в v2.2.x: домен теперь просто аргумент, флага `tls` больше нет.

---

## Этап 3 — Конфигурация

### 3.1 Создать файл конфига

```bash
mkdir -p /etc/mtg

cat > /etc/mtg/config.toml << 'EOF'
# Секрет из шага 2
secret = "ee3c3aface4d9e0000000000000000002e79616e6465782e7275"

# Слушать на всех интерфейсах (0.0.0.0 = на любом IP этой машины)
bind-to = "0.0.0.0:443"

# Локальная статистика
[stats]
bind-to = "127.0.0.1:3129"
EOF
```

### 3.2 Проверить конфиг вручную

```bash
mtg run /etc/mtg/config.toml
# Моргающий курсор = работает. Ctrl+C для остановки.
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

# Права на порт 443 без root
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE

# Изоляция процесса
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes

[Install]
WantedBy=multi-user.target
EOF
```

> `AmbientCapabilities` — обязательно при `User=nobody`. Без него `setcap` на бинарнике не поможет.

### 4.2 Запустить и включить автозапуск

```bash
systemctl daemon-reload
systemctl enable --now mtg
systemctl status mtg
```

Ожидаемый статус: `active (running)`

---

## Этап 5 — Firewall

### UFW (Ubuntu по умолчанию)

```bash
ufw allow 443/tcp comment "MTProto Proxy"
ufw reload
ufw status
```

### nftables (Debian по умолчанию)

```bash
nft list ruleset
nft add rule inet filter input tcp dport 443 accept comment \"MTProto\"
nft list ruleset > /etc/nftables.conf
systemctl restart nftables
```

---

## Этап 6 — Подключение в Telegram

### Узнать IP сервера

```bash
curl -s ifconfig.me
```

### Ссылка для подключения

```
https://t.me/proxy?server=<IP_СЕРВЕРА>&port=443&secret=<ТВОЙ_СЕКРЕТ>
```

Открой на телефоне → Telegram предложит «Подключиться».

### Вручную в настройках

`Настройки → Конфиденциальность и безопасность → Тип прокси → MTProto`

| Поле | Значение |
|------|----------|
| Сервер | IP-адрес VPS |
| Порт | `443` |
| Секрет | Из шага 2 |

---

## Управление ключами — отдельный секрет на каждого

Каждый пользователь получает свой секрет. Разошёлся по городу — удаляешь только его, остальные не замечают.

### Добавить пользователя

```bash
mtg generate-secret -x www.yandex.ru
# Сохрани вывод — это секрет для конкретного человека
```

В конфиге перечисляешь все секреты массивом:

```toml
# /etc/mtg/config.toml
secret = [
  "ee3c3a...",   # Аня
  "ff4d4b...",   # Петя
  "aa1b2c...",   # Вася
]

bind-to = "0.0.0.0:443"

[stats]
bind-to = "127.0.0.1:3129"
```

### Отозвать пользователя

Удалить его строку из `secret = [...]` и перезапустить:

```bash
nano /etc/mtg/config.toml
systemctl restart mtg
```

Его ссылка перестанет работать. Остальные продолжают работать без изменений.

---

## Проверка работы

```bash
# Порт слушается?
ss -tlnp | grep 443

# Логи
journalctl -u mtg -f

# Статистика соединений
curl -s http://127.0.0.1:3129/stats | python3 -m json.tool
```

---

## Обновление MTG

```bash
MTG_VERSION="X.Y.Z"
ARCH="amd64"

systemctl stop mtg

cd /tmp
curl -fsSL -Lo mtg.tar.gz \
  "https://github.com/9seconds/mtg/releases/download/v${MTG_VERSION}/mtg-${MTG_VERSION}-linux-${ARCH}.tar.gz"
tar -xzf mtg.tar.gz
sudo mv mtg-${MTG_VERSION}-linux-${ARCH}/mtg /usr/local/bin/mtg
sudo chmod +x /usr/local/bin/mtg

systemctl start mtg
mtg --version
```

---

## Решение частых проблем

### `permission denied` на порту 443

Убедись что в `/etc/systemd/system/mtg.service` есть:
```ini
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
```
Затем:
```bash
systemctl daemon-reload && systemctl restart mtg
```

### `systemctl reload sshd` → Unit not found

На Debian сервис называется `ssh`:
```bash
systemctl reload ssh
```

### Curl скачал 9 байт вместо бинарника

Значит URL неверный. Проверь точное имя файла в релизе:
```bash
curl -s https://api.github.com/repos/9seconds/mtg/releases/latest | grep browser_download_url
```

### Telegram не подключается

```bash
# Порт слушается?
ss -tlnp | grep 443

# Логи
journalctl -u mtg --since "5 min ago" --no-pager

# Доступность снаружи (с другой машины)
nc -zv <IP> 443
```

---

## Связанные заметки

- [[MTProto Proxy сервер]] — справочник по конфигурации и мониторингу
- [[Tailscale + Mihomo на Proxmox]] — более сложная схема с умной маршрутизацией
- [[Mihomo — расширенные правила]]
