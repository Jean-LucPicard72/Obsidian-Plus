# Mihomo — расширенные правила

> Дополнение к [[Tailscale + Mihomo на Proxmox]]. Здесь — тонкая настройка правил, GeoIP, антидетект для приложений с VPN-проверкой.

---

## Проблема: приложения с антивпн-защитой

Банки, госсервисы, стриминги используют **активное зондирование**:

```
Приложение запускается
       ↓
Пингует "маркерные" хосты:
  - 8.8.8.8
  - известные заблокированные сайты (google.com, instagram.com)
  - connectivitycheck.gstatic.com
       ↓
Если они отвечают → "ты за VPN" → блокирует функционал
```

**Твоя ситуация:** выходной IP — домашний российский. Проблема только в том, что через твою коробку заблокированные сайты *открываются*, а приложение ожидает что они *не работают*.

**Решение — инвертированная логика:** Mihomo имитирует поведение провайдера — заблокированные сайты возвращают REJECT.

---

## Как найти маркеры конкретного приложения

Запустить `tcpdump` на LXC Tailscale пока открываешь приложение:

```bash
# Установить
apt install -y tcpdump

# Смотреть трафик с телефона, фильтровать DNS-запросы
tcpdump -i tailscale0 -n port 53

# Или весь трафик с разбивкой по хостам
tcpdump -i tailscale0 -n -A 2>/dev/null | grep -E "Host:|GET |POST "
```

Запустить приложение → посмотреть к каким доменам оно обращается в первые 5-10 секунд. Эти домены — кандидаты в `REJECT`.

Либо включить подробные логи в Mihomo:

```yaml
log-level: debug
```

```bash
journalctl -u mihomo -f | grep -E "REJECT|DIRECT|match"
```

---

## Шаблон правил для антидетекта

```yaml
rules:
  # ── Локальная сеть ──────────────────────────────
  - IP-CIDR,192.168.0.0/16,DIRECT
  - IP-CIDR,10.0.0.0/8,DIRECT
  - IP-CIDR,172.16.0.0/12,DIRECT
  - IP-CIDR,127.0.0.0/8,DIRECT

  # ── Белый список: банки и госсервисы ────────────
  # Всегда DIRECT — нужен RU IP
  - DOMAIN-SUFFIX,gosuslugi.ru,DIRECT
  - DOMAIN-SUFFIX,nalog.ru,DIRECT
  - DOMAIN-SUFFIX,pfr.gov.ru,DIRECT
  - DOMAIN-SUFFIX,mos.ru,DIRECT
  - DOMAIN-SUFFIX,sberbank.ru,DIRECT
  - DOMAIN-SUFFIX,tinkoff.ru,DIRECT
  - DOMAIN-SUFFIX,alfabank.ru,DIRECT
  - DOMAIN-SUFFIX,vtb.ru,DIRECT
  - DOMAIN-SUFFIX,raiffeisen.ru,DIRECT
  - DOMAIN-SUFFIX,vk.com,DIRECT
  - DOMAIN-SUFFIX,yandex.ru,DIRECT
  - DOMAIN-SUFFIX,yandex.net,DIRECT
  - DOMAIN-SUFFIX,mail.ru,DIRECT
  - DOMAIN-SUFFIX,ok.ru,DIRECT

  # ── REJECT: маркерные домены для детекции VPN ───
  # Приложения пингуют их, ожидая что они НЕ работают
  # Mihomo отвечает "нет" — приложение успокаивается
  - DOMAIN,connectivitycheck.gstatic.com,REJECT
  - DOMAIN,connectivitycheck.android.com,REJECT
  - DOMAIN,captive.apple.com,REJECT
  - DOMAIN,detectportal.firefox.com,REJECT
  - DOMAIN,www.msftconnecttest.com,REJECT
  - DOMAIN-SUFFIX,google.com,REJECT
  - DOMAIN-SUFFIX,googleapis.com,REJECT
  - DOMAIN-SUFFIX,youtube.com,REJECT
  - DOMAIN-SUFFIX,instagram.com,REJECT
  - DOMAIN-SUFFIX,facebook.com,REJECT
  - DOMAIN-SUFFIX,twitter.com,REJECT
  - DOMAIN-SUFFIX,t.co,REJECT
  - DOMAIN-SUFFIX,telegram.org,REJECT

  # ── Российские IP — напрямую ─────────────────────
  - GEOIP,RU,DIRECT

  # ── Всё остальное — через прокси ────────────────
  - MATCH,PROXY
```

