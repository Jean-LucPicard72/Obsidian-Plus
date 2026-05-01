# Tailscale + Mihomo на Proxmox

> **Цель:** Доступ к домашней сети из любой точки мира через Tailscale, умная маршрутизация трафика через Mihomo — российские сайты напрямую, заблокированные через внешний прокси.

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
│  ┌──────────────────────────────────┐   │
│  │ LXC: Tailscale + Mihomo          │   │
│  │ Exit node + Subnet router        │   │
│  │ TPROXY → Mihomo (локально)       │   │
│  └──────────────────────────────────┘   │
│                                         │
└─────────────────────────────────────────┘
         │
         ├── GEOIP,RU → DIRECT (домашний IP)
         ├── GEOSITE,ru-blocked → vdsina (VLESS Reality)
         └── MATCH → vdsina
```

**Ключевая идея:** Mihomo работает в том же LXC что и Tailscale. TPROXY перехватывает трафик с tailscale0 и отдаёт локальному Mihomo. Российские сайты идут напрямую через домашний IP, заблокированные — через внешний прокси.

---

## Что понадобится

- Proxmox VE 8.x+
- LXC-шаблон Debian 12 (скачать в Proxmox: Datacenter → Storage → CT Templates)
- Аккаунт на [tailscale.com](https://tailscale.com) (бесплатный)
- Внешний прокси-сервер (VLESS Reality, Shadowsocks и др.)

---

## Этап 1 — LXC для Tailscale + Mihomo

### 1.1 Создание контейнера

В Proxmox UI → **Create CT**:

| Параметр | Значение |
|---|---|
| Hostname | `Tailscale` |
| Password | придумать (логин всегда `root`) |
| Template | Debian 12 |
| Disk | 4 GB |
| CPU | 1 core |
| RAM | 256 MB |
| Network | bridge `vmbr0`, DHCP |
| Firewall | снять галочку |

> ⚠️ Обязательно поставить галочку **Privileged container** на вкладке General.

### 1.2 Включить Nesting и TUN

**Nesting** нужен для корректной работы systemd внутри LXC.

В Proxmox UI: **CT → Options → Features → Nesting ✓**

На хосте Proxmox:

```bash
nano /etc/pve/lxc/<ID>.conf
```

Добавить в конец:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

> Если контейнер уже запускался — останови, добавь строки, запусти снова.

### 1.3 Проверить модуль TPROXY на хосте Proxmox

На **хосте Proxmox**:

```bash
modprobe xt_TPROXY
echo "xt_TPROXY" >> /etc/modules
```

---

## Этап 2 — Установка Tailscale

Запустить контейнер → Console → логин `root`:

```bash
apt update && apt upgrade -y

curl -fsSL https://tailscale.com/install.sh | sh

echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' >> /etc/sysctl.conf
sysctl -p

systemctl enable --now tailscaled
```

### Исправить предупреждение UDP GRO

```bash
apt install -y ethtool
ethtool -K eth0 rx-udp-gro-forwarding on rx-gro-list off
```

Добавить в `/etc/network/interfaces` в секцию `iface eth0`:

```
post-up ethtool -K eth0 rx-udp-gro-forwarding on rx-gro-list off
```

### Авторизация

```bash
tailscale up \
  --advertise-exit-node \
  --advertise-routes=192.168.1.0/24 \
  --accept-dns=false
