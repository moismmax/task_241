# Пишем юниты

1) Создайте скрипт, который создаёт каталог и четыре файла с именами `1`, `2`, `3`, `4`.
   В файлы запишите соответственно:
   - текущую дату;
   - версию ядра;
   - имя компьютера;
   - список файлов в домашнем каталоге пользователя, от которого выполняется скрипт.
   Не забудьте проверять существование каталогов и файлов.

2) Создайте systemd service, который запускает этот скрипт. Проверьте его работу.
3) Создайте timer, который запускает этот service каждые 5 минут.
4) От какого пользователя по умолчанию выполняются системные units?
5) Создайте отдельного пользователя для выполнения вашего скрипта.
6) Дополните unit так, чтобы скрипт выполнялся от созданного пользователя.
7) Дополните скрипт так, чтобы он независимо от текущего каталога выполнял работу в домашнем каталоге пользователя, от которого запущен.
8) Определите PID процесса во время выполнения, если он успевает существовать достаточно долго.
9) Найдите в journal записи о запуске service.
10) Намеренно внесите ошибку в скрипт или unit, добейтесь состояния `failed`, определите причину через `systemctl` и `journalctl`, затем исправьте.

## Критерий приёмки

В отчёте должны быть:
- полный текст скрипта;
- содержимое созданного service unit;
- содержимое созданного timer unit;
- команды `daemon-reload`, запуска и включения timer;
- вывод, подтверждающий работу service;
- вывод, подтверждающий расписание timer;
- информация о пользователе, от которого запускается скрипт;
- пример записей из journal о запуске;
- пример намеренно созданной ошибки;
- вывод, по которому была определена причина состояния `failed`;
- подтверждение успешной работы после исправления.


# Пишем юниты

## 1. Создание скрипта, который создаёт каталог и четыре файла

В macOS для автоматизации задач используется оболочка Bash или Zsh. Для создания скрипта используется Bash.

Создадим каталог для скрипта:

```bash
mkdir -p ~/scripts
```

Создадим файл:

```bash
nano ~/scripts/create_files.sh
```

Полный текст скрипта:

```bash
#!/bin/bash

set -e

# Определение домашнего каталога пользователя,
# от имени которого запущен скрипт
HOME_DIR="$HOME"

# Каталог для результатов
RESULT_DIR="$HOME_DIR/unit_results"

# Проверка существования домашнего каталога
if [ ! -d "$HOME_DIR" ]; then
    echo "Ошибка: домашний каталог не существует"
    exit 1
fi

# Создание каталога при отсутствии
if [ ! -d "$RESULT_DIR" ]; then
    mkdir -p "$RESULT_DIR"
fi

# Проверка создания каталога
if [ ! -d "$RESULT_DIR" ]; then
    echo "Ошибка: не удалось создать каталог"
    exit 1
fi

# Создание файла 1 с текущей датой
date > "$RESULT_DIR/1"

# Создание файла 2 с версией ядра
uname -r > "$RESULT_DIR/2"

# Создание файла 3 с именем компьютера
scutil --get ComputerName > "$RESULT_DIR/3"

# Создание файла 4 со списком файлов домашнего каталога
ls -la "$HOME_DIR" > "$RESULT_DIR/4"

# Проверка существования всех файлов
for file in 1 2 3 4; do
    if [ ! -f "$RESULT_DIR/$file" ]; then
        echo "Ошибка: файл $file не создан"
        exit 1
    fi
done

echo "Скрипт успешно выполнен"
echo "Результаты сохранены в $RESULT_DIR"
```

Предоставление права на выполнение:

```bash
chmod +x ~/scripts/create_files.sh
```

Запуск:

```bash
~/scripts/create_files.sh
```

Проверка созданных файлов:

```bash
ls -l ~/unit_results
```

Просмотр содержимого:

```bash
for file in ~/unit_results/{1,2,3,4}; do
    echo "Файл: $file"
    cat "$file"
    echo
done
```

Содержимое файлов:
- `1` — текущая дата и время;
- `2` — версия ядра macOS;
- `3` — имя компьютера;
- `4` — список файлов и каталогов домашнего каталога пользователя.

