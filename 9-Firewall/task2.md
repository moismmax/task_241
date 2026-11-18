# Открываем firewalld

1) Установите и запустите `firewalld`.
2) Определите активную зону и интерфейсы, относящиеся к ней.
3) Проверьте возможность подключения по SSH.
4) Если SSH не разрешён — добавьте соответствующий сервис в нужную зону.
5) Выведите список разрешённых сервисов и портов с помощью `firewall-cmd`.
6) Можно ли добавлять доступ по названию сервиса, а не только номеру порта?
7) На вашей локальной виртуальной машине подключитесь к Samba-серверу из предыдущего задания.
8) Если firewall блокирует Samba — разрешите нужный сервис.
9) Добавьте один тестовый TCP-порт по номеру, покажите его в конфигурации и удалите.
10) Сделайте требуемые изменения постоянными.
11) Перезагрузите VM или firewalld и убедитесь, что постоянная конфигурация сохранилась.
12) Определите, какой backend использует firewalld в вашей системе.

## Критерий приёмки

В отчёте должны быть:
- активная зона firewalld;
- интерфейс, привязанный к этой зоне;
- список разрешённых сервисов и портов до изменений;
- команда добавления SSH-сервиса;
- команда добавления Samba-сервиса;
- пример добавления TCP-порта по номеру;
- вывод runtime-конфигурации;
- вывод permanent-конфигурации;
- команда применения постоянных изменений;
- вывод после reload или reboot, подтверждающий сохранение правил;
- информация о backend, используемом firewalld.


# Открываем firewalld

## Особенности выполнения на macOS

`firewalld` — служба управления межсетевым экраном Linux. В macOS она штатно не поддерживается.

На моём компьютере используется macOS, пользователь `moismmax`, оболочка `zsh`. Поэтому команды `firewall-cmd`, `systemctl` и установка `firewalld` непосредственно на Mac неприменимы.

В macOS для управления сетевым экраном используется встроенный механизм PF. Однако он не является полной заменой `firewalld`: в частности, у него нет зон и сервисов `firewall-cmd`.

Для выполнения практической работы с `firewalld` необходима Linux-виртуальная машина. Ниже приведены команды для неё, а также соответствующие проверки macOS.

## 1. Установка и запуск firewalld

### Linux (Fedora, Rocky Linux, AlmaLinux)

Установка:

```bash
sudo dnf install firewalld -y
```

Запуск службы:

```bash
sudo systemctl start firewalld
```

Добавление в автозагрузку:

```bash
sudo systemctl enable firewalld
```

Проверка состояния:

```bash
sudo systemctl status firewalld
```

Для Debian и Ubuntu:

```bash
sudo apt update
sudo apt install firewalld -y
sudo systemctl enable --now firewalld
```

### macOS

Проверить встроенный механизм PF:

```bash
sudo pfctl -s info
```

Проверить состояние системного межсетевого экрана приложений:

```bash
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
```

Эти механизмы не устанавливают и не запускают `firewalld`.

## 2. Определение активной зоны и интерфейсов

На Linux посмотреть активные зоны:

```bash
sudo firewall-cmd --get-active-zones
```

Посмотреть все доступные зоны:

```bash
sudo firewall-cmd --get-zones
```

Посмотреть интерфейсы, относящиеся к активной зоне:

```bash
sudo firewall-cmd --get-active-zones
```

Посмотреть зону конкретного интерфейса:

```bash
sudo firewall-cmd --get-zone-of-interface=eth0
```

Имя интерфейса `eth0` приведено как пример. Реальное имя можно узнать командой:

```bash
ip -br link
```

### macOS

Посмотреть сетевые интерфейсы:

```bash
ifconfig
```

Посмотреть доступные сетевые службы:

```bash
networksetup -listallnetworkservices
```

Посмотреть соответствие сетевых служб интерфейсам:

```bash
networksetup -listnetworkserviceorder
```

## 3. Проверка возможности подключения по SSH

