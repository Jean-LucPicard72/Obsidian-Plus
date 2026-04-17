# Tailscale на VPS — с нуля

> **Цель:** Поднять Tailscale на чистом VPS за ~10 минут.  
> VPS становится узлом сети и может работать как **Exit Node** — весь трафик устройств идёт через него.

---

## Что нужно до начала

| Требование | Минимум |
|---|---|
| VPS | Любой провайдер, Debian 12 или Ubuntu 22.04/24.04 |
| CPU / RAM | 1 vCPU / 128 МБ |
| Доступ | Root или sudo-пользователь по SSH |
| Аккаунт | Бесплатный на [tailscale.com](https://tailscale.com) |

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

На **локальной машине**:
```bash
ssh-keygen -t ed25519 -C "tailscale-vps"
ssh-copy-id deploy@<IP_СЕРВЕРА>
```

После проверки что вход по ключу работает — отключить пароль:
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

---

## Этап 1 — Установка Tailscale

### 1.1 Официальный скрипт установки

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Скрипт сам определит дистрибутив, добавит репозиторий и установит `tailscale` и `tailscaled`.

### 1.2 Проверить что установилось

```bash
tailscale version
# Пример: 1.78.3
```

---

## Этап 2 — Включить IP-форвардинг

Это обязательно для работы Exit Node — без него трафик других устройств не пройдёт через VPS.

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Проверить что применилось:
```bash
sysctl net.ipv4.ip_forward
# Ответ: net.ipv4.ip_forward = 1
```

---

## Этап 3 — Запуск и авторизация

### 3.1 Запуск с флагом Exit Node

```bash
sudo tailscale up --advertise-exit-node
```

В терминале появится ссылка вида:
```
To authenticate, visit:
https://login.tailscale.com/a/...
```

Перейди по ссылке в браузере → войди в аккаунт → нажми **Connect**.

### 3.2 Проверить статус

```bash
tailscale status
```

Сервер должен появиться в списке со статусом `active`.

---

## Этап 4 — Одобрить Exit Node в Tailscale Admin Console

Tailscale не активирует Exit Node автоматически — нужно подтвердить в панели управления.

1. Перейди на **[login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines)**
2. Найди свой VPS в списке
3. Нажми `...` → **Edit route settings**
4. Включи переключатель **Use as exit node** ✓
5. Нажми **Save**

После этого нода готова принимать трафик.

---

## Этап 5 — Включить Exit Node на устройстве

Трафик не пойдёт через VPS автоматически — нужно выбрать ноду на каждом клиенте.

### Windows / macOS

1. Открыть иконку Tailscale в трее
2. **Exit node** → выбрать свой VPS

### Android / iOS

1. Открыть приложение Tailscale
2. Нажать на активную сеть
3. **Use exit node** → выбрать VPS

### Linux (CLI)

```bash
tailscale up --exit-node=<HOSTNAME_или_IP>

# Пример: выбрать по имени ноды
tailscale up --exit-node=my-vps

# Отключить exit node
tailscale up --exit-node=
```

> `<HOSTNAME>` — имя устройства из `tailscale status`

---

## Этап 6 — Автозапуск

Tailscale автоматически добавляет себя в systemd при установке. Убедиться:

```bash
systemctl is-enabled tailscaled
# Ответ: enabled

systemctl status tailscaled
# Ожидаемый статус: active (running)
```

Если вдруг не включён:
```bash
systemctl enable --now tailscaled
```

### Сохранить флаги запуска

При перезагрузке `tailscaled` стартует, но `tailscale up` нужно вызвать снова с теми же флагами. Чтобы не делать это вручную — создать override-файл:

```bash
mkdir -p /etc/systemd/system/tailscaled.service.d/

cat > /etc/systemd/system/tailscaled.service.d/override.conf << 'EOF'
[Service]
ExecStartPost=/usr/bin/tailscale up --advertise-exit-node
EOF

systemctl daemon-reload
systemctl restart tailscaled
```

> Tailscale 1.56+ запоминает последние флаги сам — override нужен только на более старых версиях или если хочется явного контроля.

---

## Firewall

Tailscale сам настраивает нужные правила через `nftables`/`iptables`, но если используешь UFW — убедись что порты не заблокированы.

### UFW (Ubuntu)

```bash
# Tailscale использует UDP 41641 для peer-to-peer
ufw allow 41641/udp comment "Tailscale"

# Разрешить форвардинг (нужно для exit node)
sed -i 's/DEFAULT_FORWARD_POLICY="DROP"/DEFAULT_FORWARD_POLICY="ACCEPT"/' /etc/default/ufw
ufw reload
```

### Проверить что форвардинг работает

```bash
ufw status verbose | grep -i forward
# Ожидаемый вывод: Default: disabled (incoming), allow (outgoing), allow (routed)
```

---

## Дополнительно — Subnet Router

Если хочешь открыть доступ к локальной сети VPS (например к приватным сервисам), добавь флаг:

```bash
sudo tailscale up \
  --advertise-exit-node \
  --advertise-routes=10.0.0.0/8
```

> Заменить `10.0.0.0/8` на актуальную подсеть VPS — посмотреть через `ip route`.

После этого одобрить маршрут в Admin Console так же как Exit Node.

---

## Полезные команды

```bash
# Статус сети — все устройства и их IP
tailscale status

# Свой Tailscale IP
tailscale ip -4

# Пинг до другого устройства в сети
tailscale ping <имя_устройства>

# Диагностика соединения
tailscale netcheck

# Посмотреть текущие флаги
tailscale status --self

# Логи
journalctl -u tailscaled -f

# Отключить exit node (на клиенте)
tailscale up --exit-node=

# Полностью отключиться от сети
tailscale down

# Обновить Tailscale
apt update && apt upgrade tailscale
```

---

## Решение частых проблем

| Симптом | Причина | Решение |
|---|---|---|
| Exit node не появляется в списке устройств | Не одобрена в Admin Console | Включить в Edit route settings |
| Трафик не идёт через VPS | IP-форвардинг выключен | Проверить `sysctl net.ipv4.ip_forward` |
| Tailscale не запускается | Нет `/dev/tun` (в контейнере) | Включить TUN-устройство в настройках контейнера |
| Высокий пинг через exit node | DERP-релей вместо прямого соединения | Запустить `tailscale netcheck`, проверить NAT |
| После перезагрузки нода без флага `--advertise-exit-node` | Флаги не сохранились | Добавить override для systemd (Этап 6) |

### Проверить что трафик идёт через VPS

На клиентском устройстве после включения Exit Node:
```bash
curl https://ipinfo.io/ip
# Должен вернуть IP твоего VPS, а не домашний
```

### Нода не может выйти в интернет

```bash
# На VPS проверить форвардинг
cat /proc/sys/net/ipv4/ip_forward
# Должно быть: 1

# Проверить что Tailscale видит флаг exit node
tailscale status --self | grep ExitNode
```

---

## Структура файлов

```
/etc/sysctl.d/99-tailscale.conf   ← IP forwarding
/etc/systemd/system/tailscaled.service.d/override.conf  ← флаги запуска (опционально)
/var/lib/tailscale/                ← состояние и ключи (не трогать)
```

---

## Связанные заметки

- [[Tailscale + Mihomo на Proxmox]] — Tailscale + умная маршрутизация через Mihomo
- [[MTProto Proxy — с нуля на новом VPS]] — MTProto-прокси для Telegram на том же VPS

---

*Гайд актуален для Tailscale 1.7x+, Debian 12, Ubuntu 22.04/24.04*
