# Zapret на роутере (OpenWrt)

> **Цель:** Обход DPI провайдера для всех устройств в сети — Discord, Escape from Tarkov, YouTube на Smart TV и любое другое заблокированное — прямо на роутере, без VPN-приложений на каждом устройстве и без конфликтов с Tailscale.
> Дополнение к [[Tailscale + Mihomo на Proxmox]].

---

## Почему роутер, а не Windows

**Проблема с Zapret на Win11:**
Zapret перехватывает пакеты через WinDivert до того, как они уйдут в сеть. Когда Tailscale включён как VPN-клиент, он меняет таблицу маршрутизации — часть трафика уходит в туннель раньше, чем WinDivert успевает его обработать. Приходится выключать Tailscale вручную.

**Решение — Zapret на роутере:**

```
PC (Tailscale включён)  ──┐
Smart TV (Tizen/webOS)  ──┤
Телефон, приставка...   ──┤
                           ▼
                      OpenWrt роутер
                           │
                      [nfqws/tpws]  ← Zapret обрабатывает пакеты здесь
                           │             до того как они уходят к провайдеру
                           ▼
                      ISP → DPI оборудование  ← видит «мусор» вместо TLS ClientHello
                           │
                           ▼
                      YouTube / Discord / EFT серверы
```

Всё что подключено к роутеру получает DPI-байпас автоматически. На Smart TV не нужно ставить VPN-приложение (и возможности такой нет) — телек просто использует роутер как шлюз по умолчанию, как делал всегда.

Tailscale на ПК создаёт туннель для удалённого доступа к домашней сети — это никак не мешает роутеру обрабатывать исходящий трафик на уровне провайдера.

---

## Как работает Zapret

РКН обязал провайдеров установить ТСПУ — оборудование глубокой инспекции пакетов. Оно анализирует TLS ClientHello (первый пакет HTTPS-соединения) и блокирует/замедляет Discord, некоторые серверы EFT.

Zapret разбивает или подделывает этот первый пакет — DPI видит «мусор» и пропускает соединение, а сервер получает корректные данные и всё работает.

---

## Что понадобится

- Роутер с OpenWrt 22.03+ (x86, ARM, MIPS — значения не имеет)
- SSH-доступ к роутеру
- ~2 MB свободного места в `/overlay`
- Внешний интерфейс: обычно `wan` или `eth1`

Проверить место:
```bash
df -h /overlay
```

---

## Перед началом — бэкап

> ⚠️ Zapret добавляет пакеты, кастомные nftables-правила и init-скрипт. Сделай бэкап конфига роутера до того, как начнёшь.
> Инструкция: [[OpenWrt — бэкап и восстановление]]

---

## Этап 1 — Установка Zapret

### 1.1 Через opkg (OpenWrt 23.05+)

```bash
opkg update
opkg install zapret
```

### 1.2 Вручную (если нет в opkg или нужна свежая версия)

Определить архитектуру роутера:
```bash
opkg print-architecture | head -1
```

| Вывод | Архитектура бинарников |
|---|---|
| `mipsel_24kc` | `mipsel` |
| `arm_cortex-a7` | `arm` |
| `x86_64` | `x86_64` |
| `aarch64_cortex-a53` | `aarch64` |

