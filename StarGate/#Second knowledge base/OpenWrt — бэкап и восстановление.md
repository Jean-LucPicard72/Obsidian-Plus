# OpenWrt — бэкап и восстановление

> Сохранить рабочую конфигурацию роутера перед любыми серьёзными изменениями — установкой пакетов, правкой firewall, экспериментами с нестандартными настройками.

---

## Что бэкапить

OpenWrt хранит всю пользовательскую конфигурацию в `/etc/config/` и смежных папках. Прошивка при этом остаётся нетронутой — бэкап конфига весит ~50–200 KB и восстанавливается за секунды.

| Что | Где | Входит в стандартный бэкап |
|---|---|---|
| Настройки сети, WiFi, firewall | `/etc/config/` | ✓ |
| Установленные пакеты | `/overlay/usr/` | ✓ (сами файлы) |
| Кастомные скрипты | `/etc/init.d/`, `/etc/rc.local` | ✓ |
| Пароль root | `/etc/shadow` | ✓ |
| Сама прошивка (firmware) | flash-память | ✗ (отдельно) |

---

## Способ 1 — Веб-интерфейс LuCI (рекомендуется)

**System → Backup / Flash Firmware → Backup**

Нажать **Generate archive** → скачается файл `backup-<hostname>-<date>.tar.gz`.

Сохранить на ПК. Это всё.

> ⚠️ Если LuCI не установлен: `opkg update && opkg install luci`

---

## Способ 2 — SSH (быстро, без браузера)

```bash
# Скачать бэкап одной командой с ПК (Linux/Mac/WSL)
scp root@192.168.1.1:/etc/backup-openwrt.tar.gz ./
# или сгенерировать и скачать:
ssh root@192.168.1.1 'sysupgrade --create-backup - ' > backup-openwrt-$(date +%F).tar.gz
```

На Windows (PowerShell с OpenSSH):
```powershell
ssh root@192.168.1.1 "sysupgrade --create-backup /tmp/backup.tar.gz && cat /tmp/backup.tar.gz" > backup-openwrt.tar.gz
```

Проверить содержимое:
```bash
tar -tzf backup-openwrt-2025-01-01.tar.gz | head -20
```

---

## Что точно попадёт в архив

```bash
# Посмотреть список файлов до создания бэкапа
cat /lib/upgrade/keep.d/*
# или
sysupgrade --list-backup
```

Стандартно включает: `/etc/config/*`, `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/dropbear/`, `/etc/firewall.user`, `/etc/rc.local` и всё из `/lib/upgrade/keep.d/`.

Если добавил свои файлы в нестандартные места (например `/opt/zapret/`):

```bash
echo "/opt/zapret/config" >> /lib/upgrade/keep.d/zapret
echo "/etc/zapret/" >> /lib/upgrade/keep.d/zapret
```

---

## Восстановление

### Через LuCI

**System → Backup / Flash Firmware → Restore**

Загрузить `.tar.gz` → **Upload archive** → роутер применит конфиг и перезагрузится.

### Через SSH

```bash
scp backup-openwrt.tar.gz root@192.168.1.1:/tmp/
ssh root@192.168.1.1 'sysupgrade --restore-backup /tmp/backup.tar.gz'
```

### Если роутер не загружается — Failsafe Mode

1. Отключить питание
2. Включить и **сразу быстро нажимать кнопку Reset** (обычно 5–10 раз)
3. Дождаться быстрого мигания светодиода (failsafe)
4. Подключиться по кабелю → `telnet 192.168.1.1`
5. В telnet:
```bash
mount_root        # смонтировать overlay
firstboot         # сбросить overlay к заводскому состоянию
reboot -f
```
После сброса — залить бэкап через LuCI или SSH как описано выше.

---

## Сброс к заводским настройкам (когда бэкапа нет)

```bash
# Через SSH
firstboot && reboot -f

# Или физически: зажать кнопку Reset на ~10 секунд при включённом роутере
```

---

## Автоматический бэкап на NAS / ПК

Если хочется бэкапиться регулярно — добавить cron на роутере:

```bash
crontab -e
```

```
# Каждое воскресенье в 03:00 — бэкап на NAS (192.168.1.X)
0 3 * * 0 sysupgrade --create-backup /tmp/openwrt-backup.tar.gz && scp /tmp/openwrt-backup.tar.gz user@192.168.1.100:/backups/openwrt/backup-$(date +\%F).tar.gz
```

> Для SCP нужен SSH-ключ без пароля: `ssh-keygen` на роутере → скопировать публичный ключ на NAS.

---

## Перед экспериментом: быстрый чеклист

- [ ] Бэкап сделан и скачан на ПК
- [ ] Проверил содержимое архива (`tar -tzf`)
- [ ] Знаю как зайти в Failsafe Mode на своём роутере (кнопка, светодиод)
- [ ] Есть кабель для прямого подключения к роутеру

---

## Связанные заметки

- [[Zapret на роутере (OpenWrt)]] — DPI-байпас для Discord, YouTube, EFT
- [[AdGuard Home + Nginx Proxy Manager]] — DNS и обратный прокси

---

*Актуально для OpenWrt 21.02+*
