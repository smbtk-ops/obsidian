## 1. Увеличение `client_max_body_size` (загрузка больших файлов)

По умолчанию nginx ограничивает размер тела запроса 1 МБ.  
Чтобы увеличить, например, до 100 МБ:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: upload-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
spec:
  ingressClassName: nginx
  rules:
    - host: files.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: upload-svc
                port:
                  number: 80
```

---
## 2. Настройка таймаутов

Используется для долгих соединений, запросов к медленным backend’ам и т. п.
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
```

---
## 3. Поддержка WebSocket и gRPC

### WebSocket

Активируется автоматически, но иногда требуется явно отключить буферизацию:
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
```

### gRPC

Для gRPC нужно задать правильную схему:
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
```

---
## 4. Редиректы (301/302)

Переадресация с HTTP → HTTPS:
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

Редирект всего домена на другой:
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/permanent-redirect: https://new.example.com$request_uri
```

---
## 5. Ограничение скорости запросов (rate limiting)

Для защиты от DDoS и чрезмерных запросов.
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-connections: "10"
    nginx.ingress.kubernetes.io/limit-rps: "5"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"
    nginx.ingress.kubernetes.io/limit-rate-after: "10m"
```

---
## 6. Кэширование ответов

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_cache my_cache;
      proxy_cache_valid 200 302 10m;
      proxy_cache_valid 404 1m;
      add_header X-Cache-Status $upstream_cache_status;
```

Для этого нужно создать ConfigMap с описанием кэша:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
data:
  proxy-cache-path: "/var/cache/nginx levels=1:2 keys_zone=my_cache:10m inactive=60m use_temp_path=off;"
```

---
## 7. Доступ по IP (Whitelist / Blacklist)

### Разрешить только определённые IP:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/whitelist-source-range: "192.168.0.0/24,10.0.0.0/8"
```

### Запретить конкретные IP:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/server-snippet: |
      deny 203.0.113.5;
      allow all;
```

---
## 8. Basic-Auth (ещё раз кратко)

```bash
htpasswd -c auth user1
kubectl create secret generic basic-auth --from-file=auth -n default
```

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Restricted"
```

---
## 9. HTTP-заголовки и security headers

Добавление кастомных заголовков:
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-Frame-Options "DENY";
      add_header X-Content-Type-Options "nosniff";
      add_header Referrer-Policy "no-referrer";
      add_header Strict-Transport-Security "max-age=63072000; includeSubDomains";
```

---
## 10. Перенаправление на другой backend (rewrite-target)

Для случаев, когда path внутри backend должен отличаться от пути Ingress:
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api/(.*)
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
```

---
## 11. Sticky-сессии (session affinity)

Для stateful-приложений (например, PHP, WebSocket):
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "route"
    nginx.ingress.kubernetes.io/session-cookie-hash: "sha1"
```

---
## 12. Принудительное HTTPS и HSTS

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/hsts: "true"
    nginx.ingress.kubernetes.io/hsts-max-age: "31536000"
    nginx.ingress.kubernetes.io/hsts-include-subdomains: "true"
```

---
## 13. Настройка CORS

Разрешение кросс-доменных запросов:
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://frontend.example.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "PUT, GET, POST, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "Authorization, Content-Type"
```

---
## 14. Установка заголовков X-Forwarded и Real-IP

Чтобы backend получал реальный IP клиента:
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/use-forwarded-headers: "true"
    nginx.ingress.kubernetes.io/enable-real-ip: "true"
```

---
## 15. Ограничение параллельных подключений

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-connections: "20"
```

---
## 16. Принудительный редирект HTTP→HTTPS + проксирование WebSocket

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
```

---
## 17. Настройка gzip-сжатия

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/configuration-snippet: |
      gzip on;
      gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
      gzip_min_length 256;
```

---
## 18. Пример секции с несколькими аннотациями (комплексный случай)