Скачать [последний релиз](https://github.com/bol-van/zapret/releases) под свою архитектуру:

```bash
# Пример для mipsel (TP-Link, Xiaomi и др.)
cd /tmp
wget https://github.com/bol-van/zapret/releases/latest/download/zapret-openwrt-mipsel.tar.gz
tar -xzf zapret-openwrt-mipsel.tar.gz -C /opt
```

Проверить что бинарники работают:
```bash
/opt/zapret/nfq/nfqws --version
/opt/zapret/tpws/tpws --version
```

---

## Этап 2 — Настройка для Discord и EFT

### 2.1 Файл конфигурации

```bash
vi /etc/zapret/config
```

> Если файла нет — создать: `touch /etc/zapret/config`

```bash
# Внешний интерфейс (смотреть в Network → Interfaces → WAN → Device)
IFACE_WAN=wan

# Внутренний интерфейс (LAN)
IFACE_LAN=br-lan

# Режим: только nfqws (TCP + UDP через NFQUEUE)
TPWS_ENABLE=0
NFQWS_ENABLE=1

# ─── Параметры для TCP (HTTPS / TLS) ───────────────────────────────────
# split2     — разбить ClientHello на два пакета
# fake       — отправить поддельный пакет с маленьким TTL перед настоящим
# disorder2  — изменить порядок пакетов
# ttl=5      — поддельный пакет «умрёт» не добравшись до DPI (подбирать!)
NFQWS_OPT_DESYNC_HTTP="--dpi-desync=fake,disorder2 --dpi-desync-ttl=5 --dpi-desync-fooling=md5sig"

# ─── Параметры для UDP 443 (Discord QUIC — voice/video) ─────────────────
NFQWS_OPT_DESYNC_HTTPS_UDP="--dpi-desync=fake --dpi-desync-any-protocol --dpi-desync-fake-quic --dpi-desync-ttl=5 --dpi-desync-fooling=md5sig"

# Применять ко всем адресатам (не только из ipset)
NFQWS_OPT_DESYNC_BYPASS_IP=0

# Номера очередей NFQUEUE
NFQUEUE_BASE=200
```

### 2.2 Правила nftables

Создать файл с правилами перехвата трафика:

```bash
vi /etc/zapret/rules.nft
```

```nft
table inet zapret {
    chain forward {
        type filter hook forward priority mangle; policy accept;

        # TCP 443 — HTTPS (Discord, EFT launcher, Steam)
        # Перехватываем исходящий трафик из LAN
        oifname "wan" tcp dport 443 ct state new,established
            queue num 200 bypass

        # UDP 443 — QUIC (Discord voice/video)
        oifname "wan" udp dport 443
            queue num 201 bypass
    }
}
```

> `bypass` — если nfqws не запущен, пакеты проходят без обработки (не блокируются).

Применить и сохранить:
```bash
nft -f /etc/zapret/rules.nft
nft list ruleset | grep zapret   # проверить
```

### 2.3 Запуск nfqws

Два экземпляра — для TCP и UDP:

```bash
# TCP 443 → очередь 200
/opt/zapret/nfq/nfqws \
  --daemon \
  --pidfile=/var/run/nfqws-tcp.pid \
  --queue-num=200 \
  --dpi-desync=fake,disorder2 \
  --dpi-desync-ttl=5 \
  --dpi-desync-fooling=md5sig

# UDP 443 → очередь 201
/opt/zapret/nfq/nfqws \
  --daemon \
  --pidfile=/var/run/nfqws-udp.pid \
  --queue-num=201 \
  --dpi-desync=fake \
  --dpi-desync-any-protocol \
  --dpi-desync-fake-quic \
  --dpi-desync-ttl=5 \
  --dpi-desync-fooling=md5sig
```

Проверить что запущены:
```bash
ps | grep nfqws
```

---

## Этап 3 — Автозапуск

### 3.1 Init-скрипт

```bash
vi /etc/init.d/zapret
```

```sh
#!/bin/sh /etc/rc.common

START=95
STOP=10
USE_PROCD=1

NFT_FILE=/etc/zapret/rules.nft
NFQWS=/opt/zapret/nfq/nfqws

start_service() {
    # Загрузить правила nftables
    nft -f "$NFT_FILE"

    # TCP 443
    procd_open_instance "tcp"
    procd_set_param command "$NFQWS" \
        --queue-num=200 \
        --dpi-desync=fake,disorder2 \
        --dpi-desync-ttl=5 \
        --dpi-desync-fooling=md5sig
    procd_set_param respawn
    procd_close_instance

    # UDP 443 (QUIC)
    procd_open_instance "udp"
    procd_set_param command "$NFQWS" \
        --queue-num=201 \
        --dpi-desync=fake \
        --dpi-desync-any-protocol \
        --dpi-desync-fake-quic \
        --dpi-desync-ttl=5 \
        --dpi-desync-fooling=md5sig
    procd_set_param respawn
    procd_close_instance
}

stop_service() {
    nft delete table inet zapret 2>/dev/null || true
}
```

```bash
chmod +x /etc/init.d/zapret
service zapret enable
service zapret start
```

Проверить статус:
```bash
service zapret status
```

---

## Этап 4 — Подбор TTL (обязательно)

`--dpi-desync-ttl=5` — значение по умолчанию. Смысл: поддельный пакет должен «умереть» раньше DPI, но дойти до него (чтобы он его увидел). Зависит от расположения DPI у конкретного провайдера.

Проверить количество хопов до DPI:
```bash
# С ПК в сети роутера
traceroute -n 8.8.8.8
```

Обычно DPI стоит на 2–4 хопе от тебя. Начать с TTL=3, если не работает — увеличивать до 8.

Изменить TTL в `/etc/init.d/zapret` → `--dpi-desync-ttl=<значение>` → `service zapret restart`.

---

## Этап 5 — Проверка

```bash
# Discord должен открываться и работать голос/видео
# EFT — запуститься и подключиться к серверу

# Смотреть трафик через роутер
tcpdump -i wan -n 'tcp port 443 or udp port 443' | head -20

# Проверить что очереди NFQUEUE активны
cat /proc/net/netfilter/nfnetlink_queue

# Логи zapret (если используете logread)
logread | grep nfqws
```

---

## Дополнительно: только Discord и EFT (ipset)

Если не хочется обрабатывать весь HTTPS трафик — ограничить по IP-диапазонам.

**Discord IP-диапазоны:**
```
66.22.192.0/18
162.159.128.0/17
```

**Скрипт обновления ipset:**
```bash
vi /etc/zapret/update-ipset.sh
```

```sh
#!/bin/sh

# Discord
nft add set inet zapret discord_ips '{ type ipv4_addr; flags interval; }'
nft add element inet zapret discord_ips '{ 66.22.192.0/18, 162.159.128.0/17 }'
```

Обновить правила `rules.nft` — добавить условие `ip daddr @discord_ips`:

```nft
# TCP 443 только для Discord
oifname "wan" ip daddr @discord_ips tcp dport 443 ct state new,established
    queue num 200 bypass

# UDP 443 только для Discord
oifname "wan" ip daddr @discord_ips udp dport 443
    queue num 201 bypass

# EFT — весь TCP/UDP трафик с/на их серверы (широко, без ipset)
# EFT использует нестандартные порты — проще оставить весь 443
oifname "wan" tcp dport 443 ct state new,established queue num 200 bypass
oifname "wan" udp dport 443 queue num 201 bypass
```

---

## Частые проблемы

| Симптом | Причина | Решение |
|---|---|---|
| Discord не работает после включения | Неверный TTL | Попробовать `--dpi-desync-ttl=3`, затем 4, 6, 8 |
| nfqws запустился, но нет эффекта | Правила nft не применились | `nft list ruleset` — есть ли таблица `zapret`? |
| EFT подключается, но лагает | UDP-трафик игры не только на 443 | Добавить очередь для UDP 7777-27030 (порты EFT) |
| После перезагрузки не работает | init-скрипт не включён | `service zapret enable` |
| `nfqws: command not found` | Неверная архитектура бинарника | Перепроверить вывод `opkg print-architecture` |
| Конфликт с fw4 (OpenWrt 22.03+) | nft таблица перезаписывается | Добавить `include "/etc/zapret/rules.nft"` в `/etc/nftables.d/` |

### Конфликт с fw4 (OpenWrt 22.03+)

В новых OpenWrt firewall (fw4) управляет nftables и может сбросить кастомные правила при перезагрузке интерфейсов:

```bash
mkdir -p /etc/nftables.d
ln -s /etc/zapret/rules.nft /etc/nftables.d/zapret.nft
```

---

## Структура файлов

```
/opt/zapret/
  nfq/nfqws          ← основной бинарник (TCP/UDP через NFQUEUE)
  tpws/tpws          ← альтернатива (transparent proxy, TCP только)

/etc/zapret/
  config             ← параметры (опционально, если не используешь init напрямую)
  rules.nft          ← правила nftables для перехвата трафика
  update-ipset.sh    ← обновление IP-диапазонов (опционально)

/etc/init.d/zapret   ← автозапуск через procd
```

---

## Связанные заметки

- [[Tailscale + Mihomo на Proxmox]] — основной гайд по инфраструктуре
- [[Mihomo — расширенные правила]] — правила для обхода антивпн-защиты в приложениях
- [[Smart TV — обход блокировок без VPN]] — Medium, ti.com и геоблоки через Mihomo на телике
- [[AdGuard Home + Nginx Proxy Manager]] — DNS и доступ к сервисам по имени

---

*Актуально для OpenWrt 22.03+, Zapret 67+*
