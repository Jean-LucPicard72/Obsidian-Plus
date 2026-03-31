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
              │                │               │
              ▼                ▼               ▼
        Сайт открылся   Сайт "не работает"  Через узел
```

**Ключевая идея:** Твой выходной IP — домашний российский. Для всех сайтов и приложений ты обычный пользователь. Приложения с антивпн-защитой пингуют "заблокированные" сайты — Mihomo отвечает `REJECT`, имитируя блокировку провайдера.

---

## Что понадобится

- Proxmox VE 8.x+
- LXC-шаблон Debian 12 или Ubuntu 22.04 (скачать в Proxmox)
- Аккаунт на [tailscale.com](https://tailscale.com) (бесплатный)
- (Опционально) Прокси-сервер для трафика через узел

---

## Этап 1 — LXC для Tailscale

### 1.1 Создание контейнера

В Proxmox UI:

1. **Create CT** → выбрать шаблон Debian 12
2. Параметры:
   - RAM: **128 MB** (достаточно)
   - Disk: **2 GB**
   - CPU: **1 core**
   - Network: bridge `vmbr0`, DHCP

> ⚠️ Обязательно поставить галочку **Privileged container** — без неё `/dev/tun` недоступен.

### 1.2 Настройка LXC для TUN-устройства

После создания, **до первого запуска**, отредактировать конфиг контейнера.

На хосте Proxmox (через Shell или SSH):

```bash
nano /etc/pve/lxc/<ID>.conf
```

Добавить в конец файла:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Где `<ID>` — номер твоего контейнера (например `100`).

### 1.3 Установка Tailscale

Запустить контейнер, зайти в консоль:

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

### 1.4 Запуск и авторизация

```bash
# Запустить с нужными флагами
tailscale up \
  --advertise-exit-node \
  --advertise-routes=192.168.1.0/24 \
  --accept-dns=false
```

> Заменить `192.168.1.0/24` на свою домашнюю подсеть.

Откроется ссылка — перейти в браузере и авторизовать устройство.

### 1.5 Одобрить маршруты в Tailscale Admin Console

Перейти на [login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines):

1. Найти свою ноду → **Edit route settings**
2. Включить **Use as exit node** ✓
3. Включить подсеть `192.168.1.0/24` ✓

### 1.6 Включить автозапуск

```bash
systemctl enable --now tailscaled
```

---

## Этап 2 — LXC для Mihomo

### 2.1 Создание контейнера

Аналогично первому, но этот контейнер **не нужно делать привилегированным**:

- RAM: **256 MB**
- Disk: **2 GB**
- CPU: **1 core**
- Network: bridge `vmbr0`, статический IP (например `192.168.1.11`)

> Статический IP удобен — он прописывается в правилах iptables контейнера Tailscale.

### 2.2 Установка Mihomo

```bash
apt update && apt install -y curl wget

# Скачать последний релиз Mihomo
# Актуальную версию смотреть на: https://github.com/MetaCubeX/mihomo/releases
wget https://github.com/MetaCubeX/mihomo/releases/latest/download/mihomo-linux-amd64.gz
gunzip mihomo-linux-amd64.gz
chmod +x mihomo-linux-amd64
mv mihomo-linux-amd64 /usr/local/bin/mihomo

# Проверить
mihomo -v
```

### 2.3 Базовая конфигурация Mihomo

```bash
mkdir -p /etc/mihomo
nano /etc/mihomo/config.yaml
```

Содержимое файла:

```yaml
# /etc/mihomo/config.yaml

mixed-port: 7890
tproxy-port: 7893
allow-lan: true
bind-address: "*"
mode: rule
log-level: info
ipv6: false

# DNS — критически важно для корректной работы
dns:
  enable: true
  listen: 0.0.0.0:53
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.0/15
  
  # Серверы для российских доменов
  nameserver:
    - 77.88.8.8    # Яндекс
    - 77.88.8.1    # Яндекс резервный
  
  # Серверы для остального трафика
  fallback:
    - tls://1.1.1.1
    - tls://8.8.8.8
  
  # Фильтр: российские IP идут через nameserver, остальные через fallback
  fallback-filter:
    geoip: true
    geoip-code: RU
    ipcidr:
      - 240.0.0.0/4

# Прокси-серверы (добавить свои)
proxies:
  # Пример:
  # - name: "my-proxy"
  #   type: vless
  #   server: your.server.com
  #   port: 443
  #   ...

# Группы прокси
proxy-groups:
  - name: "PROXY"
    type: select
    proxies:
      - DIRECT
      # - my-proxy