```

Открыть ссылку в браузере → авторизовать устройство.

> `192.168.1.0/24` — вся домашняя подсеть. Менять только если у тебя другой диапазон.

### Одобрить маршруты в Tailscale Admin Console

[login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines):

1. Найти ноду → **Edit route settings**
2. **Use as exit node** ✓
3. Подсеть `192.168.1.0/24` ✓

> ⚠️ Если Tailscale не может авторизоваться — остановить Mihomo (`systemctl stop mihomo`), авторизоваться, затем снова запустить.

---

## Этап 3 — Установка Mihomo

```bash
apt install -y wget
```

Актуальную версию смотреть на [github.com/MetaCubeX/mihomo/releases](https://github.com/MetaCubeX/mihomo/releases):

```bash
wget https://github.com/MetaCubeX/mihomo/releases/download/v1.19.22/mihomo-linux-amd64-compatible-v1.19.22.gz
gunzip mihomo-linux-amd64-compatible-v1.19.22.gz
chmod +x mihomo-linux-amd64-compatible-v1.19.22
mv mihomo-linux-amd64-compatible-v1.19.22 /usr/local/bin/mihomo
mihomo -v
```

> ⚠️ URL: `/releases/download/` — не `/releases/tag/`, иначе скачается HTML.

### Скачать базы GeoSite и GeoIP

Используем [runetfreedom/russia-v2ray-rules-dat](https://github.com/runetfreedom/russia-v2ray-rules-dat) — обновляется каждые 6 часов, содержит актуальные списки РКН:

```bash
mkdir -p /etc/mihomo

wget -O /etc/mihomo/geosite.dat \
  https://github.com/runetfreedom/russia-v2ray-rules-dat/releases/latest/download/geosite.dat

wget -O /etc/mihomo/geoip.dat \
  https://github.com/runetfreedom/russia-v2ray-rules-dat/releases/latest/download/geoip.dat
```

### Конфигурация Mihomo

```bash
nano /etc/mihomo/config.yaml
```

```yaml
mixed-port: 7890
tproxy-port: 7893
allow-lan: true
bind-address: "*"
mode: rule
log-level: info
ipv6: false

external-controller: 0.0.0.0:9090
secret: "yourpassword"
external-ui: /etc/mihomo/ui

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
  - name: "vdsina"
    type: vless
    server: your.server.com      # адрес сервера
    port: 443
    uuid: your-uuid-here
    network: xhttp
    tls: true
    udp: true
    reality-opts:
      public-key: your-public-key
      short-id: "your-short-id"
    client-fingerprint: firefox
    servername: github.com
    xhttp-opts:
      path: /
      mode: auto

proxy-groups:
  - name: "PROXY"
    type: select
    proxies:
      - vdsina
      - DIRECT

rules:
  - IP-CIDR,192.168.0.0/16,DIRECT
  - IP-CIDR,10.0.0.0/8,DIRECT
  - IP-CIDR,172.16.0.0/12,DIRECT
  - IP-CIDR,127.0.0.0/8,DIRECT
  - GEOSITE,ru-blocked,PROXY
  - GEOIP,RU,DIRECT
  - MATCH,PROXY
```

> Данные прокси взять из панели 3x-ui → Inbounds → иконка ссылки → скопировать `vless://...`

### Systemd-сервис

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

## Этап 4 — Перенаправление трафика (TPROXY)

> ⚠️ Все команды выполняются в том же LXC (Tailscale + Mihomo).

```bash
apt install -y iptables iptables-persistent

# Mihomo работает локально
MIHOMO_IP=127.0.0.1
MIHOMO_PORT=7893

# Исключить собственный трафик LXC (eth0)
iptables -t mangle -A PREROUTING -i eth0 -j RETURN

# Перенаправить трафик с телефона (tailscale0) в Mihomo
iptables -t mangle -A PREROUTING -i tailscale0 -p tcp \
  -j TPROXY --on-ip $MIHOMO_IP --on-port $MIHOMO_PORT --tproxy-mark 1

iptables -t mangle -A PREROUTING -i tailscale0 -p udp \
  -j TPROXY --on-ip $MIHOMO_IP --on-port $MIHOMO_PORT --tproxy-mark 1

ip rule add fwmark 1 table 100
ip route add local 0.0.0.0/0 dev lo table 100

netfilter-persistent save
```

Проверить что правила применились:

```bash
iptables -t mangle -L PREROUTING -n -v
```

### Автозапуск ip rule / ip route

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

## Этап 5 — Автообновление баз

