### **Основные ключи команды ssh-keygen**

|**Ключ**|**Назначение**|**Пример использования**|
|---|---|---|
|-t|Указывает тип ключа (rsa, ecdsa, ed25519, dsa)|ssh-keygen -t rsa|
|-b|Размер ключа в битах (например, 2048 или 4096 для RSA)|ssh-keygen -t rsa -b 4096|
|-f|Путь и имя файла для сохранения ключа|ssh-keygen -f ~/.ssh/my_key|
|-C|Комментарий (обычно e-mail или описание)|ssh-keygen -t rsa -C "myemail@example.com"|
|-N|Парольная фраза (можно задать пустую "")|ssh-keygen -t rsa -N ""|

### **Дополнительные ключи**

|**Ключ**|**Назначение**|**Пример использования**|
|---|---|---|
|-q|Тихий режим (не выводить лишнюю информацию)|ssh-keygen -q -t rsa|
|-y|Преобразует закрытый ключ в открытый|ssh-keygen -y -f ~/.ssh/id_rsa > ~/.ssh/id_rsa.pub|
|-p|Изменить парольную фразу существующего ключа|ssh-keygen -p -f ~/.ssh/id_rsa|
|-l|Показать fingerprint (отпечаток) ключа|ssh-keygen -l -f ~/.ssh/id_rsa.pub|
|-e|Экспортировать открытый ключ в формате RFC4716|ssh-keygen -e -f ~/.ssh/id_rsa.pub > id_rsa_rfc.pub|
|-m|Формат ключа при экспорте или импорте (RFC4716 или PKCS8)|ssh-keygen -e -m PKCS8 -f id_rsa.pub|

### **Примеры**

**Создание RSA-ключа с 4096 битами и комментарием:**

```
ssh-keygen -t rsa -b 4096 -C "user@example.com"
```

**Создание ED25519-ключа без парольной фразы:**

```
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
```

**Изменить парольную фразу у ключа:**

```
ssh-keygen -p -f ~/.ssh/id_rsa
```

**Вывести открытый ключ из приватного файла:**

```
ssh-keygen -y -f ~/.ssh/id_rsa > ~/.ssh/id_rsa.pub
```

**Посмотреть отпечаток открытого ключа:**

```
ssh-keygen -l -f ~/.ssh/id_rsa.pub
```
