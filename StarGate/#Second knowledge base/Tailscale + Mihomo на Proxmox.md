# Tailscale + Mihomo на Proxmox

> **Цель:** Доступ к домашней сети из любой точки мира через Tailscale, умная маршрутизация трафика через Mihomo, имитация заблокированных сайтов для приложений с антивпн-защитой.

---

## Архитектура

```
Телефон/Ноутбук (где угодно)
         │
         ▼
  Tailscale mesh VPN
  (зашифрованный туннель)
         │
         ▼
┌─────────────────────────────────────────┐
│         Proxmox VE (дома, Россия)       │
│                                         │
│  ┌──────────────┐   ┌────────────────┐  │
│  │ LXC: Tailsc. │──▶│ LXC: Mihomo   │  │
│  │ Exit node    │   │ tproxy + DNS  │  │
│  │ Subnet router│   │ правила       │  │
│  └──────────────┘   └───────┬───────┘  │
│                              │          │
└──────────────────────────────┼──────────┘
                               │
              ┌────────────────┼──────────────┐
              ▼                ▼               ▼
         Белый список    Заблокированные   Остальное
         (DIRECT)        (REJECT → "нет")  (через прокси)
```

**Ключевая идея:** Твой выходной IP — домашний российский. Для всех сайтов и приложений ты обычный пользователь. Приложения с антивпн-защитой пингуют "заблокированные" сайты — Mihomo отвечает `REJECT`, имитируя блокировку провайдера.

---

## Что понадобится

