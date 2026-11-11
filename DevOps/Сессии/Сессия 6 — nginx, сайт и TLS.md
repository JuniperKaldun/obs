---
tags:
  - devops
  - сессия
месяц: 1
дата: 2026-11-12
статус: в работе
среда: ноутбук, Ubuntu 24.04
---

# Сессия 6 — nginx, статический сайт и TLS

← [[DevOps]] · [[Дорожная карта]]

## Цель

Отдавать свой сайт через nginx по HTTPS и объяснить каждый шаг: что делает nginx, что происходит при TLS-рукопожатии и почему браузер ругается на самоподписанный сертификат.

## Что понадобится

Ноутбук с Ubuntu 24.04, `sudo`, интернет для `apt`. Домен не нужен — работаем по `localhost` и `127.0.0.1`.

## Теория, минимум (30 минут, не больше)

> [!important] nginx — это не «сервер сайтов», это обработчик соединений
> nginx принимает TCP-соединение на порту, решает, что с ним делать, и отвечает. Статику он отдаёт сам из каталога на диске. Позже тем же механизмом он будет проксировать запросы в приложение — это называется reverse proxy. Один и тот же конфиг умеет и то, и другое.

> [!important] HTTPS — это HTTP внутри TLS
> Клиент открывает TCP на 443, стороны делают TLS-рукопожатие (обменялись ключами, сервер показал сертификат), и только внутри этого зашифрованного канала идёт обычный HTTP-запрос. nginx «терминирует TLS»: расшифровывает и дальше работает с обычным HTTP.

- **Сертификат** привязывает имя (`localhost`) к открытому ключу и подписан удостоверяющим центром.
- **Самоподписанный** подписан сам собой: цепочки к доверенному центру нет, поэтому браузер показывает предупреждение. Для лаборатории это нормально и ожидаемо.
- **SAN** (`subjectAltName`) — список имён, на которые сертификат действителен. Современные браузеры и curl смотрят только на SAN, поле `CN` игнорируют. Сертификат без SAN считается недействительным, даже если `CN=localhost`.

## Лаборатория

### Шаг 1. Установить и проверить, что слушает

```bash
sudo apt update && sudo apt install -y nginx
systemctl is-enabled nginx          # enabled — автозапуск
systemctl is-active nginx           # active
sudo ss -ltnp | grep -E ':(80|443)' # кто именно слушает порт
curl -I http://localhost            # Server: nginx/1.24.x
```

### Шаг 2. Свой сайт вместо дефолтного

```bash
sudo mkdir -p /var/www/devops
echo '<h1>DevOps lab</h1><p>Сайт месяца 1</p>' | sudo tee /var/www/devops/index.html
sudo chown -R www-data:www-data /var/www/devops
```

Конфиг `/etc/nginx/sites-available/devops`:

```nginx
server {
    listen 80;
    server_name localhost;
    root /var/www/devops;
    index index.html;
}
```

```bash
sudo ln -s /etc/nginx/sites-available/devops /etc/nginx/sites-enabled/devops
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t                      # ОБЯЗАТЕЛЬНО до reload
sudo systemctl reload nginx
curl http://localhost              # должен отдать ваш index.html
```

### Шаг 3. Сертификат

```bash
sudo openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
  -keyout /etc/ssl/private/devops.key \
  -out    /etc/ssl/certs/devops.crt \
  -subj "/C=RU/ST=Moscow/L=Moscow/O=DevOps Lab/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
```

Посмотреть, что получилось:

```bash
sudo openssl x509 -in /etc/ssl/certs/devops.crt -noout -subject -issuer -dates -ext subjectAltName
```

Ожидаемо: `issuer` и `subject` одинаковые (сам себя подписал), в SAN есть `localhost` и `127.0.0.1`.

### Шаг 4. Перевести сайт на 443

Дополнить `/etc/nginx/sites-available/devops`:

```nginx
server {
    listen 80;
    server_name localhost;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name localhost;
    root /var/www/devops;
    index index.html;

    ssl_certificate     /etc/ssl/certs/devops.crt;
    ssl_certificate_key /etc/ssl/private/devops.key;
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### Шаг 5. Проверки

```bash
curl -I http://localhost        # 301 → https://localhost/
curl -kI https://localhost      # 200, Server: nginx
curl -k https://localhost       # HTML вашего сайта
```

Почему `-k`: curl без него откажется, потому что сертификат самоподписан. Браузер в такой ситуации показывает «Дополнительно → Перейти». Это то самое предупреждение, и оно корректно.

Рукопожатие глазами:

```bash
openssl s_client -connect 127.0.0.1:443 -servername localhost </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

## Ловушки

| Симптом | Причина | Решение |
|---|---|---|
| `nginx: [emerg] unknown directive` | опечатка в конфиге | `sudo nginx -t` покажет строку; после установки не запускать reload |
| `403 Forbidden` | нет `index.html` или нет прав на каталог | `ls -l /var/www/devops`, `chown -R www-data:www-data` |
| `502 Bad Gateway` | проксирование на мёртвое приложение | появится в месяце 6, но причина та же: бэкенд не отвечает |
| `address already in use` | 80-й занят другим сервисом | `sudo ss -ltnp \| grep :80` и выключить лишнее |
| Сайт открывается по HTTP со старым содержимым | не сделан `reload` или правится не тот symlink | `sudo nginx -T \| grep -A5 "server {"` |
| Браузер: `NET::ERR_CERT_COMMON_NAME_INVALID` | в сертификате нет SAN | пересоздать с `-addext "subjectAltName=..."` |

## Проверка знаний

1. Чем `systemctl reload nginx` отличается от `restart`?
2. Зачем `nginx -t` и что будет, если сделать `reload` с битым конфигом?
3. Что именно проверяет браузер, когда получает сертификат?
4. Почему сертификат с `CN=localhost`, но без SAN не работает?
5. Что означает `listen 443 ssl` и почему nginx не может просто «включить HTTPS»?
6. Где смотреть, почему nginx вернул 403?

> [!success]- Ответы (открыть после попытки)
> 1. `reload` применяет конфиг без обрыва соединений (старые воркеры дорабатывают), `restart` останавливает и запускает процесс — соединения рвутся.
> 2. Проверяет синтаксис и доступность файлов; при ошибке reload не применяется, работает старый конфиг, а вы этого можете не заметить.
> 3. Срок действия, подпись цепочкой до доверенного корня, совпадение имени из адресной строки с SAN сертификата.
> 4. Потому что браузеры смотрят SAN, а `CN` исторически не учитывают — сертификат считается выданным на другие имена.
> 5. «Принимать TCP на 443 и делать TLS-рукопожатие с этим сертификатом». Без сертификата и ключа TLS не начнётся.
> 6. `/var/log/nginx/error.log` (`sudo tail -50`), плюс `journalctl -u nginx -n 50 --no-pager`.

## Критерий «готово»

- [ ] `https://localhost` отдаёт ваш `index.html` (с предупреждением браузера)
- [ ] `http://localhost` отдаёт `301` на HTTPS
- [ ] `sudo nginx -t` проходит чисто
- [ ] `sudo ss -ltnp` показывает nginx на 80 и 443
- [ ] можете, не подглядывая, рассказать путь запроса: TCP → TLS → HTTP → файл на диске

## Что записать в выводы

По 2–3 строки своими словами: что удивило, что пришлось гуглить, какая ошибка заняла больше времени. Это черновик для [[Журнал прогресса]].

## Следующий шаг

Сессия 7 — бэкап по systemd-таймеру, проверка восстановления и `ufw`. После неё месяц 1 закрывается артефактом.
