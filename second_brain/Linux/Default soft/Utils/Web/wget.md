# **wget**

wget — это популярная утилита командной строки для загрузки файлов по протоколам HTTP, HTTPS и FTP.

### **Основные ключи команды wget**

|**Ключ**|**Описание**|**Пример**|
|---|---|---|
|-O <файл>|Сохраняет загруженный файл под указанным именем|wget -O index.html http://example.com|
|-P <директория>|Сохраняет файл в указанную директорию|wget -P /tmp/downloads http://example.com/file.zip|
|-c|Возобновляет прерванную загрузку|wget -c http://example.com/largefile.iso|
|-b|Загружает файл в фоновом режиме|wget -b http://example.com/largefile.iso|
|-q|Тихий режим (без вывода прогресса)|wget -q http://example.com/file.zip|
|-nv|Минимальный вывод (только важная информация)|wget -nv http://example.com/file.zip|
|--limit-rate=<скорость>|Ограничение скорости загрузки|wget --limit-rate=200k http://example.com/file.zip|
|--no-check-certificate|Игнорировать ошибки сертификата HTTPS|wget --no-check-certificate https://example.com|
|--user=<имя> --password=<пароль>|Указание имени пользователя и пароля для аутентификации|wget --user=admin --password=secret http://example.com/protected|
|-r|Рекурсивная загрузка сайта|wget -r http://example.com/folder/|
|-l <уровень>|Ограничение глубины рекурсивной загрузки|wget -r -l 2 http://example.com/folder/|
|--mirror|Зеркалирование сайта (эквивалентно -r -N -l inf --no-remove-listing)|wget --mirror http://example.com/|
|-np|Не подниматься к родительским директориям при рекурсивной загрузке|wget -r -np http://example.com/subfolder/|
|-N|Загружать только обновленные файлы (по времени изменения)|wget -N http://example.com/file.zip|
|--robots=off|Игнорировать ограничения из файла robots.txt|wget --robots=off http://example.com/|
|-i <файл>|Скачать список файлов из текстового файла|wget -i list_of_urls.txt|
|--user-agent="<строка>"|Задать свой User-Agent|wget --user-agent="Mozilla/5.0" http://example.com/|

### **Практические примеры**

#### **1. Простая загрузка файла**
```
wget http://example.com/file.zip
```

#### **2. Загрузка с указанием имени файла**
```
wget -O newfile.zip http://example.com/file.zip
```

#### **3. Загрузка в указанную папку**
```
wget -P /downloads http://example.com/file.zip
```

#### **4. Загрузка файла без проверки SSL-сертификата**
```
wget --no-check-certificate https://example.com/file.zip
```

#### **5. Загрузка сайта целиком с сохранением структуры каталогов**
```
wget --mirror -p --convert-links -P ./local-site http://example.com/
```
