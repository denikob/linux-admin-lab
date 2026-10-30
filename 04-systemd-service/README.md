# Лабораторная №4. Развёртывание и диагностика службы systemd

## Задача

Создать службу systemd, настроить её автозапуск, смоделировать отказ,
найти причину по журналу и восстановить работу.

## Окружение

- Windows 11
- WSL2
- Ubuntu 24.04
- Bash и systemd

## Файлы лабораторной

- [Скрипт службы](scripts/lab-heartbeat.sh)
- [Unit-файл](systemd/lab-heartbeat.service)
- [Разбор диагностики и восстановления](reports/analysis.txt)

## Результат

Служба запускает Bash-скрипт от пользователя denis.
Каждые 10 секунд скрипт записывает дату и сообщение в журнал.

Смоделирован отказ из-за неверного пути к скрипту в ExecStart.
Причина установлена по сообщению No such file or directory.
После исправления пути и перезапуска служба вернулась
в состояние active (running). Автозапуск включён — enabled.

## Установка и запуск

Команды выполняются из каталога `04-systemd-service`.

В unit-файле указаны пользователь `denis` и путь
`/home/denis/linux-admin-lab/04-systemd-service/scripts/lab-heartbeat.sh`.
При воспроизведении в другом окружении их нужно изменить.

Проверка синтаксиса скрипта:

```bash
bash -n scripts/lab-heartbeat.sh
```

Установка unit-файла, перечитывание конфигурации, запуск
и включение автозапуска:

```bash
sudo cp systemd/lab-heartbeat.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl start lab-heartbeat.service
sudo systemctl enable lab-heartbeat.service
```

Проверка состояния, автозапуска и журнала:

```bash
systemctl status lab-heartbeat.service --no-pager -l
systemctl is-enabled lab-heartbeat.service
journalctl -u lab-heartbeat.service -n 15 --no-pager
```

## Моделирование отказа и диагностика

В установленном файле `/etc/systemd/system/lab-heartbeat.service`
имя скрипта в `ExecStart` изменено на `lab-heartbeat-missing.sh`.
Затем выполнены перечитывание конфигурации и перезапуск службы.

Служба перешла в состояние `failed`.
В журнале появилось сообщение `No such file or directory`
с путём к отсутствующему скрипту.

Systemd успешно запустил `/bin/bash`, но Bash не смог открыть
указанный скрипт и завершился с кодом `127`.

## Восстановление

В установленном unit-файле восстановлен путь к `lab-heartbeat.sh`,
после чего выполнены команды:

```bash
sudo systemctl daemon-reload
sudo systemctl restart lab-heartbeat.service
```

Проверка подтвердила состояние `active (running)`
и появление новых сообщений в журнале.

## Отчёты

| Этап | Состояние службы | Журнал |
|---|---|---|
| До отказа | [Статус](reports/status-working.txt) | [Журнал](reports/journal-working.txt) |
| Отказ | [Статус](reports/status-failed.txt) | [Журнал](reports/journal-failed.txt) |
| После восстановления | [Статус](reports/status-recovered.txt) | [Журнал](reports/journal-recovered.txt) |

Подробный самостоятельный разбор: [analysis.txt](reports/analysis.txt).
