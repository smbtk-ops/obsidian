# **curl — получение данных из Интернета**

curl — утилита командной строки для передачи данных с использованием различных сетевых протоколов (HTTP, HTTPS, FTP и др.).

Наиболее часто используется для скачивания файлов, отправки HTTP-запросов и взаимодействия с API.

### **Основные ключи команды curl**

| **Ключ**     | **Описание**                                                 | **Пример использования**                                                               |
| ------------ | ------------------------------------------------------------ | -------------------------------------------------------------------------------------- |
| -o           | Сохранить вывод в файл                                       | curl -o файл.html https://example.com                                                  |
| -O           | Сохранить файл с именем, как на сервере                      | curl -O https://example.com/file.txt                                                   |
| -L           | Следовать за перенаправлениями                               | curl -L https://example.com                                                            |
| -I           | Получить только заголовки ответа                             | curl -I https://example.com                                                            |
| -X           | Указать метод запроса (например, GET, POST, PUT, DELETE)     | curl -X POST https://example.com                                                       |
| -d           | Передать данные в запросе POST                               | curl -d "key=value" -X POST https://example.com                                        |
| -H           | Добавить заголовок в запрос                                  | curl -H "Authorization: Bearer токен" https://example.com                              |
| -u           | Аутентификация через логин и пароль                          | curl -u имя:пароль https://example.com                                                 |
| -k           | Игнорировать ошибки проверки SSL-сертификата                 | curl -k https://example.com                                                            |
| -s           | Тихий режим (не выводить прогресс)                           | curl -s https://example.com                                                            |
| -v           | Подробный вывод для отладки (verbose)                        | curl -v https://example.com                                                            |
| --compressed | Запрашивать сжатый (gzip) ответ                              | curl --compressed https://example.com                                                  |
| -i           | включает в вывод **заголовки ответа** вместе с телом ответа. | curl https://example.com -i                                                            |
| -d @-        | читать данные из потока stdin                                | grep "torvalds@linux-foundation.org" \| curl -X PUT -d @- https://httpbin.org/anything |

### **Примеры использования curl**

#### **1. Скачать страницу в файл**
```
curl -o index.html https://example.com
```

#### **2. Скачать файл с сохранением имени**
```
curl -O https://example.com/file.zip
```

#### **3. Отправить POST-запрос с параметрами**
```
curl -d "username=admin&password=1234" -X POST https://example.com/login
```

#### **4. Отправить GET-запрос с кастомным заголовком**
```
curl -H "X-Custom-Header: value" https://example.com
```

#### **5. Скачивание через прокси**
```
curl -x http://proxy.example.com:8080 https://example.com
```

#### **6. Скачивание большого файла с отображением прогресса**
```
curl -# -O https://example.com/largefile.iso
```


### **Расширенные ключи команды curl**

|**Ключ**|**Описание**|**Пример использования**|
|---|---|---|
|-b или --cookie|Отправить cookie с запросом|curl -b "session=abcd1234" https://example.com|
|-c или --cookie-jar|Сохранить cookie из ответа в файл|curl -c cookies.txt https://example.com|
|-e или --referer|Установить заголовок Referer|curl -e https://google.com https://example.com|
|-A или --user-agent|Установить User-Agent|curl -A "Mozilla/5.0" https://example.com|
|--connect-timeout|Ограничить время на подключение к серверу (в секундах)|curl --connect-timeout 5 https://example.com|
|--max-time|Ограничить общее время выполнения запроса (в секундах)|curl --max-time 10 https://example.com|
|--retry|Попытки повторить запрос при ошибке|curl --retry 3 https://example.com|
|--data-urlencode|Отправить данные в кодировке URL|curl --data-urlencode "param=значение" https://example.com|
|--cert|Указать клиентский SSL-сертификат для аутентификации|curl --cert client.pem https://example.com|
|--proxy|Использовать прокси-сервер|curl --proxy http://proxy.example.com:8080 https://example.com|
|--http2|Принудительно использовать HTTP/2|curl --http2 https://example.com|
|--upload-file|Отправить файл на сервер|curl --upload-file файл.txt https://example.com/upload|

### **Часто используемые сценарии curl**

#### **Скачать файл через HTTPS с автоматическим следованием редиректам**
```
curl -L -O https://example.com/redirected/file.zip
```

#### **Аутентификация через токен (например, Bearer Token)**
```
curl -H "Authorization: Bearer <токен>" https://api.example.com/user
```

#### **Получить заголовки ответа без загрузки содержимого**
```
curl -I https://example.com
```

#### **Загрузить файл на сервер через PUT-запрос**
```
curl -T localfile.txt https://example.com/upload
```

#### **Загрузить файл с автоматическим сохранением куков**
```
curl -c cookies.txt -b cookies.txt https://example.com
```

#### **Повторить запрос 5 раз при ошибках сети**
```
curl --retry 5 https://example.com
```

#### **Ограничить время запроса 10 секундами**
```
curl --max-time 10 https://example.com
```

#### **Отправить несколько заголовков в одном запросе**
```
curl -H "Accept: application/json" -H "X-Api-Version: 2" https://api.example.com/data
```
