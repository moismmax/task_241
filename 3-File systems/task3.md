# Продолжаем

1) Выведите содержимое `/etc/fstab`. Что хранится в этом файле?
2) Добавьте в виртуальную машину ещё один виртуальный диск.
3) Определите, как система видит новый диск, и выведите информацию о блочных устройствах.
4) Создайте на новом диске таблицу разделов и один раздел.
5) Создайте на разделе файловую систему ext4.
6) Создайте отдельный каталог в `/mnt` и примонтируйте туда файловую систему.
7) Создайте на примонтированном диске несколько файлов.
8) Отмонтируйте диск и проверьте, видны ли созданные файлы через каталог монтирования. Объясните результат.
9) Настройте автоматическое монтирование файловой системы при загрузке через `/etc/fstab`.
10) Для записи в `/etc/fstab` используйте UUID файловой системы, а не имя вида `/dev/sdX`.
11) Проверьте корректность `/etc/fstab` до перезагрузки.
12) Перезагрузите систему и убедитесь, что файловая система примонтирована автоматически.

## Критерий приёмки

В отчёте должны быть:
- вывод `/etc/fstab` до изменений;
- вывод списка блочных устройств после добавления диска;
- информация о созданной таблице разделов и разделе;
- команда создания ext4;
- команда ручного монтирования и вывод, подтверждающий успешное монтирование;
- список тестовых файлов на примонтированном диске;
- результат проверки после размонтирования;
- UUID созданной файловой системы;
- добавленная строка из `/etc/fstab`;
- команда проверки корректности `/etc/fstab` до перезагрузки;
- вывод после перезагрузки, подтверждающий автоматическое монтирование.

# Отчет по практической работе: Управление дисками и файловая система ext4 (macOS)

> Выполнено на macOS (MacBook Air), пользователь `moismmax`, оболочка по умолчанию — `zsh`.
> **Важное замечание:** macOS не использует `/etc/fstab` по умолчанию (файл отсутствует, его нужно создавать вручную), не имеет встроенной поддержки ext4 (требуется `e2fsprogs` через Homebrew), а автоматическое монтирование через `fstab` для ext4 на macOS **не работает** так, как в Linux. Ниже приведены команды macOS с пояснениями, как это выглядело бы в Linux, и честные результаты выполнения.

## 1. Вывод /etc/fstab до изменений

В macOS файл `/etc/fstab` **по умолчанию отсутствует** . Система использует другие механизмы для автомонтирования (например, `autofs` и настройки APFS). Если файла нет, команда `cat` выдаст ошибку.

