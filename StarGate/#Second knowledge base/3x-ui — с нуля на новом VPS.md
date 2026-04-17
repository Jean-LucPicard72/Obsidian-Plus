# 3x-ui — с нуля на новом VPS

> **Цель:** Рабочая панель 3x-ui с VLESS/VMess/Trojan за ~20 минут.  
> Гайд учитывает совместную работу с **MTProto Proxy** (mtg) на том же сервере — порты и firewall настроены так, чтобы они не конфликтовали.

---

## Что нужно до начала

| Требование | Минимум |
|---|---|
| VPS / облако | Чистый Debian 12 или Ubuntu 22.04/24.04 |
| CPU / RAM | 1 vCPU / 512 МБ (с MTProto — 1 ГБ рекомендуется) |
| Открытые порты | `2053/tcp` — панель управления; `443/tcp` — если MTProto занял, освободить или взять другой |
| Доступ | Root или sudo-пользователь по SSH |
| Домен (опционально) | Нужен только для TLS на инбаунде. Без него — self-signed или Reality |

---

## Как 3x-ui живёт рядом с MTProto

MTG (MTProto) занимает порт **443/tcp**.  
3x-ui работает на других портах — конфликта нет, если соблюдать:

| Сервис | Порт по умолчанию | Что делать |
|---|---|---|
| mtg (MTProto) | `443/tcp` | Оставить как есть |
| 3x-ui **панель** | `2053/tcp` | Открыть в firewall |
| 3x-ui **inbound** (VLESS и др.) | На твой выбор, например `8443/tcp` | Открыть в firewall |

> Не назначай inbound'ам порт `443` — он занят MTProto.

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

### 0.3 SSH-ключи (рекомендуется)

На **локальной машине**:
```bash
ssh-keygen -t ed25519 -C "vps-3xui"
ssh-copy-id deploy@<IP_СЕРВЕРА>
```

После проверки, что вход по ключу работает:
```bash
nano /etc/ssh/sshd_config
# Поменять: PasswordAuthentication no
```

```bash
# Debian
systemctl reload ssh

# Ubuntu
systemctl reload sshd
```

---

## Этап 1 — Установка 3x-ui

### 1.1 Официальный установщик

```bash
bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
```

Установщик скачает бинарник, создаст systemd-сервис и запустит панель.  
В конце выведет порт панели и временные учётные данные — **сохрани их**.

Пример вывода:
```
Panel running at: http://<IP>:2053/
Username: admin
Password: admin
```

> Установщик сам устанавливает зависимости и определяет архитектуру.

### 1.2 Проверить статус

```bash
systemctl status x-ui
# Ожидаемо: active (running)
```

---

## Этап 2 — Первый вход и смена пароля

Открой браузер: `http://<IP_СЕРВЕРА>:2053/`

**Войди** с учётными данными из шага 1.1 (по умолчанию `admin` / `admin`).

### Сразу же смени пароль

`Настройки панели (Panel Settings) → Account → Изменить логин и пароль`

Также рекомендуется:
- Поменять порт панели (например, с `2053` на произвольный 5-значный)
- Поменять путь к панели (URL-prefix, например `/mypanel/`)
- Включить HTTPS на панели (если есть домен)

После изменения настроек — **Restart Panel** в интерфейсе.

---

## Этап 3 — Настройка SSL-сертификата (опционально, но рекомендуется)

Без домена можно пропустить этот этап и использовать VLESS+Reality (см. этап 4).

### 3.1 Установить Certbot

```bash
apt install certbot -y
```

### 3.2 Выпустить сертификат (standalone — панель должна быть остановлена на момент выпуска)

```bash
systemctl stop x-ui

certbot certonly --standalone -d <ТВО_ДОМЕН>

systemctl start x-ui
```

Сертификат будет в `/etc/letsencrypt/live/<ДОМЕН>/`:
- `fullchain.pem` — публичный сертификат
- `privkey.pem` — приватный ключ

### 3.3 Указать сертификат в панели

`Настройки панели → SSL Certificate`

| Поле | Значение |
|------|----------|
| Public Key File Path | `/etc/letsencrypt/live/<ДОМЕН>/fullchain.pem` |
| Private Key File Path | `/etc/letsencrypt/live/<ДОМЕН>/privkey.pem` |

Нажать **Save** → **Restart Panel**.  
Теперь панель доступна по `https://<ДОМЕН>:2053/`.

---

## Этап 4 — Создание инбаунда (VLESS + Reality — без домена)

**VLESS + XTLS-Reality** — лучший вариант, если домена нет. Трафик выглядит как обычный TLS к популярному сайту.

### 4.1 Добавить инбаунд

В панели: `Inbounds → +Add Inbound`

| Параметр | Значение |
|----------|----------|
| Remark | `vless-reality` |
| Protocol | `VLESS` |
| Port | `8443` (или любой свободный, не 443) |
| Transmission | `TCP` |
| Security | `Reality` |
| uTLS | `chrome` |
| Dest (SNI) | `www.google.com` (или любой крупный HTTPS-сайт) |
| Server Names | `www.google.com` |

