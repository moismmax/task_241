# Шарим

1) Установите пакет Samba.
2) Что такое общий сетевой ресурс и зачем он может быть нужен?
3) Создайте общую папку без пароля с доступом только на чтение.
4) Создайте общую папку с аутентификацией и правами на чтение и запись.
5) Создайте общую папку с доступом для отдельной группы с полными правами.
6) Создайте общий ресурс, для которого:
   - группа `teachers` имеет чтение и запись;
   - группа `students` имеет только чтение;
   - пользователи, не входящие в эти группы, не имеют доступа.
7) Проверьте доступ к ресурсу минимум от имени трёх разных пользователей.
8) Проверьте конфигурацию Samba штатной командой до перезапуска службы.
9) Покажите, какие TCP/UDP-порты слушает Samba после запуска.

## Критерий приёмки

В отчёте должны быть:
- команда установки Samba;
- фрагменты `smb.conf` для всех созданных ресурсов;
- пути к каталогам, опубликованным по SMB;
- команды создания пользователей и групп, использованных в задании;
- результаты проверки доступа от пользователя с полными правами;
- результаты проверки доступа от пользователя только с чтением;
- результат отказа в доступе для пользователя без разрешений;
- вывод встроенной проверки конфигурации Samba;
- список слушающих портов Samba.


# Шарим

## 1. Установка пакета Samba

Samba — программный пакет, обеспечивающий совместимость с протоколом SMB/CIFS и позволяющий организовать общий доступ к файлам между компьютерами с разными операционными системами.

В macOS предусмотрена встроенная поддержка SMB. Для включения общего доступа к файлам используется раздел «Системные настройки» → «Основные» → «Общий доступ» → «Общий доступ к файлам».

Установка отдельного пакета Samba в macOS возможна через Homebrew:

```bash
brew install samba
```

В данной системе Homebrew отсутствует, поэтому установка отдельного пакета Samba не выполнялась.

В Linux установка выполняется командой:

```bash
sudo apt update
sudo apt install samba smbclient -y
```

## 2. Что такое общий сетевой ресурс и зачем он нужен?

Общий сетевой ресурс — это каталог или файл, доступный другим пользователям через локальную сеть с использованием протокола SMB.

Общие ресурсы необходимы для:
- обмена файлами между компьютерами;
- совместной работы с документами;
- централизованного хранения информации;
- разграничения прав доступа;
- организации общего хранилища данных.

В macOS общий доступ к каталогам можно настроить через встроенную функцию SMB.

## 3. Создание общей папки без пароля с доступом только на чтение

В macOS можно создать каталог и настроить права доступа к нему.

Создание каталога:

```bash
mkdir -p /Users/moismmax/Public/SharedPublic
chmod 755 /Users/moismmax/Public/SharedPublic
```

Просмотр прав:

```bash
ls -ld /Users/moismmax/Public/SharedPublic
```

Для организации гостевого доступа к ресурсу средствами Samba используется следующая конфигурация:

```ini
[public]
    path = /srv/samba/public
    browseable = yes
    guest ok = yes
    read only = yes
    public = yes
```

В macOS отдельная настройка анонимного SMB-доступа через `smb.conf` не выполняется, поскольку отдельный сервер Samba не установлен.

В Linux для создания ресурса используется каталог `/srv/samba/public`, а параметры доступа задаются в `/etc/samba/smb.conf`.

## 4. Создание общей папки с аутентификацией и правами на чтение и запись

В macOS каталог с ограниченным доступом создаётся следующими командами:

```bash
mkdir -p /Users/moismmax/Public/SharedPrivate
chmod 700 /Users/moismmax/Public/SharedPrivate
```

Проверка прав:

```bash
ls -ld /Users/moismmax/Public/SharedPrivate
```

Встроенный SMB-сервер macOS позволяет предоставлять доступ к общим папкам зарегистрированным пользователям системы.

Для Samba в Linux создаётся отдельный пользователь:

```bash
sudo useradd -M -s /usr/sbin/nologin user1
sudo smbpasswd -a user1
```

Создание каталога:

```bash
sudo mkdir -p /srv/samba/private
sudo chown user1:user1 /srv/samba/private
sudo chmod 770 /srv/samba/private
```

Конфигурация ресурса:

```ini
[private]
    path = /srv/samba/private
    browseable = yes
    guest ok = no
    valid users = user1
    read only = no
    writable = yes
    create mask = 0660
    directory mask = 0770
```

Пользователь `user1` получает доступ после ввода учётных данных.

## 5. Создание общей папки для отдельной группы с полными правами

В macOS управление локальными группами возможно с помощью команды `dseditgroup`.

Создание группы:

```bash
sudo dseditgroup -o create admins
```

Проверка группы:

```bash
dseditgroup -o read admins
```

Создание каталога:

```bash
mkdir -p /Users/moismmax/Public/SharedAdmins
chmod 770 /Users/moismmax/Public/SharedAdmins
```

В macOS права каталога задаются средствами файловой системы. Полноценная настройка Samba-ресурса с параметром `valid users = @admins` требует отдельного сервера Samba.

В Linux создание группы и пользователя выполняется следующим образом:

```bash
sudo groupadd admins
sudo useradd -M -s /usr/sbin/nologin admin1
sudo usermod -aG admins admin1
sudo smbpasswd -a admin1
```

Создание каталога и назначение прав:

```bash
sudo mkdir -p /srv/samba/admins
sudo chown root:admins /srv/samba/admins
sudo chmod 2770 /srv/samba/admins
```

Конфигурация Samba:

```ini
[admins]
    path = /srv/samba/admins
    browseable = yes
    guest ok = no
    valid users = @admins
    read only = no
    writable = yes
    force group = admins
    create mask = 0660
    directory mask = 2770
```