На Linux посмотреть, разрешён ли SSH в активной зоне:

```bash
sudo firewall-cmd --list-services
```

Проверить порт SSH:

```bash
sudo firewall-cmd --query-port=22/tcp
```

Проверить состояние SSH-службы:

```bash
sudo systemctl status sshd
```

На Debian и Ubuntu служба может называться `ssh`:

```bash
sudo systemctl status ssh
```

Проверить доступность порта с другого компьютера:

```bash
nc -vz SERVER_IP 22
```

## 4. Добавление SSH-сервиса в нужную зону

Если SSH не разрешён, добавляем его в активную зону.

Посмотреть имя активной зоны:

```bash
sudo firewall-cmd --get-active-zones
```

Добавить SSH в runtime-конфигурацию:

```bash
sudo firewall-cmd --zone=public --add-service=ssh
```

Если необходимо сохранить правило после перезагрузки:

```bash
sudo firewall-cmd --permanent --zone=public --add-service=ssh
```

Здесь `public` — пример зоны. Нужно использовать фактическую активную зону своей виртуальной машины.

## 5. Просмотр разрешённых сервисов и портов

Посмотреть разрешённые сервисы:

```bash
sudo firewall-cmd --list-services
```

Посмотреть разрешённые порты:

```bash
sudo firewall-cmd --list-ports
```

Посмотреть полную runtime-конфигурацию зоны:

```bash
sudo firewall-cmd --zone=public --list-all
```

Посмотреть permanent-конфигурацию:

```bash
sudo firewall-cmd --permanent --zone=public --list-all
```

## 6. Можно ли добавлять доступ по названию сервиса?

Да. `firewalld` позволяет разрешать доступ по названию заранее определённого сервиса.

Например:

```bash
sudo firewall-cmd --add-service=ssh
```

Вместо этого можно разрешить конкретный порт:

```bash
sudo firewall-cmd --add-port=22/tcp
```

Разница заключается в том, что сервис представляет собой заранее определённый набор параметров, включая протокол и порт.

Посмотреть доступные сервисы:

```bash
sudo firewall-cmd --get-services
```

## 7. Подключение к Samba-серверу

На Linux проверить наличие Samba-сервера:

```bash
sudo systemctl status smb
```

На некоторых системах служба называется `smbd`:

```bash
sudo systemctl status smbd
```

Для Samba обычно используются следующие порты:

- TCP 445 — основной SMB.
- TCP 139 — NetBIOS Session Service.
- UDP 137 — NetBIOS Name Service.
- UDP 138 — NetBIOS Datagram Service.

Если Samba работает, проверить подключение с macOS можно через Finder:

1. Открыть Finder.
2. Нажать `Command + K`.
3. Ввести адрес:

```text
smb://SERVER_IP
```

Либо проверить доступность TCP-порта:

```bash
nc -vz SERVER_IP 445
```

Фактическое подключение возможно только при наличии работающего Samba-сервера и разрешённого сетевого доступа.

## 8. Разрешение Samba в firewalld

На Linux добавить сервис Samba в runtime-конфигурацию:

```bash
sudo firewall-cmd --zone=public --add-service=samba
```

Сделать правило постоянным:

```bash
sudo firewall-cmd --permanent --zone=public --add-service=samba
```

Проверить наличие сервиса:

```bash
sudo firewall-cmd --zone=public --list-services
```

Если Samba не используется, открывать эти порты не требуется.

## 9. Добавление тестового TCP-порта

Добавим тестовый TCP-порт `8080`.

Добавление временного правила:

```bash
sudo firewall-cmd --zone=public --add-port=8080/tcp
```

Проверка:

```bash
sudo firewall-cmd --zone=public --query-port=8080/tcp
```

Посмотреть список портов:

```bash
sudo firewall-cmd --zone=public --list-ports
```

Удалить временное правило:

```bash
sudo firewall-cmd --zone=public --remove-port=8080/tcp
```