Нажать **Generate** рядом с Public/Private Key — сгенерирует пару ключей Reality.  
Нажать **Add**.

### 4.2 Добавить клиента

В строке инбаунда нажать иконку **+** (Add Client).

| Параметр | Значение |
|----------|----------|
| Email | Имя пользователя (для идентификации) |
| Flow | `xtls-rprx-vision` |
| Expiry Date | Опционально — дата истечения доступа |

Нажать **Add** → **Save**.

---

## Этап 5 — Создание инбаунда (VMess или VLESS + TLS — если есть домен)

### 5.1 Добавить инбаунд

`Inbounds → +Add Inbound`

| Параметр | Значение |
|----------|----------|
| Protocol | `VLESS` или `VMess` |
| Port | `8443` |
| Transmission | `WebSocket` (ws) |
| Path | `/vless` (любой путь) |
| Security | `TLS` |
| Public Key | `/etc/letsencrypt/live/<ДОМЕН>/fullchain.pem` |
| Private Key | `/etc/letsencrypt/live/<ДОМЕН>/privkey.pem` |

---

## Этап 6 — Firewall

### UFW (Ubuntu)

```bash
# Порт панели (поменяй 2053 если менял)
ufw allow 2053/tcp comment "3x-ui panel"

# Порт инбаунда
ufw allow 8443/tcp comment "3x-ui inbound"

# MTProto (если ещё не добавлен)
ufw allow 443/tcp comment "MTProto"

ufw reload
ufw status
```

### nftables (Debian)

```bash
# Добавить правила
nft add rule inet filter input tcp dport 2053 accept comment \"3x-ui panel\"
nft add rule inet filter input tcp dport 8443 accept comment \"3x-ui inbound\"

# Сохранить
nft list ruleset > /etc/nftables.conf
systemctl restart nftables
```

---

## Этап 7 — Получение ссылки для клиента

В панели напротив нужного клиента нажать иконку **QR** или **Copy Link**.

Ссылка выглядит так (VLESS+Reality):
```
vless://UUID@IP:8443?encryption=none&flow=xtls-rprx-vision&security=reality&sni=www.google.com&fp=chrome&pbk=PUBLIC_KEY&sid=SHORT_ID&type=tcp#vless-reality
```

Импортируй в клиент: **v2rayNG** (Android), **Streisand** / **Hiddify** (iOS), **Nekobox** / **v2rayN** (Windows/Linux).

---

## Управление пользователями

### Добавить пользователя

В панели → инбаунд → `+` (Add Client) → заполнить Email → **Save**.

### Отозвать пользователя

Инбаунд → список клиентов → иконка удаления напротив пользователя → **Save**.

### Ограничить трафик или время

При создании/редактировании клиента:
- `Total Flow (GB)` — лимит трафика (0 = без лимита)
- `Expiry Time` — дата истечения

---

## Проверка работы

```bash
# Сервис запущен?
systemctl status x-ui

# Порты слушаются?
ss -tlnp | grep -E "2053|8443"

# Логи в реальном времени
journalctl -u x-ui -f

# Версия
x-ui version
```

---

## Обновление 3x-ui

```bash
# Запустить установщик повторно — он обновит до последней версии
bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
```

Конфиг и база данных при этом сохраняются.

---

## Резервное копирование

База данных с настройками находится здесь:
```
/etc/x-ui/x-ui.db
```

Скопировать на локальную машину:
```bash
scp deploy@<IP_СЕРВЕРА>:/etc/x-ui/x-ui.db ./x-ui-backup.db
```

Восстановить: загрузить через `Panel Settings → Database → Import DB`.

---

## Решение частых проблем

### Панель недоступна после смены порта

```bash
# Убедиться что новый порт открыт в firewall
ufw allow <НОВЫЙ_ПОРТ>/tcp

# Проверить что сервис слушает на новом порту
ss -tlnp | grep x-ui
```

### Пароль от панели утерян — сброс

```bash
x-ui
# → пункт "Reset username and password"
```

Или напрямую:
```bash
x-ui setting -username admin -password newpassword
systemctl restart x-ui
```

### Клиент подключается, но нет интернета

Проверь IP-forwarding:
```bash
cat /proc/sys/net/ipv4/ip_forward
# Должно быть: 1
```

Если `0`:
```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

### Конфликт с MTProto на порту 443

3x-ui inbound по умолчанию не использует 443 — но если ты случайно назначил его:
```bash
# Посмотреть что занимает порт
ss -tlnp | grep 443

# Поменять порт inbound'а в панели на свободный (например 8443)
# Затем обновить firewall-правило
```

### `x-ui` команда не найдена

```bash
# Проверить установку
which x-ui
ls /usr/local/x-ui/

# Переустановить
bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
```

---

## Связанные заметки

- [[MTProto Proxy — с нуля на новом VPS]] — параллельный сервис на том же VPS
- [[MTProto Proxy сервер]] — справочник по конфигурации MTG
- [[Tailscale + Mihomo на Proxmox]] — более сложная схема маршрутизации