# Правила маршрутизации
rules:
  # Локальная сеть — всегда напрямую
  - IP-CIDR,192.168.0.0/16,DIRECT
  - IP-CIDR,10.0.0.0/8,DIRECT
  - IP-CIDR,172.16.0.0/12,DIRECT
  - IP-CIDR,127.0.0.0/8,DIRECT
  
  # Российские домены — напрямую
  - GEOIP,RU,DIRECT
  
  # Домены белого списка (банки, госсервисы) — напрямую
  - DOMAIN-SUFFIX,gosuslugi.ru,DIRECT
  - DOMAIN-SUFFIX,nalog.ru,DIRECT
  - DOMAIN-SUFFIX,sberbank.ru,DIRECT
  - DOMAIN-SUFFIX,tinkoff.ru,DIRECT
  - DOMAIN-SUFFIX,vk.com,DIRECT
  - DOMAIN-SUFFIX,yandex.ru,DIRECT
  - DOMAIN-SUFFIX,mail.ru,DIRECT
  
  # Домены-маркеры для детекции VPN — REJECT
  # Приложения пингуют их, чтобы проверить "а вдруг VPN?"
  # Mihomo отвечает "нет такого" — приложение успокаивается
  - DOMAIN-SUFFIX,google.com,REJECT
  - DOMAIN-SUFFIX,youtube.com,REJECT
  - DOMAIN-SUFFIX,instagram.com,REJECT
  - DOMAIN-SUFFIX,facebook.com,REJECT
  - DOMAIN-SUFFIX,twitter.com,REJECT
  - DOMAIN,connectivitycheck.gstatic.com,REJECT
  - DOMAIN,detectportal.firefox.com,REJECT
  
  # Всё остальное — через прокси (или DIRECT если прокси нет)
  - MATCH,PROXY
```

> ⚠️ Список `REJECT`-доменов нужно **подбирать под конкретное приложение**. Смотреть что оно пингует через `tcpdump` или логи Mihomo.

### 2.4 Systemd-сервис для Mihomo

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

---

## Этап 3 — Перенаправление трафика

Этот этап выполняется **в контейнере Tailscale** (LXC 1). Весь трафик с телефона, пришедший через Tailscale, перенаправляется в Mihomo.

### 3.1 Установка iptables

```bash
apt install -y iptables iptables-persistent
```

### 3.2 Правила перенаправления

```bash
# Адрес LXC с Mihomo
MIHOMO_IP=192.168.1.11
MIHOMO_PORT=7893

# Отметить трафик с tailscale0
iptables -t mangle -A PREROUTING -i tailscale0 -p tcp \
  -j TPROXY --on-ip $MIHOMO_IP --on-port $MIHOMO_PORT --tproxy-mark 1

iptables -t mangle -A PREROUTING -i tailscale0 -p udp \
  -j TPROXY --on-ip $MIHOMO_IP --on-port $MIHOMO_PORT --tproxy-mark 1

# Маршрутизация помеченного трафика
ip rule add fwmark 1 table 100
ip route add local 0.0.0.0/0 dev lo table 100

# Сохранить правила
netfilter-persistent save
```

### 3.3 Добавить маршрут в автозапуск

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

---

## Этап 4 — Настройка телефона

### Android

1. Установить **Tailscale** из Play Store
2. Войти в аккаунт
3. Нажать на свою сеть → **Use exit node** → выбрать домашнюю ноду
4. DNS настроится автоматически через Mihomo

### iOS

1. Установить **Tailscale** из App Store
2. Войти в аккаунт  
3. В настройках сети → выбрать exit node

---

## Этап 5 — Отладка

### Проверить что Tailscale работает

```bash
# В LXC Tailscale
tailscale status
tailscale ping <имя другого устройства>
```

### Проверить маршруты

```bash
# Должен быть маршрут через tailscale0
ip route show table main
```

### Посмотреть логи Mihomo

```bash
# В LXC Mihomo
journalctl -u mihomo -f
```

### Понять что пингует приложение (tcpdump)

```bash
# В LXC Tailscale, смотреть весь трафик с телефона
apt install -y tcpdump
tcpdump -i tailscale0 -n
```

Запустить приложение на телефоне → в логах видно к каким хостам оно обращается. Эти домены добавить в REJECT-список Mihomo.

### Веб-интерфейс Mihomo (Yacd)

Mihomo поддерживает веб-интерфейс для управления:

```bash
# Добавить в config.yaml
# external-controller: 0.0.0.0:9090
# secret: "yourpassword"
```

Открыть в браузере: `http://192.168.1.11:9090/ui`

---

## Частые проблемы

| Симптом | Причина | Решение |
|---|---|---|
| Tailscale не стартует | Нет `/dev/tun` | Добавить `lxc.mount.entry` в конфиг LXC |
| Трафик не идёт через Mihomo | Нет TPROXY-модуля | `modprobe xt_TPROXY` на хосте Proxmox |
| DNS не резолвится | Конфликт DNS | Проверить что Mihomo слушает `0.0.0.0:53` |
| Приложение всё равно видит VPN | Не все маркеры в REJECT | Поймать tcpdump что именно оно пингует |
| Высокий пинг | Трафик идёт лишние хопы | Проверить что RU-IP идут DIRECT |

---

## Структура файлов

```
/etc/pve/lxc/
  100.conf          ← конфиг LXC Tailscale (lxc.mount.entry и т.д.)
  101.conf          ← конфиг LXC Mihomo

/etc/mihomo/
  config.yaml       ← основная конфигурация Mihomo
  
/etc/systemd/system/
  mihomo.service    ← сервис автозапуска
```

---

## Следующие шаги

- [ ] [[Mihomo — расширенные правила]] — GeoIP базы, доменные списки, группы прокси
- [ ] [[Tailscale — MagicDNS]] — красивые имена для устройств в сети
- [ ] [[Proxmox — бэкапы LXC]] — сохранить конфигурацию контейнеров
- [ ] [[Mihomo — Yacd веб-интерфейс]] — управление через браузер

---

*Гайд актуален для Proxmox VE 9.x, Tailscale 1.6x+, Mihomo 1.18+*