Ingress для API с CORS, gzip и auth:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      gzip on;
      gzip_types text/plain text/css application/json application/javascript;
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
```
---
# Справочник аннотаций ingress-nginx

|Аннотация|Назначение / описание|Тип значения|Значение по умолчанию|
|---|---|---|---|
|**nginx.ingress.kubernetes.io/proxy-body-size**|Максимальный размер тела запроса (аналог `client_max_body_size`)|строка (например `"50m"`)|`"1m"`|
|**nginx.ingress.kubernetes.io/proxy-connect-timeout**|Таймаут соединения между nginx и backend (в секундах)|строка (например `"60"`)|`"5"`|
|**nginx.ingress.kubernetes.io/proxy-send-timeout**|Таймаут отправки данных на backend|строка|`"60"`|
|**nginx.ingress.kubernetes.io/proxy-read-timeout**|Таймаут чтения ответа от backend|строка|`"60"`|
|**nginx.ingress.kubernetes.io/proxy-buffering**|Включение/выключение буферизации ответов backend|`"on"` / `"off"`|`"on"`|
|**nginx.ingress.kubernetes.io/backend-protocol**|Протокол взаимодействия с backend (`HTTP`, `HTTPS`, `GRPC`, `GRPCS`, `AJP`)|строка|`"HTTP"`|
|**nginx.ingress.kubernetes.io/rewrite-target**|Переписывание пути запроса перед передачей на backend|строка|пусто|
|**nginx.ingress.kubernetes.io/force-ssl-redirect**|Принудительное перенаправление HTTP→HTTPS|`"true"` / `"false"`|`"true"`|
|**nginx.ingress.kubernetes.io/ssl-redirect**|Включить SSL редирект (альтернатива force-ssl-redirect)|`"true"` / `"false"`|`"true"`|
|**nginx.ingress.kubernetes.io/permanent-redirect**|Перенаправление на другой URL (301)|строка (URL)|пусто|
|**nginx.ingress.kubernetes.io/proxy-redirect-from**|Указывает, откуда выполнять редирект (аналог `proxy_redirect`)|строка|пусто|
|**nginx.ingress.kubernetes.io/proxy-redirect-to**|Указывает, куда выполнять редирект|строка|пусто|
|**nginx.ingress.kubernetes.io/configuration-snippet**|Вставка произвольных директив nginx в `location {}`|YAML multiline|пусто|
|**nginx.ingress.kubernetes.io/server-snippet**|Вставка директив nginx в `server {}`|YAML multiline|пусто|
|**nginx.ingress.kubernetes.io/auth-type**|Тип аутентификации (`basic` или `digest`)|строка|пусто|
|**nginx.ingress.kubernetes.io/auth-secret**|Имя секрета с данными аутентификации|строка|пусто|
|**nginx.ingress.kubernetes.io/auth-realm**|Текст, отображаемый в окне авторизации|строка|`"Authentication Required"`|
|**nginx.ingress.kubernetes.io/enable-cors**|Разрешение CORS-запросов|`"true"` / `"false"`|`"false"`|
|**nginx.ingress.kubernetes.io/cors-allow-origin**|Разрешённые источники (`*` или список доменов)|строка|`"*"`|
|**nginx.ingress.kubernetes.io/cors-allow-methods**|Разрешённые HTTP-методы|строка|`"GET, PUT, POST, OPTIONS"`|
|**nginx.ingress.kubernetes.io/cors-allow-headers**|Разрешённые заголовки|строка|`"DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization"`|
|**nginx.ingress.kubernetes.io/limit-connections**|Ограничение одновременных соединений на IP|число|нет|
|**nginx.ingress.kubernetes.io/limit-rps**|Ограничение запросов в секунду|число|нет|
|**nginx.ingress.kubernetes.io/limit-burst-multiplier**|Размер буфера burst|число|`"5"`|
|**nginx.ingress.kubernetes.io/limit-rate-after**|Начало ограничения скорости после определённого объёма (например `10m`)|строка|пусто|
|**nginx.ingress.kubernetes.io/affinity**|Включение sticky-сессий (`cookie`)|строка|пусто|
|**nginx.ingress.kubernetes.io/session-cookie-name**|Имя cookie для session affinity|строка|`"route"`|
|**nginx.ingress.kubernetes.io/session-cookie-hash**|Алгоритм хэширования cookie (`md5`, `sha1`)|строка|`"sha1"`|
|**nginx.ingress.kubernetes.io/whitelist-source-range**|Список разрешённых IP/CIDR для доступа|строка|пусто|
|**nginx.ingress.kubernetes.io/use-forwarded-headers**|Использовать заголовки `X-Forwarded-*` клиента|`"true"` / `"false"`|`"false"`|
|**nginx.ingress.kubernetes.io/enable-real-ip**|Передавать реальный IP клиента в backend|`"true"` / `"false"`|`"false"`|
|**nginx.ingress.kubernetes.io/proxy-cookie-path**|Переписывание `Set-Cookie Path`|строка|пусто|
|**nginx.ingress.kubernetes.io/proxy-cookie-domain**|Переписывание `Set-Cookie Domain`|строка|пусто|
|**nginx.ingress.kubernetes.io/proxy-buffer-size**|Размер одного буфера для чтения заголовков|строка (например `"8k"`)|`"4k"`|
|**nginx.ingress.kubernetes.io/proxy-buffers-number**|Количество буферов|число|`"4"`|
|**nginx.ingress.kubernetes.io/hsts**|Включение HSTS|`"true"` / `"false"`|`"false"`|
|**nginx.ingress.kubernetes.io/hsts-max-age**|Время жизни HSTS (в секундах)|строка|`"31536000"`|
|**nginx.ingress.kubernetes.io/hsts-include-subdomains**|Включить HSTS для поддоменов|`"true"` / `"false"`|`"false"`|
|**nginx.ingress.kubernetes.io/add-base-url**|Добавлять `base` тег в HTML|`"true"` / `"false"`|`"false"`|
|**nginx.ingress.kubernetes.io/proxy-request-buffering**|Разрешить/запретить буферизацию входящих запросов|`"on"` / `"off"`|`"on"`|
|**nginx.ingress.kubernetes.io/proxy-response-buffering**|Разрешить/запретить буферизацию ответов|`"on"` / `"off"`|`"on"`|
|**nginx.ingress.kubernetes.io/client-body-buffer-size**|Размер буфера тела клиента|строка|`"8k"`|
|**nginx.ingress.kubernetes.io/custom-http-errors**|Список HTTP-кодов для кастомных error pages|строка, например `"404,503"`|пусто|
|**nginx.ingress.kubernetes.io/upstream-hash-by**|Балансировка трафика по значению выражения (например `$remote_addr`)|строка|пусто|
|**nginx.ingress.kubernetes.io/server-tokens**|Отображать ли версию nginx в ответах|`"true"` / `"false"`|`"false"`|
|**nginx.ingress.kubernetes.io/proxy-next-upstream**|Поведение при ошибках backend (например `"error timeout"`)|строка|`"error timeout"`|
|**nginx.ingress.kubernetes.io/proxy-next-upstream-tries**|Количество попыток перехода к следующему upstream|число|`"3"`|
|**nginx.ingress.kubernetes.io/denylist-source-range**|Явный список запрещённых IP/CIDR|строка|пусто|
|**nginx.ingress.kubernetes.io/enable-access-log**|Включить access.log для данного Ingress|`"true"` / `"false"`|`"true"`|
|**nginx.ingress.kubernetes.io/enable-rewrite-log**|Включить rewrite.log (для отладки правил переписывания)|`"true"` / `"false"`|`"false"`|
|**nginx.ingress.kubernetes.io/proxy-buffering**|Управление буферизацией ответов backend|`"on"` / `"off"`|`"on"`|
|**nginx.ingress.kubernetes.io/proxy-http-version**|Версия HTTP-протокола для backend|`"1.0"` / `"1.1"`|`"1.1"`|
