# AdGuard Home + Nginx Proxy Manager

> Свой DNS-сервер с блокировкой рекламы и доступ к домашним сервисам по имени вместо IP:порт.
> Дополнение к [[Tailscale + Mihomo на Proxmox]].

---

## Что даёт эта связка

| Без неё | С ней |
|---|---|
| `https://192.168.1.104:8006` | `https://proxmox.home` |
| `http://192.168.1.163:9090/ui` | `http://mihomo.home` |
| Реклама везде | Реклама заблокирована на всех устройствах |
| Стандартный DNS провайдера | Свой DNS с кэшем и контролем |

---

## Архитектура

```
Устройства в сети
      │
      ▼
AdGuard Home (DNS, порт 53)
      │
      ├── proxmox.home → 192.168.1.80  ← IP Nginx PM !
      ├── mihomo.home  → 192.168.1.80  ← IP Nginx PM !
      ├── adguard.home → 192.168.1.53
      └── Реклама      → заблокировано
      │
      ▼
Nginx Proxy Manager (192.168.1.80)
      │
      ├── Host: proxmox.home → 192.168.1.104:8006
      └── Host: mihomo.home  → 192.168.1.163:9090
```

---

## Этап 1 — LXC для AdGuard Home

### 1.1 Создание контейнера

В Proxmox UI → **Create CT**:

| Параметр | Значение |
|---|---|
| Hostname | `AdGuard` |
| Template | Debian 12 |
| Disk | 2 GB |
| RAM | 256 MB |
| Network | bridge `vmbr0`, статический IP (например `192.168.1.53/24`), **Gateway: `192.168.1.1`** |
| Firewall | снять галочку |

> **Обязательно указать Gateway** — без него контейнер не имеет доступа в интернет (`Network is unreachable`). Это частая ошибка при создании LXC.

> Статический IP удобен — его прописываешь в настройках роутера как DNS-сервер.

Включить Nesting: **CT → Options → Features → Nesting ✓**

### 1.2 Установка AdGuard Home

```bash
apt update && apt install -y curl

curl -sSL https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

Открыть в браузере: `http://192.168.1.53:3000` — запустится мастер установки (5 шагов).

**Шаг 1 — Welcome:** нажать **Get Started**.

**Шаг 2 — Admin Web Interface + DNS** (скриншот):
- Admin Web Interface → Listen interface: `All interfaces`, Port: `80`
- DNS server → Listen interface: `All interfaces`, Port: `53`
- Нажать **Next**

> Если порт 80 занят — поставить `3000`, тогда веб-интерфейс будет постоянно на 3000.

**Шаг 3 — Authentication:**
- Задать логин и пароль администратора
- Нажать **Next**

**Шаг 4 — Configure devices:**
- Страница показывает IP-адреса DNS-сервера — ничего менять не нужно
- Нажать **Next**

**Шаг 5 — Done:**
- Нажать **Open Dashboard** → откроется веб-интерфейс уже на порту **80** (или 3000)

После настройки AdGuard Home слушает на порту **53** (DNS) и **80** (веб-интерфейс).

### 1.3 Прописать AdGuard как DNS на роутере (OpenWrt)

Есть два способа. **Способ 1 — рекомендуемый** (клиенты видят AdGuard напрямую, в логах AdGuard видны реальные IP устройств):

**Network → Interfaces → LAN → Edit → вкладка DHCP Server → Advanced Settings**
- Поле **DHCP-Options**: добавить `6,192.168.1.53`

Это говорит Dnsmasq: «раздавай клиентам AdGuard как DNS-сервер через DHCP».

---

**Способ 2 — через DNS forwardings** (проще, но AdGuard видит только IP роутера, не клиентов):

**Network → DHCP and DNS → General Settings**
- Поле **DNS forwardings**: нажать `+`, ввести `/#/192.168.1.53`

Синтаксис: `/#/` = все домены → forwarding на `192.168.1.53`.

---

> После сохранения и Apply — переподключить устройства (или дождаться обновления DHCP lease), чтобы они получили новый DNS.

