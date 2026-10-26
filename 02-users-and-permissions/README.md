# Лабораторная №2. Пользователи и права доступа

## Задача

Воспроизвести ошибку Permission denied при чтении конфигурации
служебным пользователем, определить причину и восстановить доступ
по принципу минимально необходимых прав.

## Окружение

- Ubuntu 24.04 в WSL2
- Bash
- Учебная конфигурация без секретных данных

## Подготовка

Созданы группа для чтения конфигурации и два системных пользователя:

```bash
sudo groupadd labapp-readers
sudo useradd -r -M -s /usr/sbin/nologin labapp
sudo useradd -r -M -s /usr/sbin/nologin labother
```

Ключи useradd:

- `-r` — системная учётная запись;
- `-M` — без создания домашнего каталога;
- `-s` — назначение оболочки; nologin запрещает интерактивный вход.

Пользователь labapp представляет приложение,
labother используется для проверки запрета доступа посторонним.

Создан каталог:

```bash
sudo mkdir /opt/linux-admin-lab-permissions
sudo nano /opt/linux-admin-lab-permissions/app.conf
```

Содержимое app.conf:

```ini
APP_NAME=permissions-lab
LOG_LEVEL=INFO
PORT=8080
```

Установлены владелец, группа и права:

```bash
sudo chown root:root /opt/linux-admin-lab-permissions
sudo chmod 755 /opt/linux-admin-lab-permissions
sudo chown root:labapp-readers /opt/linux-admin-lab-permissions/app.conf
sudo chmod 640 /opt/linux-admin-lab-permissions/app.conf
```

## Проблема

Попытка чтения от имени приложения:

```bash
sudo -u labapp cat /opt/linux-admin-lab-permissions/app.conf
```

Результат: Permission denied.

Ключ `-u` у sudo выбирает пользователя для выполнения команды.

## Диагностика

Проверены группы пользователя и права файла и каталогов:

```bash
id labapp
ls -l /opt/linux-admin-lab-permissions/app.conf
ls -ld /opt /opt/linux-admin-lab-permissions
```

У ls: `-l` — подробный вывод, `-d` — сам каталог вместо его содержимого.

Установлено:

- файл принадлежит root:labapp-readers и имеет права 640;
- чтение разрешено владельцу и группе;
- labapp первоначально не состоял в labapp-readers;
- права каталогов 755 разрешают остальным пользователям проход через них.

Для чтения файла необходимы право x на каталогах пути
и право r на самом файле.

## Исправление

Пользователь добавлен в дополнительную группу:

```bash
sudo usermod -aG labapp-readers labapp
```

- `-a` — добавить, сохранив прежние дополнительные группы;
- `-G` — указать дополнительные группы.

Владелец и права файла при исправлении не изменялись.

## Проверка результата

Проверено членство в группе:

```bash
id labapp
getent group labapp-readers
```

Повторное чтение от имени labapp:

```bash
sudo -u labapp cat /opt/linux-admin-lab-permissions/app.conf
```

Результат: содержимое конфигурации выведено.

Проверка постороннего пользователя:

```bash
sudo -u labother cat /opt/linux-admin-lab-permissions/app.conf
```

Результат: Permission denied.

Проверка доступности записи без изменения файла:

```bash
sudo -u labapp test -w /opt/linux-admin-lab-permissions/app.conf
echo $?
```

У test параметр `-w` проверяет доступность записи.
`$?` содержит код завершения предыдущей команды.

Получен код 1: условие ложно, запись недоступна.

| Проверка | Результат |
|---|---|
| labapp читает конфигурацию | Успешно |
| labapp может записывать в файл | Нет |
| labother читает конфигурацию | Доступ запрещён |

## Материалы

- `reports/after.txt` — сохранённый вывод проверок после исправления;
- `reports/analysis.txt` — разбор причины и решения.

## Вывод

Причиной отказа было отсутствие labapp в группе чтения конфигурации.

После добавления в labapp-readers пользователь получил необходимое
чтение. Запись осталась запрещена, посторонний пользователь доступа
к содержимому не получил.

Решение соответствует принципу Least Privilege —
минимально необходимых прав.