## 2. Создание systemd service

В macOS systemd отсутствует. Для автоматического запуска программ используется служба `launchd`.

В Linux systemd service создаётся в каталоге:

```text
/etc/systemd/system/
```

Пример файла `/etc/systemd/system/create-files.service`:

```ini
[Unit]
Description=Create four files in user home directory
After=network.target

[Service]
Type=oneshot
ExecStart=/home/scriptuser/scripts/create_files.sh
User=scriptuser

[Install]
WantedBy=multi-user.target
```

В macOS аналогичная задача реализуется с помощью LaunchDaemon или LaunchAgent.

Для пользовательского процесса создаётся LaunchAgent:

```bash
mkdir -p ~/Library/LaunchAgents
nano ~/Library/LaunchAgents/com.moismmax.create-files.plist
```

Содержимое файла:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
 "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.moismmax.create-files</string>

    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/Users/moismmax/scripts/create_files.sh</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/Users/moismmax/unit_results/service.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/moismmax/unit_results/service-error.log</string>
</dict>
</plist>
```

Загрузка задания:

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.moismmax.create-files.plist
```

Проверка:

```bash
launchctl list | grep com.moismmax.create-files
```

## 3. Создание timer с запуском каждые 5 минут

В Linux для периодического запуска создаётся файл `/etc/systemd/system/create-files.timer`:

```ini
[Unit]
Description=Run create-files service every 5 minutes

[Timer]
OnBootSec=1min
OnUnitActiveSec=5min
Unit=create-files.service

[Install]
WantedBy=timers.target
```

В macOS аналогичное расписание задаётся в plist-файле с помощью ключа `StartInterval`.

Для этого в файл `com.moismmax.create-files.plist` добавляется:

```xml
<key>StartInterval</key>
<integer>300</integer>
```

Значение `300` означает 300 секунд, то есть 5 минут.

После изменения конфигурации задание необходимо перезагрузить:

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.moismmax.create-files.plist

launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.moismmax.create-files.plist
```

Проверка загруженного задания:

```bash
launchctl list | grep com.moismmax.create-files
```

## 4. От какого пользователя по умолчанию выполняются системные units?

В Linux системные units по умолчанию выполняются от имени пользователя `root`, если в конфигурации не указано другое значение параметра `User`.

В macOS:
- LaunchDaemon обычно выполняется от имени `root`;
- LaunchAgent выполняется от имени пользователя, в чьём пользовательском сеансе он загружен.

В данном случае используется LaunchAgent пользователя `moismmax`.

## 5. Создание отдельного пользователя для выполнения скрипта

В macOS для управления локальными пользователями используется `sysadminctl`.

Создание отдельной учётной записи:

```bash
sudo sysadminctl -addUser scriptuser -password -
```

Проверка пользователя:

```bash
id scriptuser
```

В Linux создание пользователя выполняется командой:

```bash
sudo useradd -m -s /bin/bash scriptuser
```

## 6. Настройка выполнения скрипта от отдельного пользователя

В Linux для запуска службы от отдельного пользователя в service unit указывается:

```ini
[Service]
Type=oneshot
User=scriptuser
ExecStart=/home/scriptuser/scripts/create_files.sh
```

В macOS для выполнения программы от имени отдельного пользователя можно использовать LaunchDaemon с параметром `UserName`.

Пример конфигурации:

```xml
<key>UserName</key>
<string>scriptuser</string>
```

При использовании LaunchAgent скрипт выполняется от имени владельца пользовательского сеанса. Поэтому для выполнения от имени отдельной учётной записи необходимо загрузить LaunchAgent в сеансе `scriptuser` либо использовать LaunchDaemon.

## 7. Независимость от текущего рабочего каталога

Для независимости от текущего каталога в скрипте используется переменная `$HOME`.

```bash
HOME_DIR="$HOME"
RESULT_DIR="$HOME_DIR/unit_results"
```

Все операции выполняются с абсолютными путями, поэтому скрипт может быть запущен из любого каталога.

Проверка:

```bash
cd /tmp
~/scripts/create_files.sh
```

Результаты при этом сохраняются в домашнем каталоге пользователя, от имени которого запущен скрипт.

## 8. Определение PID процесса

В macOS для просмотра процессов используется команда `ps`.

```bash
ps aux | grep create_files
```

Также можно получить PID процесса при запуске скрипта:

```bash
~/scripts/create_files.sh &
echo $!
```

В Linux для просмотра процессов используется:

```bash
systemctl status create-files.service
```

Для определения PID:

```bash
systemctl show create-files.service -p MainPID
```

Поскольку скрипт выполняется быстро и имеет тип `oneshot`, процесс может завершиться до выполнения команды просмотра PID. В таком случае `MainPID` будет равен `0`.

## 9. Просмотр записей журнала о запуске service

В macOS для просмотра системных журналов используется команда `log`.

Пример просмотра записей, связанных с заданием:

```bash
log show --last 1h --style compact \
--predicate 'eventMessage CONTAINS "com.moismmax.create-files"'
```

Также можно просмотреть журнал, записываемый самим скриптом:

```bash
cat ~/unit_results/service.log
cat ~/unit_results/service-error.log
```

В Linux для просмотра журнала службы используется:

```bash
journalctl -u create-files.service
```

Последние записи:

```bash
journalctl -u create-files.service -n 20
```

Просмотр записей в реальном времени:

```bash
journalctl -u create-files.service -f
```

## 10. Создание ошибки, определение причины и исправление

Для проверки обработки ошибок в macOS можно временно изменить путь к скрипту в plist-файле на несуществующий:

```xml
<string>/Users/moismmax/scripts/nonexistent.sh</string>
```

После изменения конфигурации задание перезагружается:

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.moismmax.create-files.plist

launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.moismmax.create-files.plist
```