Проверка на ПК:
```bash
# Linux/Mac
nslookup google.com 192.168.1.53

# Windows
nslookup google.com 192.168.1.53
```

Если ответ приходит — AdGuard работает. В AdGuard Home → **Query Log** появятся запросы.

### 1.4 Настроить upstream DNS-серверы

> Это обязательный шаг. Без upstream DNS AdGuard не может резолвить ничего сам — ни обновления проверить, ни интернет-запросы клиентов обработать.

**AdGuard Home → Settings → DNS settings → Upstream DNS servers:**

```
https://dns10.quad9.net/dns-query
8.8.8.8
1.1.1.1
```

Нажать **Apply** → **Test upstreams** (кнопка рядом) — все три должны показать ✓.

После этого ошибка «Update check failed» пропадёт.

---

> **Диагностика**, если ошибка осталась — из консоли LXC:
> ```bash
> ping -c 3 8.8.8.8          # есть ли интернет
> cat /etc/resolv.conf        # что прописано как DNS у контейнера
> ```
> Если ping не проходит — проблема в сети LXC (проверь bridge vmbr0 и настройки сети Proxmox).

---

### 1.5 Добавить кастомные локальные имена

В AdGuard Home → **Filters → DNS rewrites → Add DNS rewrite**:

| Домен | IP | Куда |
|---|---|---|
| `proxmox.home` | `192.168.1.80` | Nginx PM |
| `mihomo.home` | `192.168.1.80` | Nginx PM |
| `adguard.home` | `192.168.1.53` | AdGuard |

> DNS rewrites должны указывать на **IP Nginx Proxy Manager** (`192.168.1.80`), а не на сервисы напрямую. NPM получит запрос, посмотрит на заголовок `Host:` и перенаправит куда нужно. Если указать IP сервиса — NPM будет обойдён.

> **До настройки NPM** (Этап 2) сервисы доступны с явным портом:
> - `https://192.168.1.104:8006` — Proxmox
> - `http://192.168.1.163:9090/ui` — Mihomo

---

## Этап 2 — LXC для Nginx Proxy Manager

### 2.1 Создание контейнера

| Параметр | Значение |
|---|---|
| Hostname | `NginxPM` |
| Template | Debian 12 |
| Disk | 4 GB |
| RAM | 256 MB |
| Network | bridge `vmbr0`, статический IP (например `192.168.1.80`) |

### 2.2 Установка Docker

```bash
apt update && apt install -y curl

curl -fsSL https://get.docker.com | sh
```

### 2.3 Установка Nginx Proxy Manager

```bash
mkdir -p /opt/nginxpm && cd /opt/nginxpm

cat > docker-compose.yml << 'EOF'
services:
  app:
    image: jc21/nginx-proxy-manager:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "81:81"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
EOF

docker compose up -d
```

Открыть веб-интерфейс: `http://192.168.1.80:81`

Логин по умолчанию:
- Email: `admin@example.com`
- Password: `changeme`

### 2.4 Добавить прокси-хосты

В Nginx Proxy Manager → **Proxy Hosts → Add Proxy Host**:

**Proxmox:**
- Domain: `proxmox.home`
- Scheme: `https`
- Forward IP: `192.168.1.104`
- Port: `8006`
- Включить: **Websockets Support**
- SSL: отключить верификацию (`SSL → Custom certificate → off`)

**Mihomo UI:**
- Domain: `mihomo.home`
- Scheme: `http`
- Forward IP: `192.168.1.163` (или IP Tailscale LXC)
- Port: `9090`

---

## Проверка

1. На любом устройстве в сети открыть `http://proxmox.home` — должен открыться Proxmox
2. В AdGuard Home → **Query Log** — видны все DNS-запросы и что заблокировано

---

## Следующие шаги

- [ ] Настроить списки блокировки в AdGuard Home (AdGuard DNS filter, EasyList Russia)
- [ ] Настроить SSL-сертификаты для локальных доменов (Let's Encrypt или self-signed)
- [ ] Добавить AdGuard Home в Tailscale для работы вне дома

---

*Актуально для AdGuard Home 0.107+, Nginx Proxy Manager 2.x*
