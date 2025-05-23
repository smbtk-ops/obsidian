# **SSH (Secure Shell)**

SSH — это протокол защищённого удалённого доступа, позволяющий управлять серверами, запускать команды, пробрасывать порты и даже запускать X-приложения через зашифрованный канал.

## **Основные компоненты**

| **Компонент**                             | **Описание**                                                                          |
| ----------------------------------------- | ------------------------------------------------------------------------------------- |
| [[Linux/Default soft/Utils/ssh/ssh\|ssh]] | Клиентская часть, команда для подключения.                                            |
| [[sshd]]                                  | Серверная часть, демон, который слушает порт (обычно 22) и принимает подключения.     |
| [[ssh-keygen]]                            | Утилита для генерации и управления SSH-ключами.                                       |
| [[ssh-copy-id]]                           | Утилита для проброса публичного ключа на сервер (добавляет в ~/.ssh/authorized_keys). |
| [[ssh-agent]]                             | Агент, который хранит приватные ключи в памяти и предоставляет их клиенту.            |
| [[ssh-keyscan]]                           | используется для **быстрого получения публичных SSH-ключей удалённых серверов**.      |

## **Конфигурационные файлы**

|**Файл**|**Назначение**|
|---|---|
|/etc/ssh/sshd_config|Конфигурация sshd (сервера).|
|~/.ssh/config|Конфигурация клиента, определение алиасов, настроек хостов.|
|~/.ssh/authorized_keys|Список публичных ключей, которые имеют доступ к аккаунту на сервере.|
|~/.ssh/known_hosts|Список отпечатков серверов, к которым подключался клиент.|
|~/.ssh/id_rsa, id_ed25519|Приватные ключи клиента.|
|~/.ssh/id_rsa.pub, id_ed25519.pub|Соответствующие публичные ключи.|

## **Генерация и управление ключами**

| **Действие**                       | **Команда**                                                                   |
| ---------------------------------- | ----------------------------------------------------------------------------- |
| Генерация ключа                    | ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 (рекомендуемый алгоритм — Ed25519) |
| Извлечение публичного ключа        | ssh-keygen -y -f ~/.ssh/id_ed25519                                            |
| Просмотр отпечатка ключа           | ssh-keygen -l -f ~/.ssh/id_ed25519                                            |
| Проброс публичного ключа на сервер | ssh-copy-id user@host                                                         |

## **Основные команды SSH**

| **Сценарий**                               | **Команда**                                         |
| ------------------------------------------ | --------------------------------------------------- |
| Подключение к серверу                      | ssh user@host                                       |
| Использование конкретного порта            | ssh -p 2222 user@host                               |
| Использование конкретного приватного ключа | ssh -i ~/.ssh/custom_key user@host                  |
| Выполнение команды удалённо                | ssh user@host 'uptime'                              |
| Проброс порта (локальный forward)          | ssh -L 8080:localhost:80 user@host                  |
| Просмотр отладочных сообщений              | ssh -v user@host (-vv или -vvv для большего уровня) |

## *SSH Agent*

ssh-agent — это процесс, который хранит приватные ключи **в оперативной памяти**, чтобы ты не вводил пароль (passphrase) к каждому ключу при каждом подключении.

**Как это работает:**
- запускаешь ssh-agent.
- добавляешь ключ в агент командой ssh-add ~/.ssh/id_ed25519.
- Когда ssh клиенту нужно подключиться, он автоматически обращается к ssh-agent и проверяет, есть ли там нужный ключ.
- Если ключ найден, SSH использует его **без запроса пароля**.

**Преимущества:**
- Один раз разблокировал ключ → используешь во множестве сессий.
- Удобно для автоматизированных скриптов или CI/CD.
- Поддержка агент-форвардинга (см. ниже).

|**Команда**|**Описание**|
|---|---|
|eval $(ssh-agent)|Запустить агент.|
|ssh-add ~/.ssh/id_ed25519|Добавить ключ в агент.|
|ssh-add -l|Список ключей в агенте.|
|ssh-add -d ~/.ssh/id_ed25519|Удалить конкретный ключ из агента.|
|ssh -A user@host|Проброс SSH-агента на сервер (для дальнейших подключений с него).|

## **Jump Host (бастион)**

