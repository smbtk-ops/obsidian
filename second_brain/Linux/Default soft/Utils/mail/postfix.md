## **Postfix — почтовый агент (MTA) для отправки/получения почты в Linux**

Postfix — это один из самых популярных MTA (Mail Transfer Agent), используемый для обработки исходящей и входящей почты. Он используется как локальный почтовый сервер или как шлюз для передачи писем другим MTA.

### **Основные команды Postfix**

|**Команда**|**Описание**|
|---|---|
|postfix start|Запустить Postfix|
|postfix stop|Остановить Postfix|
|postfix reload|Перезагрузить конфигурацию|
|postfix check|Проверить конфигурационные файлы|
|postfix status|Проверить состояние Postfix|
|postfix flush|Принудительно отправить очередь писем|
|postfix upgrade-configuration|Обновить конфигурацию до актуального формата|

### **Основные конфигурационные файлы**

|**Файл**|**Назначение**|
|---|---|
|/etc/postfix/main.cf|Главный конфигурационный файл|
|/etc/postfix/master.cf|Определяет службы (SMTP, pickup, cleanup и т.д.)|
|/etc/aliases|Таблица перенаправлений для локальной почты (использовать newaliases)|

### **Ключевые параметры main.cf**

```
myhostname = mail.example.com         # имя хоста
mydomain = example.com                # домен
myorigin = $mydomain                 # от имени какого домена отправляется почта
inet_interfaces = all                # слушать все интерфейсы (или loopback-only)
inet_protocols = all                 # поддержка IPv4 и IPv6
mydestination = $myhostname, localhost.$mydomain, localhost
relayhost = [smtp.relay.com]:587     # SMTP-релей
mynetworks = 127.0.0.0/8             # какие хосты могут отправлять почту
home_mailbox = Maildir/             # использовать Maildir вместо mbox
smtpd_tls_cert_file = /etc/ssl/certs/postfix.pem  # TLS сертификат
smtpd_tls_key_file = /etc/ssl/private/postfix.key # TLS ключ
smtpd_use_tls = yes                  # включить TLS
smtpd_sasl_auth_enable = yes         # включить SMTP-аутентификацию
```

#### **Отправка письма вручную (локально)**
```
echo "Test email body" | mail -s "Subject" user@localhost
```

#### **Просмотр очереди сообщений**
```
mailq
```

#### **Удаление сообщений из очереди**
```
postsuper -d ALL
```

### **Диагностика и отладка**

| **Утилита**               | **Описание**                            |
| ------------------------- | --------------------------------------- |
| postconf                  | Просмотр или изменение настроек         |
| postcat -vq <queue_id>    | Посмотреть содержимое письма из очереди |
| tail -f /var/log/mail.log | Лог работы Postfix                      |

## **Postfix: настройка SMTP-клиента с SASL и STARTTLS (relay через внешний SMTP)**

Позволяет использовать Postfix для отправки почты через внешний SMTP-сервер с авторизацией и шифрованием.

### **1. Основные изменения в main.cf**

Добавьте или измените следующие строки:
```
# Общие параметры
relayhost = [smtp.gmail.com]:587               # SMTP-релей (например, Gmail)
smtp_use_tls = yes                              # Включить TLS
smtp_tls_security_level = encrypt               # Требовать шифрование
smtp_tls_note_starttls_offer = yes              # Записать, если STARTTLS был предложен

# SASL аутентификация
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous       # Запретить анонимную авторизацию
smtp_sasl_tls_security_options = noanonymous
smtp_sasl_mechanism_filter = login              # Используем LOGIN/PLAIN (в зависимости от провайдера)

# (необязательно) логгирование TLS
smtp_tls_loglevel = 1
```

### **2. Файл с логином и паролем sasl_passwd**

Создайте файл /etc/postfix/sasl_passwd:
```
[smtp.gmail.com]:587 username@gmail.com:your_app_password
```
> **Важно**: для Gmail потребуется [создать app-password](https://myaccount.google.com/apppasswords), если включена двухфакторная аутентификация.

### **3. Защита и генерация базы Postfix**
```
sudo chmod 600 /etc/postfix/sasl_passwd
sudo postmap /etc/postfix/sasl_passwd
```

### **4. Перезапуск Postfix**
```
sudo systemctl restart postfix
```

### **5. Проверка отправки почты**

Пример отправки тестового письма:
```
echo "Email body" | mail -s "Test subject" recipient@example.com
```

Логи смотреть через:
```
tail -f /var/log/mail.log
```

### **Альтернативные relay-сервисы**

|**SMTP**|**Хост**|**Порт**|**Примечание**|
|---|---|---|---|
|Gmail|smtp.gmail.com|587|App Password, 2FA|
|Yandex|smtp.yandex.ru|587|Требует авторизацию|
|Mailgun|smtp.mailgun.org|587|Работает с login:password|
|SendGrid|smtp.sendgrid.net|587|username: apikey, password: <API-KEY>|

## **Настройка входящей почты: Postfix + Dovecot (Maildir + IMAP)**

Postfix доставляет почту в формат Maildir, а Dovecot предоставляет доступ к ней по IMAP/POP3.

### **1. Настройка Postfix для Maildir**

В /etc/postfix/main.cf:
```
home_mailbox = Maildir/
mailbox_command =
```
> Это указывает Postfix доставлять входящие письма в ~/Maildir/ (вместо mbox /var/mail/USER).


### **2. Создание структуры Maildir для пользователя**
```
sudo su - user
maildirmake ~/Maildir
maildirmake ~/Maildir/.Sent
maildirmake ~/Maildir/.Trash
maildirmake ~/Maildir/.Drafts
```

Или одной командой:
```
sudo apt install dovecot-core
sudo doveadm mailbox create -u user INBOX
```

### **3. Установка и базовая настройка Dovecot**

Установка:
```
sudo apt install dovecot-imapd dovecot-pop3d
```

#### **Пример** 
#### **/etc/dovecot/dovecot.conf**
```
protocols = imap pop3
```

#### **Пример** 
#### **/etc/dovecot/conf.d/10-mail.conf**
```
mail_location = maildir:~/Maildir
```

#### **Пример** 
#### **/etc/dovecot/conf.d/10-auth.conf**
```
disable_plaintext_auth = no
auth_mechanisms = plain login
!include auth-system.conf.ext
```

#### **Пример** 

#### **/etc/dovecot/conf.d/10-master.conf**
####  **(для доступа по IMAP)**
```
service imap-login {
  inet_listener imap {
    port = 143
  }
  inet_listener imaps {
    port = 993
    ssl = yes
  }
}
```

### **4. (Опционально) SSL для IMAPS (порт 993)**

В /etc/dovecot/conf.d/10-ssl.conf:
```
ssl = yes
ssl_cert = </etc/ssl/certs/mail.pem
ssl_key = </etc/ssl/private/mail.key
```

### **5. Перезапуск служб**
```
sudo systemctl restart postfix
sudo systemctl restart dovecot
```


### **6. Проверка работы IMAP**

Через telnet:
```
telnet localhost 143
```

Или почтовым клиентом (Thunderbird, K-9 Mail, Apple Mail):

- Протокол: IMAP
- Сервер: your-server
- Пользователь: Linux user
- Пароль: Linux password

### **Лог-файлы для отладки**

| **Сервис** | **Файл**                                   |
| ---------- | ------------------------------------------ |
| Postfix    | /var/log/mail.log                          |
| Dovecot    | /var/log/mail.log или /var/log/dovecot.log |