- Proxmox VE 8.x+
- LXC-шаблон Debian 12 (скачать в Proxmox: Datacenter → Storage → CT Templates)
- Аккаунт на [tailscale.com](https://tailscale.com) (бесплатный)
- (Опционально) Прокси-сервер для трафика через узел

---

## Этап 1 — LXC для Tailscale

### 1.1 Создание контейнера

В Proxmox UI → **Create CT**:

| Параметр | Значение |
|---|---|
| Hostname | `Tailscale` |
| Password | придумать (логин всегда `root`) |
| Template | Debian 12 |
| Disk | 2 GB |
| CPU | 1 core |
| RAM | 128 MB (своп не нужен) |
| Network | bridge `vmbr0`, DHCP |
| Firewall | снять галочку |

> ⚠️ Обязательно поставить галочку **Privileged container** на вкладке General — без неё `/dev/tun` недоступен и Tailscale не запустится.

### 1.2 Включить Nesting и TUN до первого запуска

**Nesting** нужен для корректной работы systemd внутри LXC.

В Proxmox UI: **CT → Options → Features → Nesting ✓**

Затем на хосте Proxmox (Shell или SSH):

```bash
nano /etc/pve/lxc/<ID>.conf
```

Добавить в конец файла:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Где `<ID>` — номер контейнера (например `100`). Посмотреть: в Proxmox UI номер указан рядом с именем контейнера.

> Если контейнер уже запускался — не страшно. Останови (Stop), добавь строки, запусти снова.

### 1.3 Установка Tailscale

Запустить контейнер → Console → логин `root`, пароль из шага 1.1:

```bash
# Обновить систему
apt update && apt upgrade -y

# Установить Tailscale (официальный скрипт)
curl -fsSL https://tailscale.com/install.sh | sh

# Включить IP forwarding (нужно для subnet router)
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' >> /etc/sysctl.conf
sysctl -p
```

### 1.4 Включить автозапуск и запустить

```bash
systemctl enable --now tailscaled
```

### 1.5 Исправить предупреждение UDP GRO

```bash
apt install -y ethtool
ethtool -K eth0 rx-udp-gro-forwarding on rx-gro-list off
```

Чтобы применялось после перезагрузки — добавить в `/etc/network/interfaces` в секцию `iface eth0`:

```
post-up ethtool -K eth0 rx-udp-gro-forwarding on rx-gro-list off
```

### 1.6 Авторизация в Tailscale

```bash
tailscale up \
  --advertise-exit-node \
  --advertise-routes=192.168.1.0/24 \
  --accept-dns=false
```

> `192.168.1.0/24` — это вся твоя домашняя подсеть целиком, не IP роутера. Менять только если у тебя другой диапазон (проверить: `ip route` на хосте Proxmox).

Команда выдаст ссылку — открыть в браузере и авторизовать устройство.

### 1.7 Одобрить маршруты в Tailscale Admin Console

Перейти на [login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines):

1. Найти свою ноду → **Edit route settings**
2. Включить **Use as exit node** ✓
3. Включить подсеть `192.168.1.0/24` ✓

---

## Этап 2 — LXC для Mihomo

### 2.1 Создание контейнера

В Proxmox UI → **Create CT**:

| Параметр | Значение |
|---|---|
| Hostname | `Mihomo` |
| Password | придумать |
| Template | Debian 12 |
| Disk | 2 GB |
| CPU | 1 core |
| RAM | 256 MB |
| Network | bridge `vmbr0`, **статический IP** (например `192.168.1.11`), Gateway = IP роутера |
| Firewall | снять галочку |

> Привилегированный контейнер для Mihomo **не нужен**.
> Статический IP обязателен — он прописывается в правилах Tailscale-контейнера.
> Если не указать Gateway — не будет интернета и ничего не скачается.

Включить Nesting: **CT → Options → Features → Nesting ✓**

### 2.2 Установка Mihomo

```bash
apt update && apt install -y wget
```

Скачать бинарник. Актуальную версию смотреть на [github.com/MetaCubeX/mihomo/releases](https://github.com/MetaCubeX/mihomo/releases) — нужен файл вида `mihomo-linux-amd64-compatible-vX.X.X.gz`:

```bash
# Заменить версию на актуальную
wget https://github.com/MetaCubeX/mihomo/releases/download/v1.19.22/mihomo-linux-amd64-compatible-v1.19.22.gz
gunzip mihomo-linux-amd64-compatible-v1.19.22.gz
chmod +x mihomo-linux-amd64-compatible-v1.19.22
mv mihomo-linux-amd64-compatible-v1.19.22 /usr/local/bin/mihomo

# Проверить
mihomo -v
```

> ⚠️ Важно: использовать `/releases/download/` в URL, а не `/releases/tag/` — иначе скачается HTML-страница вместо бинарника.

### 2.3 Конфигурация Mihomo

```bash
mkdir -p /etc/mihomo
nano /etc/mihomo/config.yaml
```

Содержимое — **копировать аккуратно**, опечатки в словах `DIRECT`/`REJECT`/`PROXY` приведут к ошибке запуска:

```yaml
mixed-port: 7890
tproxy-port: 7893
allow-lan: true
bind-address: "*"
mode: rule
log-level: info
ipv6: false

dns:
  enable: true
  listen: 0.0.0.0:53
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.0/15
  nameserver:
    - 77.88.8.8
    - 77.88.8.1
  fallback:
    - tls://1.1.1.1
    - tls://8.8.8.8
  fallback-filter:
    geoip: true
    geoip-code: RU
    ipcidr:
      - 240.0.0.0/4

proxies:
  # Добавить свои прокси-серверы здесь

proxy-groups:
  - name: "PROXY"
    type: select
    proxies:
      - DIRECT

rules:
  - IP-CIDR,192.168.0.0/16,DIRECT
  - IP-CIDR,10.0.0.0/8,DIRECT
  - IP-CIDR,172.16.0.0/12,DIRECT
  - IP-CIDR,127.0.0.0/8,DIRECT
  - GEOIP,RU,DIRECT
  - DOMAIN-SUFFIX,gosuslugi.ru,DIRECT
  - DOMAIN-SUFFIX,nalog.ru,DIRECT
  - DOMAIN-SUFFIX,sberbank.ru,DIRECT
  - DOMAIN-SUFFIX,tinkoff.ru,DIRECT
  - DOMAIN-SUFFIX,vk.com,DIRECT
  - DOMAIN-SUFFIX,yandex.ru,DIRECT
  - DOMAIN-SUFFIX,mail.ru,DIRECT
  - DOMAIN-SUFFIX,google.com,REJECT
  - DOMAIN-SUFFIX,youtube.com,REJECT
  - DOMAIN-SUFFIX,instagram.com,REJECT
  - DOMAIN-SUFFIX,facebook.com,REJECT
  - DOMAIN-SUFFIX,twitter.com,REJECT
  - DOMAIN,connectivitycheck.gstatic.com,REJECT
  - DOMAIN,detectportal.firefox.com,REJECT
  - MATCH,PROXY
```

### 2.4 Systemd-сервис

```bash
nano /etc/systemd/system/mihomo.service
```

```ini
[Unit]
Description=Mihomo proxy
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/mihomo -d /etc/mihomo
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now mihomo
systemctl status mihomo
```

Если статус `active (running)` — всё хорошо. Если `failed` — смотреть ошибку:

```bash
journalctl -u mihomo -n 30 --no-pager
```

Самая частая ошибка — опечатка в config.yaml (например `DI2BRECT` вместо `DIRECT`). Исправить в файле, затем `systemctl restart mihomo`.

---

## Этап 3 — Перенаправление трафика

> ⚠️ **Все команды этого этапа выполняются в контейнере Tailscale (не Mihomo).**

Весь трафик с телефона, пришедший через Tailscale, перенаправляется в Mihomo через TPROXY.

### 3.1 Проверить IP контейнера Mihomo

Сначала убедиться в IP-адресе LXC Mihomo. В консоли Mihomo:

```bash
ip addr show eth0
```

Запомнить IP (должен совпадать с тем что указывали при создании, например `192.168.1.11`).

### 3.2 Проверить наличие TPROXY-модуля на хосте Proxmox

На **хосте Proxmox** (не в LXC):

```bash
modprobe xt_TPROXY
lsmod | grep TPROXY
```

Если пусто — модуль не загружен, TPROXY работать не будет. Добавить в автозагрузку:

```bash
echo "xt_TPROXY" >> /etc/modules
```

### 3.3 Установка iptables в контейнере Tailscale

В консоли **контейнера Tailscale**:

```bash
apt install -y iptables iptables-persistent
```

### 3.4 Правила перенаправления

```bash
# Заменить на реальный IP контейнера Mihomo
MIHOMO_IP=192.168.1.11
MIHOMO_PORT=7893

# Перенаправить трафик с tailscale0 в Mihomo через TPROXY
iptables -t mangle -A PREROUTING -i tailscale0 -p tcp \
  -j TPROXY --on-ip $MIHOMO_IP --on-port $MIHOMO_PORT --tproxy-mark 1

iptables -t mangle -A PREROUTING -i tailscale0 -p udp \
  -j TPROXY --on-ip $MIHOMO_IP --on-port $MIHOMO_PORT --tproxy-mark 1

# Маршрутизация помеченного трафика
ip rule add fwmark 1 table 100
ip route add local 0.0.0.0/0 dev lo table 100

# Сохранить правила iptables (переживут перезагрузку)
netfilter-persistent save
```

Проверить что правила применились:

```bash
iptables -t mangle -L PREROUTING -n -v
```

Должны быть две строки с `TPROXY`.

### 3.5 Добавить ip rule и ip route в автозапуск

Правила `ip rule` и `ip route` **не сохраняются** через `netfilter-persistent` — их нужно добавить отдельно:

```bash
nano /etc/rc.local
```

```bash
#!/bin/bash
ip rule add fwmark 1 table 100 2>/dev/null || true
ip route add local 0.0.0.0/0 dev lo table 100 2>/dev/null || true
exit 0
```

```bash
chmod +x /etc/rc.local
```

Проверить что файл корректный:

```bash
bash /etc/rc.local && echo "OK"
```

---

## Этап 4 — Настройка телефона

### Android

1. Установить **Tailscale** из Play Store
2. Войти в аккаунт
3. Нажать на свою сеть → **Use exit node** → выбрать домашнюю ноду
4. DNS придёт автоматически через Mihomo

### iOS

1. Установить **Tailscale** из App Store
2. Войти в аккаунт
3. В настройках сети → выбрать exit node

---

## Этап 5 — Проверка и отладка

### Проверить Tailscale

```bash
# В LXC Tailscale
tailscale status
```

Должны быть видны все устройства сети, включая телефон.

### Проверить что трафик идёт через Mihomo

```bash
# В LXC Tailscale — смотреть трафик с телефона
apt install -y tcpdump
tcpdump -i tailscale0 -n
```

Открыть любой сайт на телефоне — в выводе должны быть пакеты.

### Посмотреть логи Mihomo

```bash
# В LXC Mihomo
journalctl -u mihomo -f
```

### Найти маркерные домены конкретного приложения

```bash
# В LXC Tailscale — фильтровать DNS-запросы
tcpdump -i tailscale0 -n port 53
```

Открыть банковское приложение → посмотреть к каким доменам оно обращается в первые секунды. Добавить их в REJECT-список `/etc/mihomo/config.yaml`, затем `systemctl restart mihomo`.

### Веб-интерфейс Mihomo (Yacd)

Добавить в `/etc/mihomo/config.yaml`:

```yaml
external-controller: 0.0.0.0:9090
secret: "yourpassword"
```

Открыть в браузере: `http://192.168.1.11:9090/ui`

---

## Частые проблемы

| Симптом | Причина | Решение |
|---|---|---|
| Tailscale не стартует | Нет `/dev/tun` | Добавить `lxc.mount.entry` в конфиг LXC, перезапустить |
| Mihomo не запускается | Опечатка в config.yaml | `journalctl -u mihomo -n 20` — покажет строку с ошибкой |
| Нет интернета в LXC | Не указан Gateway | CT → Network → добавить Gateway (IP роутера) |
| Трафик не идёт через Mihomo | Нет модуля xt_TPROXY | `modprobe xt_TPROXY` на хосте Proxmox |
| DNS не резолвится | Конфликт DNS | Проверить что Mihomo слушает `0.0.0.0:53` |
| Приложение всё равно видит VPN | Не все маркеры в REJECT | tcpdump что именно оно пингует, добавить в REJECT |
| Ошибка `command not found` после скачивания | Нет прав на выполнение | `chmod +x /usr/local/bin/mihomo` |

---

## Структура файлов

```
/etc/pve/lxc/
  <ID>.conf         ← конфиг LXC Tailscale (lxc.mount.entry и т.д.)

/etc/mihomo/
  config.yaml       ← конфигурация Mihomo

/etc/systemd/system/
  mihomo.service    ← сервис автозапуска

/etc/rc.local       ← автозапуск ip rule / ip route (в LXC Tailscale)
```

---

## Следующие шаги

- [ ] [[Mihomo — расширенные правила]] — GeoIP базы, доменные списки, два профиля
- [ ] [[Tailscale — MagicDNS]] — красивые имена для устройств в сети
- [ ] [[Proxmox — бэкапы LXC]] — сохранить конфигурацию контейнеров

---

*Актуально для Proxmox VE 9.x, Tailscale 1.6x+, Mihomo 1.19+*