Проверка состояния:

```bash
launchctl list | grep com.moismmax.create-files
```

Просмотр ошибок:

```bash
cat ~/unit_results/service-error.log
```

Для исправления необходимо вернуть правильный путь:

```xml
<string>/Users/moismmax/scripts/create_files.sh</string>
```

После исправления снова загрузить задание:

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.moismmax.create-files.plist

launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.moismmax.create-files.plist
```

Проверка результата:

```bash
ls -l ~/unit_results
```

В Linux аналогичная проверка выполняется с помощью systemd.

Для намеренного создания ошибки можно изменить путь `ExecStart` на несуществующий файл:

```ini
ExecStart=/home/scriptuser/scripts/nonexistent.sh
```

Перечитать конфигурацию:

```bash
sudo systemctl daemon-reload
```

Запустить службу:

```bash
sudo systemctl start create-files.service
```

Проверить состояние:

```bash
systemctl status create-files.service
```

Ожидаемое состояние:

```text
Active: failed (Result: exit-code)
```

Для определения причины ошибки:

```bash
journalctl -u create-files.service -n 20
```

После этого необходимо вернуть правильный путь в `ExecStart`, повторно выполнить `daemon-reload` и запустить службу:

```bash
sudo systemctl daemon-reload
sudo systemctl start create-files.service
systemctl status create-files.service
```

После исправления служба должна успешно завершить выполнение.

## Итог

В ходе работы рассмотрено создание Bash-скрипта, который формирует каталог и четыре файла с системной информацией.

Для macOS подготовлен скрипт, использующий:
- `date` для получения текущей даты;
- `uname -r` для получения версии ядра;
- `scutil --get ComputerName` для получения имени компьютера;
- `ls -la` для получения списка файлов домашнего каталога.

Для автоматического запуска в macOS используется `launchd`, а конфигурация заданий хранится в plist-файлах.

Для Linux рассмотрено создание systemd service и timer, настройка пользователя выполнения, просмотр PID, анализ журнала через `journalctl` и поиск причин состояния `failed`.

Systemd units по умолчанию выполняются от имени `root`, если не указан другой пользователь. В macOS аналогичные задачи выполняются через LaunchDaemon и LaunchAgent.

Таким образом, systemd и launchd решают похожие задачи управления процессами, но используют разные форматы конфигурации и команды управления.
