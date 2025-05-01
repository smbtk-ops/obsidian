Команда chpasswd используется для **массового назначения/смены паролей** в Linux. Она читает пары username:password из stdin и обновляет /etc/shadow.

## **Ключи  chpasswd**

| **Ключ** | **Описание**                                                         |
| -------- | -------------------------------------------------------------------- |
| -e       | Указывает, что пароли уже **зашифрованы** (в виде хэшей)             |
| -c       | Указывает **метод шифрования**, если не установлен в /etc/login.defs |
| -m       | Использует MD5 для хэширования (устаревшее, используйте -c вместо)   |
| -s       | Указывает используемую оболочку (редко используется)                 |
| --help   | Показать справку                                                     |

## ** Примеры использования chpasswd**

### **Назначить простой пароль пользователю**
```
echo "ivanov:Qwerty123" | sudo chpasswd
```

### **Массово задать пароли нескольким пользователям**
```
cat <<EOF | sudo chpasswd
user1:Password1
user2:Password2
user3:Password3
EOF
```

### **Указать, что пароли уже зашифрованы (-e)**
```
echo "user1:\$6\$qG7K...A1" | sudo chpasswd -e
```
 $6$... — это SHA-512. Получить можно через:
```
openssl passwd -6 'Password1'
```

### **Использовать указанный алгоритм (-c)**
```
echo "user1:Password1" | sudo chpasswd -c SHA512
```
Возможные значения: DES, MD5, SHA256, SHA512.

## **Где применяется chpasswd**
- В **скриптах автоматизации** (init-скрипты, cloud-init)
- В **CI/CD пайплайнах** (например, создание пользователей в Docker или на VM)
- В **Ansible**: ansible.builtin.user использует его внутри

## ** Вариант 1: Использование chpasswd в Dockerfile**

```
FROM ubuntu:22.04

RUN apt-get update && \
    apt-get install -y passwd && \
    useradd -m -s /bin/bash devuser && \
    echo "devuser:SuperSecure123" | chpasswd
```

При запуске контейнера пользователь devuser сможет зайти с паролем SuperSecure123.

### ** Пример запуска с bash:**
```
docker build -t myimage .
docker run -it myimage bash
```

## ** Вариант 2: Использование chpasswd в cloud-init (для VPS, EC2 и т.д.)**

### **user-data.yaml**
```
#cloud-config
users:
  - name: devuser
    gecos: Dev Ops
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: false
    passwd: $6$rounds=4096$3zX1tVuVEXz6oW0k$LO4NWW1z/7sSIZOjkV...
	ssh_authorized_keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC3g2S...

chpasswd:
  expire: false
```

passwd: должен быть **SHA-512 хэш**, который можно сгенерировать так:

```
openssl passwd -6 'SuperSecure123'
```

Пример использования (DigitalOcean, Yandex.Cloud, Hetzner):
- вставляется в поле **cloud-init** или **user-data**
- применяется автоматически при создании инстанса

|**Элемент**|**Назначение**|
|---|---|
|lock_passwd: false|Разрешает логин по паролю|
|passwd:|SHA-512 хэш пароля, создаётся через openssl passwd -6 'пароль'|
|ssh_authorized_keys:|Открытый SSH-ключ (добавляется в ~/.ssh/authorized_keys)|
|sudo: ALL=(ALL) NOPASSWD:ALL|Позволяет использовать sudo без ввода пароля|
|chpasswd.expire: false|Отключает принудительную смену пароля при первом входе|