> ⚠️ Список REJECT — **отправная точка**. Под каждое приложение нужно добавлять свои домены по результатам tcpdump.

---

## GeoIP базы

По умолчанию Mihomo использует встроенную базу. Для актуальных данных — обновить вручную:

```bash
cd /etc/mihomo

# Скачать свежие базы
wget -O Country.mmdb https://github.com/Loyalsoldier/geoip/releases/latest/download/Country.mmdb
wget -O geosite.dat https://github.com/Loyalsoldier/v2ray-rules-dat/releases/latest/download/geosite.dat

# Перезапустить
systemctl restart mihomo
```

Использование в правилах:

```yaml
rules:
  - GEOSITE,category-ads-all,REJECT      # реклама
  - GEOSITE,ru,DIRECT                     # российские домены
  - GEOSITE,private,DIRECT               # локальные адреса
  - GEOIP,RU,DIRECT
  - MATCH,PROXY
```

---

## Два профиля: дома и в роуминге

Удобно держать два конфига и переключаться между ними.

### Профиль "Дома" (без exit node)

Mihomo только для разблокировки — трафик выходит с домашним IP напрямую:

```yaml
# /etc/mihomo/config-home.yaml
rules:
  - GEOIP,RU,DIRECT
  - DOMAIN-SUFFIX,gosuslugi.ru,DIRECT
  # ... белый список
  - MATCH,PROXY   # остальное через прокси для разблокировки
```

### Профиль "В роуминге" (с exit node через Tailscale)

Весь трафик через домашнюю коробку + инвертированная логика:

```yaml
# /etc/mihomo/config-roaming.yaml
rules:
  - GEOIP,RU,DIRECT
  - DOMAIN-SUFFIX,gosuslugi.ru,DIRECT
  # REJECT для маркерных доменов
  - DOMAIN-SUFFIX,google.com,REJECT
  - DOMAIN-SUFFIX,instagram.com,REJECT
  # ...
  - MATCH,DIRECT  # всё остальное — напрямую (уже через домашний IP)
```

Переключение сервиса:

```bash
# Отредактировать mihomo.service
ExecStart=/usr/local/bin/mihomo -d /etc/mihomo -f /etc/mihomo/config-roaming.yaml

systemctl daemon-reload && systemctl restart mihomo
```

---

## Веб-интерфейс Yacd

Добавить в `config.yaml`:

```yaml
external-controller: 0.0.0.0:9090
external-ui: /etc/mihomo/ui
secret: "yourpassword"   # поменять!
```

Скачать UI:

```bash
mkdir -p /etc/mihomo/ui
wget -O /tmp/yacd.tar.gz https://github.com/haishanh/yacd/releases/latest/download/yacd.tar.xz
tar -xf /tmp/yacd.tar.gz -C /etc/mihomo/ui --strip-components=1
```

Открыть: `http://192.168.1.11:9090/ui`

---

## Векторы fingerprinting и как их закрыть

| Вектор | Как палит | Решение |
|---|---|---|
| **MTU пакетов** | У VPN/tun интерфейсов 1280/1420 вместо 1500 | Установить MTU 1500 на tailscale0: `ip link set tailscale0 mtu 1500` |
| **DNS leak** | DNS-запросы уходят мимо Mihomo | DNS Mihomo должен слушать на `0.0.0.0:53`, клиент смотреть туда |
| **WebRTC leak** | Браузер светит реальный IP через WebRTC | Плагин uBlock Origin → настройки → блокировать WebRTC |
| **Latency паттерны** | Пинг до "заблокированных" хостов нулевой вместо timeout | REJECT возвращает connection refused, это нормально |
| **IP геолокация** | Прокси с зарубежным IP | У тебя выходной IP домашний RU — проблемы нет |

---

## Связанные заметки

- [[Tailscale + Mihomo на Proxmox]] — основной гайд
- [[Tailscale — MagicDNS]] — красивые имена для устройств
- [[Proxmox — бэкапы LXC]] — резервное копирование контейнеров

---

*Актуально для Mihomo 1.18+, Proxmox VE 9.x*
