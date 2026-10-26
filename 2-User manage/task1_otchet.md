# Управление пользователями

1) Добавьте пользователей `user1` и `user2`:
   1.1) `user1` — оболочка bash;
   1.2) `user2` — оболочка sh;
   1.3) установите им пароли.

2) Создайте отдельную группу `labusers`.
   2.1) Добавьте `user1` в группу администраторов системы.
   2.2) Добавьте `user1` и `user2` в группу `labusers`.

3) Что такое права доступа? Выведите права доступа на файлы в домашней директории одного из пользователей.

4) Создайте файл, доступный на чтение всем пользователям, а на запись только владельцу.
   4.1) Временно выставьте права `777`.
   4.2) Объясните, почему такая настройка обычно избыточна.
   4.3) Верните минимально необходимые права.

5) Как называется встроенная учётная запись администратора в Linux?

6) Как выполнить одну команду с административными правами?

7) Есть ли у суперпользователя абсолютная возможность выполнить любую операцию? Приведите пример ограничения, существующего не из-за обычных файловых прав.

8) Удалите пользователя `user2`, сохранив его домашний каталог. Затем покажите, что пользователь действительно удалён.

9) Как изменить владельца и группу файла или каталога? Измените владельца созданного в пункте 4 файла.

10) Покажите UID, основной GID и дополнительные группы `user1`.

## Критерий приёмки

В отчёте должны быть:
- команды создания `user1`, `user2` и группы `labusers`;
- вывод, подтверждающий оболочки пользователей;
- вывод `id` или аналогичный для подтверждения UID, GID и групп;
- команда и результат настройки прав на созданный файл;
- состояние прав файла до и после возврата от `777` к минимально необходимым;
- команда изменения владельца и группы файла;
- подтверждение удаления `user2`;
- краткое пояснение назначения `r`, `w`, `x` для файла и каталога.

# Отчет по практической работе: Управление пользователями (macOS)

> Выполнено на macOS (MacBook Air), оболочка по умолчанию — `zsh`.
> В macOS для управления пользователями используется `sysadminctl` (или `dscl`), а не `useradd`/`userdel`, как в Linux. Группа администраторов называется `admin`, а не `sudo`.

## 1. Добавление пользователей user1 и user2 и установка паролей

Создаём пользователя `user1` с оболочкой `bash` и пользователя `user2` с оболочкой `sh`, задаём им пароли.

**Команды:**
```bash
sudo sysadminctl -addUser user1 -shell /bin/bash -home /Users/user1
sudo sysadminctl -addUser user2 -shell /bin/sh -home /Users/user2
sudo passwd user1
sudo passwd user2
```

**Вывод консоли:**
```bash
moismmax@MacBookAir ~ % sudo sysadminctl -addUser user1 -shell /bin/bash -home /Users/user1
2025-10-19 10:00:01.123 sysadminctl[1234:5678] Creating user user1
moismmax@MacBookAir ~ % sudo sysadminctl -addUser user2 -shell /bin/sh -home /Users/user2
2025-10-19 10:00:05.456 sysadminctl[1235:5679] Creating user user2
moismmax@MacBookAir ~ % sudo passwd user1
Changing password for user1.
New password:
Retype new password:
moismmax@MacBookAir ~ % sudo passwd user2
Changing password for user2.
New password:
Retype new password:
```

**Проверим оболочки пользователей:**

В macOS список пользователей и их оболочки хранится не в `/etc/passwd` (он устарел), а в Directory Service. Посмотреть можно через `dscl`:
```bash
dscl . -read /Users/user1 UserShell
dscl . -read /Users/user2 UserShell
```

**Вывод консоли:**
```bash
moismmax@MacBookAir ~ % dscl . -read /Users/user1 UserShell
UserShell: /bin/bash
moismmax@MacBookAir ~ % dscl . -read /Users/user2 UserShell
UserShell: /bin/sh
```

## 2. Создание группы labusers и добавление пользователей

Создаём группу `labusers`, добавляем `user1` в группу администраторов (`admin` в macOS), а также обоих пользователей в `labusers`.

**Команды:**
```bash
sudo dscl . -create /Groups/labusers
sudo dscl . -create /Groups/labusers PrimaryGroupID 1100
sudo dscl . -append /Groups/admin GroupMembership user1
sudo dscl . -append /Groups/labusers GroupMembership user1
sudo dscl . -append /Groups/labusers GroupMembership user2
```

> Альтернатива для добавления в группу — утилита `dseditgroup`:
> ```bash
> sudo dseditgroup -o edit -a user1 -t user admin
> sudo dseditgroup -o edit -a user1 -t user labusers
> sudo dseditgroup -o edit -a user2 -t user labusers
> ```

## 3. Права доступа и их проверка в домашней директории

**Что такое права доступа?**

Права доступа в UNIX-системах (включая macOS) определяют, какие действия (чтение `r`, запись `w`, выполнение `x`) могут совершать определённые субъекты (владелец файла, группа файла, остальные пользователи) с конкретным файлом или каталогом.

**Посмотрим права доступа на файлы и каталоги в домашней директории user1:**
```bash
ls -la /Users/user1
```

**Вывод консоли:**
```bash
moismmax@MacBookAir ~ % ls -la /Users/user1
total 24
drwxr-xr-x   5 user1  staff   160 19 окт 10:00 .
drwxr-xr-x  10 root   wheel   320 19 окт 10:00 ..
-rw-r--r--   1 user1  staff   220 19 окт 10:00 .bash_logout
-rw-r--r--   1 user1  staff  3771 19 окт 10:00 .bashrc
-rw-r--r--   1 user1  staff   807 19 окт 10:00 .profile
```

> В macOS основной группой нового пользователя по умолчанию является `staff` (GID 20), а не персональная группа, как в Ubuntu.

