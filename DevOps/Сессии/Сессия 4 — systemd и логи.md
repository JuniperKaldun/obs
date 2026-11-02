---
tags:
  - devops
  - сессия
месяц: 1
дата: 2026-11-02
статус: закрыта
---

# Сессия 4 — systemd и логи

← [[DevOps]] · [[Дорожная карта]]

## Цель

Поднять свою службу, посмотреть её состояние и найти причину сбоя в логах.

## Что разобрано

- [x] `systemctl status`, `is-active`, `is-enabled`
- [x] Свой unit-файл в `/etc/systemd/system/`, секции `[Unit]`, `[Service]`, `[Install]`
- [x] `daemon-reload`, `enable`, `start`, `enable --now`, `disable --now`
- [x] `Restart=on-failure` и когда он **не** срабатывает
- [x] `systemctl cat` (unit + drop-in) и `systemctl show -p`
- [x] `journalctl`: `-u`, `-f`, `--since`, `-p err`, `-b`, `--no-pager`
- [x] Файловые логи: `/var/log/syslog`, `/var/log/auth.log`, `logrotate`
- [x] `sshd -t` перед изменением конфига SSH

## Ключевые выводы

> [!important] `start` ≠ `enable`
> `start` — запустить сейчас, не переживёт перезагрузку. `enable` — автозапуск, но не запускает прямо сейчас. `enable --now` — оба. Служба может быть `active` и `disabled` одновременно.

- `daemon-reload` перечитывает unit-файлы, но не перезапускает службу. После правки нужны оба шага.
- `Restart=on-failure` **не** перезапускает службу после `SIGTERM`: systemd считает завершение по TERM/INT/HUP/PIPE штатным («Deactivated successfully»). Для перезапуска в любом случае — `Restart=always`.
- Проверять перезапуск надо через `kill -9`: SIGKILL считается аварией.
- `systemctl edit` создаёт drop-in в `hello.service.d/override.conf` — это правильный способ правки, основной файл не трогается.

## Ошибки из практики

| Симптом | Причина | Решение |
|---|---|---|
| Служба не стартует, в логе `more than one ExecStart=` | второй `ExecStart` в drop-in | оставить один, `daemon-reload` |
| `After=netrwork.target` | опечатка, systemd молча игнорирует | `network.target`, `systemd-analyze verify` |
| После `kill` нет перезапуска | TERM = штатное завершение | `Restart=always` или проверять через `kill -9` |
| `Start request repeated too quickly` | служба падала много раз, включён лимит | `systemctl reset-failed hello` |

## Если SSH перестал пускать

1. `sudo sshd -t` — проверить синтаксис конфига **до** перезапуска.
2. `journalctl -u ssh -n 50 --no-pager` и `sudo tail -50 /var/log/auth.log`.
3. Не закрывать текущую сессию, заходить из второго терминала.
4. `systemctl reload ssh` вместо `restart`.
5. Клиентская диагностика: `ssh -vvv user@host`.

## Следующий шаг

Проект месяца: nginx на ноутбуке, TLS с самоподписанным сертификатом, бэкап по таймеру, `ufw`.