Участники группы `admins` получают полный доступ к общему ресурсу.

## 6. Создание общего ресурса с разными правами доступа

В macOS можно создавать локальные группы пользователей и назначать права доступа к каталогам.

Создание групп:

```bash
sudo dseditgroup -o create teachers
sudo dseditgroup -o create students
```

Проверка:

```bash
dseditgroup -o read teachers
dseditgroup -o read students
```

Создание каталога:

```bash
mkdir -p /Users/moismmax/Public/Education
chmod 755 /Users/moismmax/Public/Education
```

Для реализации разграничения доступа через Samba в Linux создаются пользователи:

```bash
sudo useradd -M -s /usr/sbin/nologin teacher1
sudo useradd -M -s /usr/sbin/nologin student1
sudo useradd -M -s /usr/sbin/nologin outsider
```

Создание групп:

```bash
sudo groupadd teachers
sudo groupadd students
```

Добавление пользователей в группы:

```bash
sudo usermod -aG teachers teacher1
sudo usermod -aG students student1
```

Добавление пользователей в базу Samba:

```bash
sudo smbpasswd -a teacher1
sudo smbpasswd -a student1
sudo smbpasswd -a outsider
```

Создание каталога:

```bash
sudo mkdir -p /srv/samba/education
sudo chown root:teachers /srv/samba/education
sudo chmod 2770 /srv/samba/education
```

Конфигурация общего ресурса:

```ini
[education]
    path = /srv/samba/education
    browseable = yes
    guest ok = no
    valid users = @teachers @students
    read only = yes
    write list = @teachers
    force group = teachers
    create mask = 0660
    directory mask = 2770
```

Распределение прав доступа:

| Группа | Чтение | Запись |
|---|---|---|
| teachers | Да | Да |
| students | Да | Нет |
| Остальные пользователи | Нет | Нет |

Таким образом, пользователи группы `teachers` могут читать и изменять файлы, пользователи `students` могут только просматривать содержимое, а остальные пользователи не имеют доступа.

## 7. Проверка доступа от имени трёх пользователей

Для проверки доступа в Linux используется утилита `smbclient`.

Проверка от имени пользователя с полными правами:

```bash
smbclient //localhost/education -U teacher1
```

Внутри клиента выполняются команды:

```text
ls
put test.txt
```

Ожидаемый результат: пользователь может просматривать содержимое каталога и загружать файлы.

Проверка от имени пользователя только с чтением:

```bash
smbclient //localhost/education -U student1
```

Команда просмотра:

```text
ls
```

Ожидаемый результат: пользователь может просматривать содержимое каталога.

Попытка записи:

```text
put test.txt
```

Ожидаемый результат: операция записи отклоняется из-за отсутствия соответствующих прав.

Проверка от имени пользователя без разрешений:

```bash
smbclient //localhost/education -U outsider
```

Ожидаемый результат: сервер отказывает в доступе, поскольку пользователь не входит в группы `teachers` и `students`.

В macOS без отдельной установки Samba данные проверки через `smbclient` выполнить нельзя.

## 8. Проверка конфигурации Samba

В Linux для проверки конфигурационного файла используется штатная утилита `testparm`.

Команда:

```bash
sudo testparm
```

Пример успешного результата:

```text
Load smb config files from /etc/samba/smb.conf
Loaded services file OK.
Server role: ROLE_STANDALONE
```

Для просмотра всех опубликованных ресурсов используется команда:

```bash
sudo testparm -s
```

Проверка выполняется до перезапуска службы Samba.

В macOS без установленного пакета Samba утилита `testparm` отсутствует. Проверить конфигурацию встроенного SMB-сервера командой `testparm` нельзя.

## 9. Запуск Samba и проверка TCP/UDP-портов

В Linux запуск и автоматическое включение службы выполняются командами:

```bash
sudo systemctl restart smbd
sudo systemctl enable smbd
```

Проверка состояния:

```bash
sudo systemctl status smbd
```

Просмотр TCP- и UDP-портов:

```bash
sudo ss -tulpn | grep -E 'smbd|nmbd'
```

Основные порты Samba:

| Протокол | Порт | Назначение |
|---|---:|---|
| TCP | 445 | SMB непосредственно поверх TCP |
| TCP | 139 | SMB через NetBIOS Session Service |
| UDP | 137 | NetBIOS Name Service |
| UDP | 138 | NetBIOS Datagram Service |

В macOS для просмотра портов встроенных сетевых служб используется команда:

```bash
sudo lsof -nP -iTCP -iUDP
```

Для проверки конкретных SMB-портов:

```bash
sudo lsof -nP -iTCP:445 -iTCP:139 -iUDP:137 -iUDP:138
```

Команда показывает процессы, использующие указанные порты. Наличие слушающего процесса зависит от состояния общего доступа к файлам.

## 10. Итог

В ходе работы рассмотрены способы организации общего доступа к файлам с использованием SMB.

На macOS рассмотрены создание общих каталогов, изменение прав доступа с помощью `chmod`, управление локальными группами через `dseditgroup`, а также просмотр сетевых портов с помощью `lsof`.

Для Linux приведены команды установки Samba, создания пользователей и групп, настройки общих ресурсов, разграничения прав доступа и проверки конфигурации с помощью `testparm`.

Рассмотрены четыре типа общих ресурсов:
- `public` — анонимный доступ только для чтения;
- `private` — доступ по логину и паролю с правами чтения и записи;
- `admins` — полный доступ для участников отдельной группы;
- `education` — чтение для группы `students`, чтение и запись для группы `teachers`, запрет доступа остальным пользователям.

Также рассмотрены способы проверки доступа от имени пользователей с разными полномочиями и основные TCP/UDP-порты, используемые Samba.