Jump Host (бастион, jump server) — это промежуточный сервер, через который можно подключаться **к другим внутренним серверам**.

Часто используется, когда:
- Внешний сервер доступен из интернета.
- Внутренние серверы (например, в приватной сети) недоступны напрямую.

**Как это выглядит:**
```
[твой ноутбук] → [jump host (публичный)] → [внутренний сервер (приватный)]
```

**Основные методы:**

| **Метод**                                                | **Команда**                                                         |
| -------------------------------------------------------- | ------------------------------------------------------------------- |
| Через ProxyCommand                                       | ssh -o ProxyCommand="ssh -W %h:%p user@jump_host" user@private_host |
| Через jump-флаг -J                                       | ssh -J user@jump_host user@private_host                             |

**Как работает:**
- SSH-клиент сначала подключается к jump host.
- Оттуда он делает второе подключение к внутреннему серверу.
- Весь трафик туннелируется, **шифруется end-to-end**.

**Советы:**
- Настрой ~/.ssh/config для удобства, чтобы не писать длинные команды.
- Jump host может требовать отдельный ключ, отличающийся от того, который используется на внутренних серверах.

## **Дополнительно: проверка сервера**

Чтобы удостовериться, что сервер — это тот, к которому ты уже подключался (и избежать MITM-атак), SSH хранит его отпечаток в ~/.ssh/known_hosts.

Если отпечаток поменялся, SSH предупредит тебя.

## **Отладка**

|**Что проверить**|**Инструмент**|
|---|---|
|На клиенте|ssh -vvv user@host (подробные логи)|
|На сервере (логи)|/var/log/auth.log|
|На сервере (debug-режим)|Остановить sshd и запустить вручную: sshd -d -p 2222|


# **Шаблон ~/.ssh/config**
**jump host**
**агент-форвардинга**
**проброса портов (локального, удалённого и SOCKS)**

````
SSH config — шаблон с jump host, пробросом портов и агентом

Файл: ~/.ssh/config

=== Общие настройки для всех хостов ===
Host *
    ForwardAgent yes                # Пробрасывать SSH-агент (для использования ключей с удалённых хостов)
    Compression yes                # Включить сжатие трафика
    ServerAliveInterval 60         # Поддерживать соединение живым
    ControlMaster auto             # Мультиплексирование SSH-соединений
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m             # Держать соединение открытым после выхода (кэширование)

=== Bastion / Jump Host ===
Host bastion
    HostName 203.0.113.10           # Внешний IP бастиона
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519  # Путь к приватному ключу

=== Приватный сервер за бастионом ===
Host internal
    HostName 10.0.1.5
    User ubuntu
    ProxyJump bastion              # Использовать bastion как промежуточный хост
    IdentityFile ~/.ssh/id_ed25519

=== Проброс локального порта (локальный forward) ===
Host web-proxy
    HostName web.internal
    User webadmin
    LocalForward 8080 localhost:80  # Пробрасываем 8080 на локальной → 80 на удалённой

=== Проброс удалённого порта (remote forward) ===
Host expose-local
    HostName app.server
    User admin
    RemoteForward 9000 localhost:3000  # Пробрасываем удалённый 9000 → локальный 3000

=== SOCKS-прокси (динамический проброс) ===
Host dynamic-proxy
    HostName jump.proxy.net
    User proxy
    DynamicForward 1080            # SOCKS5-прокси на локальном порту 1080
````

----

- [sshd config (разбор параметров)](http://tdkare.ru/sysadmin/index.php/Sshd_config)
- [ssh-keygen - Generate a New SSH Key](https://www.ssh.com/ssh/keygen)
- [Управление ключами SSH с помощью агента](http://xgu.ru/wiki/%D0%A3%D0%BF%D1%80%D0%B0%D0%B2%D0%BB%D0%B5%D0%BD%D0%B8%D0%B5_%D0%BA%D0%BB%D1%8E%D1%87%D0%B0%D0%BC%D0%B8_SSH_%D1%81_%D0%BF%D0%BE%D0%BC%D0%BE%D1%89%D1%8C%D1%8E_%D0%B0%D0%B3%D0%B5%D0%BD%D1%82%D0%B0)
- [Как использовать ssh-keyscan в Ubuntu](https://ru.linux-console.net/?p=15519)
  
  [[Shared Libraries]]