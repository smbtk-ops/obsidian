### **Основные ключи команды ssh**

|**Ключ**|**Описание**|**Пример использования**|
|---|---|---|
|-l|Указать имя пользователя (аналогично user@host)|ssh -l user server.com|
|-p|Указать нестандартный порт|ssh -p 2222 user@server.com|
|-i|Указать приватный ключ|ssh -i ~/.ssh/id_rsa user@server.com|
|-v|Включить подробный вывод (debug)|ssh -v user@server.com|
|-X|Включить X11 forwarding (графический вывод)|ssh -X user@server.com|
|-A|Включить пересылку ssh-агента|ssh -A user@server.com|
|-N|Не выполнять команду, только устанавливать туннель|ssh -N -L 8080:localhost:80 user@server.com|
|-L|Настроить локальный порт-форвардинг|ssh -L 8080:localhost:80 user@server.com|
|-R|Настроить удалённый порт-форвардинг|ssh -R 2222:localhost:22 user@server.com|
|-C|Включить сжатие данных|ssh -C user@server.com|

### **Примеры подключения**

```
# Подключение по стандартному порту 22
ssh user@server.com

# Подключение по порту 2222
ssh -p 2222 user@server.com

# Подключение с использованием приватного ключа
ssh -i ~/.ssh/id_rsa user@server.com

# Пересылка локального порта 8080 на удалённый порт 80
ssh -L 8080:localhost:80 user@server.com

# Пересылка удалённого порта 2222 на локальный порт 22
ssh -R 2222:localhost:22 user@server.com

# Только установка туннеля, без запуска shell
ssh -N -L 3306:localhost:3306 user@server.com
```

### **Расширенные опции**

|**Ключ**|**Описание**|**Пример использования**|
|---|---|---|
|-o StrictHostKeyChecking=no|Отключить проверку ключа хоста|ssh -o StrictHostKeyChecking=no user@server.com|
|-o UserKnownHostsFile=/dev/null|Не сохранять ключ хоста в known_hosts|ssh -o UserKnownHostsFile=/dev/null user@server.com|
|-F|Указать альтернативный конфиг-файл (по умолчанию ~/.ssh/config)|ssh -F /path/to/config user@server.com|
