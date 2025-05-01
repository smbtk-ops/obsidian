### **Основные ключи команды useradd**

|**Ключ**|**Описание**|**Пример использования**|
|---|---|---|
|-m|Создаёт домашний каталог|useradd -m username|
|-d /путь|Указывает путь к домашнему каталогу|useradd -d /home/custom username|
|-s /путь|Указывает оболочку пользователя|useradd -s /bin/bash username|
|-c "..."|Добавляет комментарий (например, ФИО)|useradd -c "Иван Иванов" username|
|-e YYYY-MM-DD|Устанавливает дату истечения уч. записи|useradd -e 2025-12-31 username|
|-f ЧИСЛО|Кол-во дней до отключения после истечения пароля|useradd -f 7 username|
|-g группа|Основная группа пользователя|useradd -g users username|
|-G grp1,grp2|Дополнительные группы|useradd -G sudo,adm username|
|-u UID|Задает UID|useradd -u 1001 username|
|-p хэш|Зашифрованный пароль|useradd -p $(openssl passwd -1 'пароль') username|
|-r|Создаёт системного пользователя|useradd -r username|
|-M|Не создаёт домашнюю директорию|useradd -M username|
|-N|Не создаёт группу с именем пользователя|useradd -N username|
|-o|Разрешает неуникальный UID|useradd -o -u 0 username|
|-K ключ=знач|Переопределяет значения из /etc/login.defs|useradd -K UID_MIN=500 username|
|-b /базовый/каталог|Базовый путь для домашнего каталога|useradd -b /home username|
|-k /шаблон|Каталог-шаблон (обычно /etc/skel)|useradd -k /etc/skel username|
|-Z SELinux-пользователь|Устанавливает SELinux-контекст|useradd -Z user_u username|
    
### **Примеры использования команды useradd**

   **Создание пользователя с домашним каталогом и оболочкой Bash**:
```
useradd -m -s /bin/bash username
```
   **Создание пользователя с определенным UID и GID**:
```
useradd -u 1050 -g 100 username
```
   **Создание системного пользователя без домашнего каталога**:
```
useradd -r -M username
```
   **Создание пользователя с ограниченным сроком действия учетной записи**:
```
useradd -e 2025-12-31 username
```
   **Создание пользователя и добавление его в дополнительные группы**:
```
useradd -m -G sudo,adm username
```
   **Создание пользователя с определенным домашним каталогом и шаблоном**:
```
useradd -m -d /home/custom -k /etc/skel username
```
   **Создание пользователя с зашифрованным паролем**:
```
useradd -m -p $(openssl passwd -1 'пароль') username
```