Проверить удаление:

```bash
sudo firewall-cmd --zone=public --query-port=8080/tcp
```

Если правило отсутствует, команда вернёт `no`.

## 10. Сохранение изменений

В `firewalld` существует два вида конфигурации:

- Runtime — правила, действующие в текущем сеансе работы службы.
- Permanent — правила, сохраняющиеся после перезапуска службы и системы.

Чтобы сделать правило постоянным, используется параметр `--permanent`.

Например:

```bash
sudo firewall-cmd --permanent --zone=public --add-service=ssh
```

После изменения permanent-конфигурации применить её:

```bash
sudo firewall-cmd --reload
```

Если ранее были добавлены Samba и тестовый порт, а тестовый порт уже удалён, итоговые постоянные правила могут быть следующими:

```bash
sudo firewall-cmd --permanent --zone=public --add-service=ssh
sudo firewall-cmd --permanent --zone=public --add-service=samba
```

Применить изменения:

```bash
sudo firewall-cmd --reload
```

## 11. Проверка после перезагрузки

Проверить состояние службы:

```bash
sudo systemctl status firewalld
```

Проверить активную зону:

```bash
sudo firewall-cmd --get-active-zones
```

Посмотреть runtime-конфигурацию:

```bash
sudo firewall-cmd --zone=public --list-all
```

Посмотреть permanent-конфигурацию:

```bash
sudo firewall-cmd --permanent --zone=public --list-all
```

Проверить наличие SSH:

```bash
sudo firewall-cmd --query-service=ssh
```

Проверить наличие Samba:

```bash
sudo firewall-cmd --query-service=samba
```

После перезагрузки виртуальной машины постоянные правила должны загрузиться автоматически, если служба `firewalld` включена в автозагрузку.

## 12. Определение backend firewalld

В современных версиях `firewalld` обычно используется backend `nftables`. В более старых конфигурациях может использоваться `iptables`.

Проверить backend:

```bash
sudo firewall-cmd --get-backend
```

Если команда не поддерживается установленной версией, посмотреть настройки:

```bash
grep -i '^FirewallBackend' /etc/firewalld/firewalld.conf
```

Возможные значения:

```text
FirewallBackend=nftables
```

или:

```text
FirewallBackend=iptables
```

Если параметр отсутствует, это может означать использование значения по умолчанию для установленной версии `firewalld`.

Дополнительно можно проверить наличие соответствующих инструментов:

```bash
command -v nft
command -v iptables
```

### macOS

В macOS нет `firewalld`, поэтому команда `firewall-cmd --get-backend` не работает.

Для PF можно посмотреть состояние:

```bash
sudo pfctl -s info
```

PF является отдельным механизмом фильтрации пакетов, а не backend `firewalld`.

---

# Вывод

`firewalld` — средство управления межсетевым экраном Linux. Оно позволяет организовывать правила с помощью зон, сервисов и портов.

В ходе настройки используются команды:

- `firewall-cmd --get-active-zones` — просмотр активных зон.
- `firewall-cmd --list-services` — просмотр разрешённых сервисов.
- `firewall-cmd --add-service=ssh` — разрешение SSH.
- `firewall-cmd --add-service=samba` — разрешение Samba.
- `firewall-cmd --add-port=8080/tcp` — временное открытие TCP-порта.
- `firewall-cmd --permanent` — сохранение изменений.
- `firewall-cmd --reload` — применение постоянной конфигурации.

Для постоянного хранения правил используется permanent-конфигурация. После перезагрузки службы или системы она применяется автоматически.

В macOS вместо `firewalld` используется PF и встроенный Application Firewall. Поэтому данная практическая работа с `firewall-cmd` полностью выполняется только на Linux-виртуальной машине.

**Примечание:** адреса серверов, активная зона, интерфейсы, backend и фактические результаты команд не указаны, поскольку они зависят от конкретной Linux VM. Их необходимо получить непосредственно в терминале виртуальной машины.