```bash
crontab -e
```

```
0 */6 * * * wget -q -O /etc/mihomo/geosite.dat https://github.com/runetfreedom/russia-v2ray-rules-dat/releases/latest/download/geosite.dat && wget -q -O /etc/mihomo/geoip.dat https://github.com/runetfreedom/russia-v2ray-rules-dat/releases/latest/download/geoip.dat && systemctl restart mihomo
```

---

## Этап 6 — Настройка телефона

### Android

1. Установить **Tailscale** из Play Store
2. Войти в аккаунт
3. **Use exit node** → выбрать домашнюю ноду (gmk-tec)
4. В настройках Tailscale включить **Use Tailscale subnets** — для доступа к локальной сети

### iOS

1. Установить **Tailscale** из App Store
2. Войти в аккаунт → выбрать exit node

### Доступ к локальным сервисам

С телефона через Tailscale доступна вся домашняя сеть по обычным IP:

- Proxmox: `https://192.168.1.X:8006`
- Mihomo UI: `http://192.168.1.X:9090/ui`

---

## Веб-интерфейс Mihomo (Yacd)

Mihomo автоматически скачивает UI при первом запуске (нужен интернет).

Открыть: `http://<IP LXC>:9090/ui`

- **Host:** `http://<IP LXC>:9090`
- **Secret:** пароль из `config.yaml`

В разделе **Proxies** → группа **PROXY** → выбрать нужный прокси (vdsina или DIRECT).

---

## Проверка и отладка

```bash
# Статус Tailscale
tailscale status
tailscale ping <устройство>

# Логи Mihomo в реальном времени
journalctl -u mihomo -f

# Трафик с телефона
tcpdump -i tailscale0 -n port 53

# Правила iptables
iptables -t mangle -L PREROUTING -n -v
```

---

## Частые проблемы

| Симптом | Причина | Решение |
|---|---|---|
| Tailscale не стартует | Нет `/dev/tun` | Добавить `lxc.mount.entry` в конфиг LXC |
| Tailscale не авторизуется | Mihomo перехватывает DNS | `systemctl stop mihomo`, авторизоваться, `systemctl start mihomo` |
| Mihomo не запускается | Опечатка в config.yaml | `journalctl -u mihomo -n 20` — покажет строку с ошибкой |
| Нет интернета в LXC | Не указан Gateway | CT → Network → добавить Gateway (IP роутера) |
| Трафик не идёт через Mihomo | Нет модуля xt_TPROXY | `modprobe xt_TPROXY` на хосте Proxmox |
| Proxmox не открывается с телефона | Subnet routes не включены | Tailscale Admin → одобрить подсеть; на телефоне включить subnets |
| Нода оффлайн в статусе | Задержка обновления | `tailscale ping <устройство>` — реальная проверка связи |

---

## Структура файлов

```
/etc/pve/lxc/<ID>.conf     ← конфиг LXC (lxc.mount.entry)

/etc/mihomo/
  config.yaml              ← конфигурация Mihomo
  geosite.dat              ← базы доменов (runetfreedom, обновляется cron)
  geoip.dat                ← базы IP (runetfreedom, обновляется cron)
  ui/                      ← веб-интерфейс Yacd

/etc/systemd/system/
  mihomo.service           ← сервис автозапуска

/etc/rc.local              ← автозапуск ip rule / ip route
```

---

## Следующие шаги

- [ ] [[AdGuard Home + Nginx Proxy Manager]] — свой DNS, блокировка рекламы, доступ к сервисам по имени
- [ ] [[Mihomo — расширенные правила]] — антидетект для банковских приложений
- [ ] [[Zapret на роутере (OpenWrt)]] — обход DPI для Discord и EFT прямо на роутере, без конфликтов с Tailscale
- [ ] [[Proxmox — бэкапы LXC]] — резервное копирование контейнеров

---

*Актуально для Proxmox VE 9.x, Tailscale 1.6x+, Mihomo 1.19+*