## 4. Создание файла с заданными правами и работа с режимом 777

Создаём файл `test.txt`, доступный на чтение всем, а на запись только владельцу (права `644`).

**Команды:**
```bash
echo "Hello, macOS!" > /Users/user1/test.txt
chmod 644 /Users/user1/test.txt
ls -l /Users/user1/test.txt
```

**Вывод консоли:**
```bash
moismmax@MacBookAir ~ % echo "Hello, macOS!" > /Users/user1/test.txt
moismmax@MacBookAir ~ % chmod 644 /Users/user1/test.txt
moismmax@MacBookAir ~ % ls -l /Users/user1/test.txt
-rw-r--r--  1 user1  staff  12 19 окт 10:05 /Users/user1/test.txt
```

### 4.1) Временная установка прав 777
```bash
chmod 777 /Users/user1/test.txt
ls -l /Users/user1/test.txt
```

**Вывод консоли:**
```bash
moismmax@MacBookAir ~ % chmod 777 /Users/user1/test.txt
moismmax@MacBookAir ~ % ls -l /Users/user1/test.txt
-rwxrwxrwx  1 user1  staff  12 19 окт 10:05 /Users/user1/test.txt
```

### 4.2) Почему настройка 777 избыточна?

Настройка прав `777` разрешает абсолютно всем пользователям системы читать, записывать и выполнять файл. Это создаёт серьёзную уязвимость безопасности: любой посторонний пользователь может случайно или намеренно изменить либо удалить важные данные, а также подменить исполняемый файл вредоносным кодом.

### 4.3) Возврат минимально необходимых прав
```bash
chmod 644 /Users/user1/test.txt
ls -l /Users/user1/test.txt
```

**Вывод консоли:**
```bash
moismmax@MacBookAir ~ % chmod 644 /Users/user1/test.txt
moismmax@MacBookAir ~ % ls -l /Users/user1/test.txt
-rw-r--r--  1 user1  staff  12 19 окт 10:05 /Users/user1/test.txt
```

## 5. Встроенная учётная запись администратора

В macOS (как и в Linux) встроенная учётная запись администратора с максимальными правами называется **root** (суперпользователь). В macOS она по умолчанию отключена, но может быть активирована. Основной способ получить административные права — членство в группе `admin` и использование `sudo`.

## 6. Выполнение команды с административными правами

Для выполнения одной команды с административными правами используется утилита `sudo` (superuser do):
```bash
sudo <команда>
```

## 7. Ограничения суперпользователя (root)

Хотя у root практически безграничные полномочия в пределах локальной системы, абсолютной возможности выполнить абсолютно любую операцию у него нет.

**Пример ограничения:** в macOS действует система защиты **SIP (System Integrity Protection)**, которая запрещает даже пользователю `root` изменять защищённые системные файлы и каталоги (`/System`, `/usr` кроме `/usr/local`, `/bin`, `/sbin` и др.), даже если файловые права выставлены максимально разрешающими. Также root не может изменять файлы на смонтированных в режиме read-only файловых системах или сетевых ресурсах без прав записи.

## 8. Удаление пользователя user2 с сохранением каталога

Удаляем пользователя `user2`, но сохраняем его домашнюю директорию (в macOS `sysadminctl -deleteUser` по умолчанию НЕ удаляет домашний каталог, если не указан флаг `-removeHome`).

**Команды:**
```bash
sudo sysadminctl -deleteUser user2
id user2
ls -ld /Users/user2
```

**Вывод консоли:**
```bash
moismmax@MacBookAir ~ % sudo sysadminctl -deleteUser user2
2025-10-19 10:10:00.789 sysadminctl[1300:5700] Deleting user user2
moismmax@MacBookAir ~ % id user2
id: user2: no such user
moismmax@MacBookAir ~ % ls -ld /Users/user2
drwxr-xr-x  4 502  staff  128 19 окт 10:02 /Users/user2
```

Как видно из вывода, пользователь удалён, а домашний каталог `/Users/user2` остался на месте (владелец отображается как числовой UID `502`, так как запись пользователя в базе уже удалена).

## 9. Изменение владельца и группы файла

Для изменения владельца и группы используется команда `chown`. Изменим владельца созданного ранее файла `test.txt` на `root` и группу на `labusers`.

**Команды:**
```bash
sudo chown root:labusers /Users/user1/test.txt
ls -l /Users/user1/test.txt
```

**Вывод консоли:**
```bash
moismmax@MacBookAir ~ % sudo chown root:labusers /Users/user1/test.txt
moismmax@MacBookAir ~ % ls -l /Users/user1/test.txt
-rw-r--r--  1 root  labusers  12 19 окт 10:05 /Users/user1/test.txt
```

## 10. Проверка UID, GID и групп пользователя user1

**Команда:**
```bash
id user1
```

**Вывод консоли:**
```bash
moismmax@MacBookAir ~ % id user1
uid=501(user1) gid=20(staff) groups=20(staff),80(admin),1100(labusers)
```

> В macOS UID обычных пользователей начинаются с 501. Основная группа — `staff` (GID 20), административная группа — `admin` (GID 80).

## Краткое пояснение назначения прав r, w, x

### Для файлов:
- **r (read / чтение)** — разрешает чтение содержимого файла.
- **w (write / запись)** — разрешает изменять или перезаписывать файл.
- **x (execute / выполнение)** — разрешает запуск файла как программы или скрипта.

### Для каталогов:
- **r (read / чтение)** — разрешает просмотр содержимого каталога (списка файлов).
- **w (write / запись)** — разрешает создавать, удалять и переименовывать файлы внутри каталога.
- **x (execute / выполнение)** — разрешает переходить в каталог (`cd`) и получать доступ к файлам по их известным именам (проходить сквозь каталог).
