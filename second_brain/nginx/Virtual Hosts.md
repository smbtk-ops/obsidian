## 1. Роль Nginx

Nginx выполняет следующие функции:

- пограничная точка входа трафика в приватную сеть проекта
- терминация TLS (ресурсоёмкая для CPU операция)
- проксирование запросов на backend-серверы (серверы приложений)
- модификация и проверка запросов (basic auth, добавление `X-Forwarded-For` и др.)
- балансировка нагрузки с исключением недоступных серверов

---

## 2. Установка

### 2.1. Из системного пакетного менеджера

```bash
# Debian/Ubuntu
apt install nginx

# CentOS/RHEL
yum install nginx
```

Версия в системных репозиториях часто устаревшая — рекомендуется использовать [официальный репозиторий nginx](https://nginx.org/en/linux_packages.html).

### 2.2. Из официального репозитория (Debian/Ubuntu)

```bash
apt install curl gnupg2 ca-certificates lsb-release
echo "deb http://nginx.org/packages/ubuntu $(lsb_release -cs) nginx" \
  > /etc/apt/sources.list.d/nginx.list
curl -fsSL https://nginx.org/keys/nginx_signing.key | apt-key add -
apt update && apt install nginx
```

### 2.3. Сборка из исходников

Необходима при подключении сторонних модулей. Исходники доступны на [nginx.org](https://nginx.org/en/download.html).

---

## 3. Структура конфигурации

### 3.1. Расположение файлов

В зависимости от дистрибутива и способа установки используются два подхода:

| Подход | Каталоги | Управление |
|--------|----------|------------|
| sites-available / sites-enabled | `/etc/nginx/sites-available/`, `/etc/nginx/sites-enabled/` | Симлинки для включения/отключения конфигов |
| conf.d | `/etc/nginx/conf.d/` | Расширение `.conf` обязательно — оно включает/отключает конфиг |

Всегда проверять `nginx.conf`, чтобы понять из каких директорий подключается дополнительная конфигурация:

```bash
grep -n 'include' /etc/nginx/nginx.conf
```

### 3.2. Именование файлов

Рекомендуется называть файлы по DNS-имени и протоколу:

```
app.example.com.conf           # общий конфиг
app.example.com.http.conf      # только HTTP
app.example.com.https.conf     # только HTTPS
```

### 3.3. Секции основного конфига

```nginx
### main — базовые параметры процесса nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

### events — параметры обработки подключений
events {
    worker_connections 768;
}

### http — основная секция для web-трафика
http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    gzip on;

    include /etc/nginx/conf.d/*.conf;
}
```

| Секция | Назначение |
|--------|-----------|
| `main` | Пользователь, pid, воркеры, подключение модулей |
| `events` | Библиотека обработки соединений, `worker_connections` |
| `http` | MIME types, gzip, SSL, логи, подключение virtual hosts |
| `http { server }` | Описание виртуальных хостов |
| `mail` | Проксирование почтовых протоколов (imap, smtp, pop3) |

---

## 4. Основные директивы

### 4.1. include

Подключает содержимое других файлов в место вызова директивы. Nginx компилирует конфигурацию, подставляя содержимое подключаемых файлов и применяя директивы последовательно — порядок важен.

При использовании wildcard (`*`) файлы подключаются в алфавитном порядке. Чтобы файл обработался первым, можно использовать числовой префикс в имени:

```
00-defaults.conf
10-app.example.com.conf
20-api.example.com.conf
```

### 4.2. server

Определяет виртуальный хост — позволяет обрабатывать несколько доменов на одном IP-адресе. Определение производится по HTTP заголовку `Host`.

```nginx
server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
    }
}
```

### 4.3. server_name

Указывает, какие DNS-имена обрабатывает данный `server`.

```nginx
# Один домен
server_name example.com;

# Несколько доменов
server_name example.com www.example.com;

# Все субдомены
server_name example.com *.example.com;

# Запросы без Host (обращение по IP)
server_name "";
```

Если указать значение с символами, недопустимыми в DNS (например `_`), такой `server` будет обрабатывать запросы с любым hostname:

```nginx
server_name _;
```

При длинных значениях `server_name` может потребоваться увеличить хеш-таблицу на уровне `http`:

```nginx
server_names_hash_max_size 512;
server_names_hash_bucket_size 128;
```

### 4.4. listen

Определяет адрес и порт, на которых слушает сервер.

```nginx
# Только порт
listen 80;
listen 8080;

# HTTPS с HTTP/2
listen 443 ssl http2;

# Конкретный IP + порт
listen 127.0.0.1:80;

# IPv6
listen [::]:80;
```

Опция `default_server` — определяет сервер по умолчанию для пары IP:порт, если клиент не прислал заголовок `Host` или прислал невалидный:

```nginx
server {
    listen 80 default_server;
    server_name _;
    return 444;
}
```

### 4.5. return

Возвращает текст или выполняет редирект.

```nginx
# Текстовый ответ
return 200 "hello from nginx";

# Постоянный редирект
return 301 https://example.com;

# Редирект HTTP → HTTPS (для всех доменов)
return 301 https://$host$request_uri;

# Редирект www → без www
server {
    listen 80;
    server_name www.example.com;
    return 301 https://example.com$request_uri;
}
```

### 4.6. client_max_body_size

Максимальный размер тела запроса от клиента. Важно для приложений, принимающих файлы.

```nginx
client_max_body_size 128m;    # мегабайты
client_max_body_size 4096k;   # килобайты
client_max_body_size 1024;    # байты
client_max_body_size 0;       # без ограничений (не рекомендуется)
```

Контекст: `http`, `server`, `location`.

---

## 5. Контекст директив

Каждая директива разрешена только в определённых секциях конфигурации. Это описано в документации модуля.

Пример — директива `user` (только `main`):

```
Syntax:  user user [group];
Default: user nobody nobody;
Context: main
```

Пример — директива `error_log` (практически везде):

```
Syntax:  error_log file [level];
Default: error_log logs/error.log error;
Context: main, http, mail, stream, server, location
```

Для поиска документации по любой директиве: `nginx <имя_директивы>` в поисковике.

---

## 6. Примеры конфигураций Virtual Hosts

### 6.1. Статический сайт

```nginx
server {
    listen 80;
    server_name static.example.com;

    root /var/www/static;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### 6.2. Reverse proxy на backend

```nginx
server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 6.3. HTTPS с редиректом HTTP → HTTPS

```nginx
# Редирект HTTP → HTTPS
server {
    listen 80;
    server_name app.example.com;
    return 301 https://$host$request_uri;
}

# Основной HTTPS сервер
server {
    listen 443 ssl http2;
    server_name app.example.com;

    ssl_certificate     /etc/ssl/certs/app.example.com.crt;
    ssl_certificate_key /etc/ssl/private/app.example.com.key;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 6.4. Default server (заглушка)

Обрабатывает все запросы, не совпавшие ни с одним `server_name`. Возвращает 444 (nginx закрывает соединение без ответа):

```nginx
server {
    listen 80 default_server;
    listen 443 ssl default_server;
    server_name _;

    ssl_certificate     /etc/ssl/certs/default.crt;
    ssl_certificate_key /etc/ssl/private/default.key;

    return 444;
}
```

---

## 7. Тестирование конфигурации

```bash
# Проверить синтаксис
nginx -t

# Перечитать конфигурацию без перезапуска
nginx -s reload

# Или через systemd
systemctl reload nginx
```

Для тестирования `server_name` с локальной машины:

```bash
# Через /etc/hosts
echo "127.0.0.1 app.example.com" >> /etc/hosts

# Через curl с заголовком Host
curl -H "Host: app.example.com" http://127.0.0.1
```

---

## 8. Полезные команды

| Действие | Команда |
|----------|---------|
| Проверить конфиг | `nginx -t` |
| Reload без даунтайма | `nginx -s reload` |
| Посмотреть версию и модули | `nginx -V` |
| Логи доступа | `tail -f /var/log/nginx/access.log` |
| Логи ошибок | `tail -f /var/log/nginx/error.log` |
| Найти все конфиги | `find /etc/nginx -name '*.conf'` |
| Активные подключения | `curl http://127.0.0.1/nginx_status` (требует модуль `stub_status`) |