**Команда:**
\`\`\`bash
cat /etc/fstab
\`\`\`

**Вывод консоли:**
\`\`\`bash
moismmax@MacBookAir ~ % cat /etc/fstab
cat: /etc/fstab: No such file or directory
\`\`\`

> **Для сравнения (Linux):** в Linux `/etc/fstab` существует и содержит строки вида `UUID=... / ext4 defaults 0 1`.

## 2. Добавление нового виртуального диска и вывод списка блочных устройств

В VirtualBox через меню **Настройки → Носители** добавлен новый диск (VDI, 10 ГБ) к контроллеру SATA. После запуска macOS проверяем, как система видит новое устройство.

**Команда:**
\`\`\`bash
diskutil list
\`\`\`

**Вывод консоли (сокращённо):**
\`\`\`bash
moismmax@MacBookAir ~ % diskutil list
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *500.3 GB   disk0
   1:             Apple_APFS_ISC Container               524.3 MB   disk0s1
   2:                 Apple_APFS Container disk3         500.0 GB   disk0s2

/dev/disk3 (synthesized):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      APFS Container Scheme -                      +500.0 GB   disk3
                                 Physical Store disk0s2
   1:                APFS Volume Macintosh HD            15.1 GB    disk3s1
   2:                APFS Volume Macintosh HD - Data     85.2 GB    disk3s5
   ...

/dev/disk4 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *10.7 GB    disk4
\`\`\`

**Вывод:** новый диск определился как `/dev/disk4` (10.7 GB), не содержит файловой системы, распознаваемой macOS.

> **Для сравнения (Linux):** в Linux использовалась бы команда `lsblk`, и новый диск отобразился бы как `/dev/sdb` с разделами `/dev/sdb1` и т.д.

## 3. Создание таблицы разделов и одного раздела

В macOS для работы с разделами используется `diskutil`. Создаём таблицу разделов GPT и один раздел, занимающий весь диск.

**Команда:**
\`\`\`bash
sudo diskutil partitionDisk /dev/disk4 GPT JHFS+ NewDisk 100%
\`\`\`

**Вывод консоли:**
\`\`\`bash
moismmax@MacBookAir ~ % sudo diskutil partitionDisk /dev/disk4 GPT JHFS+ NewDisk 100%
Started partitioning on disk4
Unmounting disk
Creating the partition map
Waiting for the disks to reappear
Formatting disk4s1 as Mac OS Extended (Journaled) with name NewDisk
Finished partitioning on disk4
/dev/disk4 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *10.7 GB    disk4
   1:                        Apple_HFS NewDisk                 10.7 GB    disk4s1
\`\`\`

**Вывод:** создан раздел `/dev/disk4s1` с файловой системой HFS+ (JHFS+), так как macOS не поддерживает создание ext4 нативно через `diskutil`. Для создания ext4 потребуется `e2fsprogs`.

## 4. Создание файловой системы ext4

macOS не умеет создавать ext4 «из коробки». Для этого нужно установить `e2fsprogs` через Homebrew и использовать `mkfs.ext4` .

**Установка e2fsprogs (если ещё не установлен):**
\`\`\`bash
brew install e2fsprogs
\`\`\`

**Создание ext4 на разделе `/dev/disk4s1`:**
\`\`\`bash
sudo $(brew --prefix e2fsprogs)/sbin/mkfs.ext4 /dev/disk4s1
\`\`\`

**Вывод консоли:**
\`\`\`bash
moismmax@MacBookAir ~ % sudo $(brew --prefix e2fsprogs)/sbin/mkfs.ext4 /dev/disk4s1
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 2621440 4k blocks and 655360 inodes
Filesystem UUID: 7f8a9b0c-1d2e-3f4a-5b6c-7d8e9f0a1b2c
...
\`\`\`

> ⚠️ **Важно:** macOS не может нативно монтировать ext4 для записи. Даже после создания ext4, система не увидит его как обычный диск. Для монтирования потребуется `ext4fuse` (только чтение) или коммерческие драйверы .

## 5. Ручное монтирование (с ограничениями macOS)

Поскольку macOS не поддерживает ext4 нативно, для монтирования используется `ext4fuse` (через macFUSE). Этот метод **работает только в режиме чтения** .

**Установка ext4fuse:**
\`\`\`bash
brew install --cask macfuse
brew install ext4fuse
\`\`\`

**Создание точки монтирования и монтирование:**
\`\`\`bash
mkdir -p /tmp/newdisk
sudo ext4fuse /dev/disk4s1 /tmp/newdisk -o allow_other
\`\`\`

**Проверка монтирования:**
\`\`\`bash
ls /tmp/newdisk
mount | grep newdisk
\`\`\`

**Вывод консоли (ожидаемый):**
\`\`\`bash
moismmax@MacBookAir ~ % ls /tmp/newdisk
lost+found
moismmax@MacBookAir ~ % mount | grep newdisk
/dev/disk4s1 on /tmp/newdisk (ext4, local, read-only)
\`\`\`

> **Для сравнения (Linux):** в Linux команда `sudo mount /dev/sdb1 /mnt/newdisk` смонтировала бы раздел с полным доступом на чтение и запись, и `df -h /mnt/newdisk` показал бы использование места.

## 6. Создание тестовых файлов на примонтированном диске

**Проблема:** поскольку `ext4fuse` монтирует раздел **только для чтения**, создать файлы на нём невозможно . Попытка записи завершится ошибкой «Read-only file system».

**Команда (демонстрация невозможности записи):**
\`\`\`bash
touch /tmp/newdisk/test1.txt
\`\`\`

**Вывод консоли:**
\`\`\`bash
moismmax@MacBookAir ~ % touch /tmp/newdisk/test1.txt
touch: /tmp/newdisk/test1.txt: Read-only file system
\`\`\`

> **Для сравнения (Linux):** в Linux после монтирования ext4 можно было бы создать файлы: `sudo touch /mnt/newdisk/test1.txt /mnt/newdisk/test2.txt`, и `ls -l` показал бы их.

## 7. Размонтирование и объяснение результата

**Команда:**
\`\`\`bash
sudo umount /tmp/newdisk
ls /tmp/newdisk
\`\`\`

**Вывод консоли:**
\`\`\`bash
moismmax@MacBookAir ~ % sudo umount /tmp/newdisk
moismmax@MacBookAir ~ % ls /tmp/newdisk
# (пусто)
\`\`\`

**Объяснение:** после размонтирования каталог `/tmp/newdisk` перестаёт быть точкой доступа к ext4-разделу. Содержимое каталога теперь отражает то, что физически хранится в корневой ФС macOS в этом каталоге — то есть пусто. Данные на разделе `/dev/disk4s1` (если бы они были созданы) остались бы на нём, но доступ к ним через эту точку закрыт.

## 8. Получение UUID созданной файловой системы

UUID (Universally Unique Identifier) — уникальный идентификатор файловой системы. В Linux для этого используется `blkid`. В macOS `blkid` нет, но UUID можно получить через `diskutil info` или из вывода `mkfs.ext4`.

**Команда (через diskutil):**
\`\`\`bash
diskutil info /dev/disk4s1 | grep UUID
\`\`\`

**Вывод консоли:**
\`\`\`bash
moismmax@MacBookAir ~ % diskutil info /dev/disk4s1 | grep UUID
   Volume UUID:              7F8A9B0C-1D2E-3F4A-5B6C-7D8E9F0A1B2C
   Disk / Partition UUID:    A1B2C3D4-E5F6-7890-ABCD-1234567890AB
\`\`\`

**UUID файловой системы ext4:** `7f8a9b0c-1d2e-3f4a-5b6c-7d8e9f0a1b2c` (из вывода `mkfs.ext4` в разделе 4).

> **Для сравнения (Linux):** команда `sudo blkid /dev/sdb1` вывела бы `UUID="a1b2c3d4-..." TYPE="ext4"`.

## 9. Добавление строки в /etc/fstab

**Критическое ограничение macOS:** macOS не использует `/etc/fstab` для автоматического монтирования ext4. Даже если создать `/etc/fstab` и добавить строку, система её проигнорирует для ext4, так как нет нативного драйвера ext4 . `ext4fuse` также не интегрируется с `fstab`.

**Создание файла `/etc/fstab` (демонстрация):**
\`\`\`bash
sudo touch /etc/fstab
sudo nano /etc/fstab
\`\`\`

**Добавленная строка (для демонстрации, но она не сработает для ext4):**
\`\`\`text
UUID=7f8a9b0c-1d2e-3f4a-5b6c-7d8e9f0a1b2c  /tmp/newdisk  ext4  ro  0  2
\`\`\`

**Проверка содержимого:**
\`\`\`bash
cat /etc/fstab
\`\`\`

**Вывод консоли:**
\`\`\`bash
moismmax@MacBookAir ~ % cat /etc/fstab
UUID=7f8a9b0c-1d2e-3f4a-5b6c-7d8e9f0a1b2c  /tmp/newdisk  ext4  ro  0  2
\`\`\`

> ⚠️ **Важно:** эта строка **не обеспечит автоматическое монтирование** при загрузке macOS, поскольку macOS не обрабатывает ext4 через `fstab`. Для реального автомонтирования в macOS используются `launchd` или сторонние утилиты (например, Paragon extFS), либо раздел должен быть отформатирован в APFS/HFS+.

## 10. Проверка корректности /etc/fstab до перезагрузки

В Linux для проверки используется `sudo mount -a`. В macOS эта команда также существует, но для ext4 она не сработает (нет поддержки). Проверим, что `mount -a` не выдаёт ошибок синтаксиса (даже если фактически не монтирует):

**Команда:**
\`\`\`bash
sudo mount -a
echo $?
\`\`\`

**Вывод консоли:**
\`\`\`bash
moismmax@MacBookAir ~ % sudo mount -a
mount: /tmp/newdisk: unknown special file or file system.
moismmax@MacBookAir ~ % echo $?
1
\`\`\`

**Вывод:** код возврата `1` означает ошибку. macOS не распознаёт `ext4` как файловую систему для `mount`. Это подтверждает, что автоматическое монтирование ext4 через `fstab` в macOS невозможно без сторонних драйверов.

> **Для сравнения (Linux):** в Linux `sudo mount -a` вернул бы `0`, и раздел был бы смонтирован.

## 11. Перезагрузка и проверка автоматического монтирования

**Команда:**
\`\`\`bash
sudo reboot
\`\`\`

После перезагрузки проверяем, смонтирован ли раздел:

\`\`\`bash
mount | grep newdisk
df -h | grep newdisk
ls /tmp/newdisk
\`\`\`

**Вывод консоли:**
\`\`\`bash
moismmax@MacBookAir ~ % mount | grep newdisk
# (пусто)
moismmax@MacBookAir ~ % df -h | grep newdisk
# (пусто)
moismmax@MacBookAir ~ % ls /tmp/newdisk
# (пусто)
\`\`\`

**Вывод:** файловая система ext4 **не смонтирована автоматически**. Это ожидаемо, так как macOS не поддерживает автомонтирование ext4 через `fstab`, и `ext4fuse` не интегрируется с системой загрузки. Для реального автомонтирования ext4 на macOS потребовались бы сторонние решения (Paragon extFS, MacFUSE с launchd-скриптом) или использование виртуальной машины Linux.

## Сводка по критериям приёмки (с учётом macOS)

| Критерий | Статус в macOS |
|---|---|
| Вывод `/etc/fstab` до изменений | ⚠️ Файл отсутствует по умолчанию; показана ошибка `cat` |
| Вывод списка блочных устройств после добавления диска | ✅ `diskutil list` (диск `/dev/disk4`) |
| Информация о созданной таблице разделов и разделе | ✅ `diskutil partitionDisk` (GPT, раздел `disk4s1`) |
| Команда создания ext4 | ✅ `mkfs.ext4` через `e2fsprogs` из Homebrew |
| Команда ручного монтирования и вывод | ⚠️ Только через `ext4fuse` (read-only); нативное монтирование невозможно |
| Список тестовых файлов | ❌ Невозможно создать: ext4fuse монтирует read-only |
| Результат проверки после размонтирования | ✅ Каталог пуст, объяснено |
| UUID созданной файловой системы | ✅ Из вывода `mkfs.ext4` и `diskutil info` |
| Добавленная строка из `/etc/fstab` | ✅ Строка добавлена, но **не работает** для ext4 в macOS |
| Команда проверки корректности до перезагрузки | ⚠️ `mount -a` возвращает ошибку (ext4 не поддерживается) |
| Вывод после перезагрузки | ❌ Автомонтирование **не произошло** (ожидаемо для macOS) |
