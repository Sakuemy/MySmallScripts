# Отключение парковки головок HDD (APM) на Linux-серверах через systemd

Проверено на Ubuntu 22.04 (ядро 5.15), hdparm 9.x.

## Когда это нужно

Симптомы:
- Алерты Zabbix о высоком `w_await` / `f_await` при почти нулевой нагрузке (0,5-3 IOPS).
- В `iostat -x` у части дисков flush ~20 мс и выше, у остальных ~5 мс.
- Массив (md RAID) чистый, ресинка нет, ошибок в `dmesg` нет.
- В SMART очень большой `Load_Cycle_Count`.

Причина: APM-уровень ниже 128 (например 96 у ноутбучных WD) заставляет диск парковать головки после нескольких секунд простоя. Каждая редкая запись будит диск, а RAID ждёт самого медленного участника. Побочный эффект: быстрый износ (у 2,5" дисков ресурс ~300 тыс. циклов).

Подходит только для HDD, подключённых напрямую по SATA/PATA. Не применимо к SSD, дискам за аппаратным RAID-контроллером (там нужны утилиты вендора: `storcli`, `perccli`, `ssacli`) и большинству USB-мостов.

---

## Шаг 1. Диагностика

```bash
# Установлен ли hdparm и smartmontools
apt install -y hdparm smartmontools

# Список дисков, тип (ROTA=1 это HDD), модель, серийник
lsblk -d -o NAME,ROTA,SIZE,MODEL,SERIAL

# Текущий уровень APM по каждому HDD
for d in /dev/sd?; do echo "== $d"; hdparm -B $d 2>&1 | grep -E 'APM|not supported'; done

# Счётчик парковок и часы работы
for d in /dev/sd?; do echo "== $d"; smartctl -A $d | grep -E 'Load_Cycle|Start_Stop|Power_On_Hours'; done
```

Как читать:
- `APM_level = 254` или `not supported`: парковка по APM не мешает.
- `APM_level` от 1 до 127: диск может останавливать шпиндель, нужно менять.
- `APM_level` 128-253: парковка допустима, но реже. Для серверов лучше тоже поднять до 254.
- `Load_Cycle_Count` в десятках тысяч и растёт: парковка идёт активно.

Чтобы убедиться, что счётчик растёт сейчас:
```bash
smartctl -A /dev/sdX | grep Load_Cycle; sleep 600; smartctl -A /dev/sdX | grep Load_Cycle
```

## Шаг 2. Применить сразу (до перезагрузки)

```bash
hdparm -B 254 /dev/sdX
hdparm -B /dev/sdX        # проверка: APM_level = 254
```

Если диск не поддерживает 254, попробуйте `hdparm -B 255` (полное отключение APM). Если `not supported`, переходите к разделу про WD idle3.

## Шаг 3. Узнать стабильные имена дисков

Имена `sdX` могут меняться между загрузками, поэтому в юните используем `/dev/disk/by-id/`:

```bash
ls -l /dev/disk/by-id/ | grep '^l' | grep -E 'ata-' | grep -v part
```

Берите пути вида `ata-<модель>_<серийник>`, например:
```
ata-WDC_WD10JPCX-24UE4T0_WD-WX91EA40W7A4
```

## Шаг 4. Создать systemd-юнит

Файл `/etc/systemd/system/hdparm-apm.service`:

```ini
[Unit]
Description=Disable APM (head parking) on HDDs
After=local-fs.target

[Service]
Type=oneshot
RemainAfterExit=yes
# Один ExecStart на каждый диск. "-" перед путём: ошибка одного диска не валит юнит.
ExecStart=-/usr/sbin/hdparm -B 254 /dev/disk/by-id/ata-ДИСК_1
ExecStart=-/usr/sbin/hdparm -B 254 /dev/disk/by-id/ata-ДИСК_2
ExecStart=-/usr/sbin/hdparm -B 254 /dev/disk/by-id/ata-ДИСК_3

[Install]
WantedBy=multi-user.target
```

Пример для сервера из этой инструкции (ls13):

```ini
ExecStart=-/usr/sbin/hdparm -B 254 /dev/disk/by-id/ata-WDC_WD10JPCX-24UE4T0_WD-WX91EA40W7A4
ExecStart=-/usr/sbin/hdparm -B 254 /dev/disk/by-id/ata-WDC_WD10JPCX-24UE4T0_WD-WXD1E15EW0XR
ExecStart=-/usr/sbin/hdparm -B 254 /dev/disk/by-id/ata-ST1000DM003-1SB10C_Z9A01B3Q
```

Включить:
```bash
systemctl daemon-reload
systemctl enable --now hdparm-apm.service
systemctl status hdparm-apm.service
```

### Вариант без ручного перечисления дисков

Если дисков много и они однотипны, юнит может применить настройку ко всем вращающимся дискам, которые принимают APM:

```ini
[Unit]
Description=Disable APM (head parking) on all rotational disks
After=local-fs.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/sh -c 'for d in /sys/block/sd*; do [ "$(cat $d/queue/rotational)" = 1 ] && /usr/sbin/hdparm -B 254 /dev/$(basename $d) || true; done'

[Install]
WantedBy=multi-user.target
```

Минус: заденет и те диски, где настройка нежелательна (например, архивные, которые должны засыпать). Ограничьте список, если такие есть.

## Шаг 5. Проверка

Сразу:
```bash
hdparm -B /dev/sdX
```

После перезагрузки (обязательно):
```bash
systemctl is-active hdparm-apm.service
for d in /dev/sd?; do echo "== $d"; hdparm -B $d | grep APM; done
```

Парковка действительно прекратилась:
```bash
for d in /dev/sdX /dev/sdY; do smartctl -A $d | grep Load_Cycle; done
sleep 900
for d in /dev/sdX /dev/sdY; do smartctl -A $d | grep Load_Cycle; done
```
Счётчик не должен расти.

Задержки (до и после, на одной и той же нагрузке):
```bash
apt install -y ioping
ioping -W -c 30 -i 5 /путь/на/массиве
iostat -x 2 5
```
`f_await` и `w_await` у бывших «медленных» дисков должны упасть к уровню остальных.

## Если Load_Cycle_Count продолжает расти (WD)

У части WD (Green/Blue/Mobile) есть собственный таймер парковки idle3, независимый от APM.

```bash
apt install -y idle3-tools
idle3ctl -g /dev/sdX          # прочитать таймер
idle3ctl -d /dev/sdX          # отключить
# затем полностью выключить питание диска (не перезагрузка), чтобы настройка применилась
```
Для дисков новых серий используйте `openSeaChest`/`wdidle3` под конкретную модель. Для Seagate аналог: `openSeaChest_PowerControl --EPCfeature disable`.

## Откат

```bash
systemctl disable --now hdparm-apm.service
rm /etc/systemd/system/hdparm-apm.service
systemctl daemon-reload
# вернуть прежний уровень, например:
hdparm -B 96 /dev/sdX
```

## Чек-лист для каждого нового сервера

1. `lsblk -d -o NAME,ROTA,MODEL,SERIAL`: определить HDD.
2. `hdparm -B` и `smartctl -A | grep Load_Cycle`: есть ли проблема.
3. `hdparm -B 254` вручную и замер `ioping`.
4. Создать `hdparm-apm.service` с by-id путями, `enable --now`.
5. Перезагрузка и повторная проверка `hdparm -B`.
6. Через 15 минут сверить `Load_Cycle_Count`.
7. Проверить `smartctl -a` (Reallocated, Pending, Uncorrectable) и свежесть бэкапа: диски с сотнями тысяч циклов и 50+ тыс. часов работы пора планировать к замене.

## Связанные проблемы, которые стоит проверить

- В `dmesg` строки `NCQ (depth 0/32)` и по два диска на одном канале (`ata3.00` и `ata3.01`): контроллер в режиме IDE/Legacy. Переключите SATA в BIOS на AHCI (после бэкапа) или поставьте AHCI-контроллер.
- Если после отключения APM задержки остались высокими, а нагрузка 1-3 IOPS, поправьте триггер Zabbix: считать `await` только при достаточной нагрузке (например, `w/s > 20`).
