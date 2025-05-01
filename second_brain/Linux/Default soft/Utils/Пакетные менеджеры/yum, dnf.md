
## **Менеджер пакетов RHEL (rpm, yum, dnf)**

### **Особенности форматов пакетов**

|**Формат**|**Описание**|
|---|---|
|**deb**|ar-архив с двумя tar.xz: один для метаданных, другой для данных.|
|**rpm**|Содержит cpio-архив с метаданными (заголовок) и cpio-архив с данными (gzip-сжатие).|

Извлечение файлов из rpm-пакета:
```
rpm2cpio package.rpm | cpio -i -d
```

### **Основные команды rpm**

|**Действие**|**Команда**|**Примечание**|
|---|---|---|
|Установить пакет|rpm -i package.rpm|Аналог dpkg -i.|
|Удалить пакет|rpm -e package|В отличие от deb, используется -e.|
|Показать все установленные пакеты|rpm -qa|q — query, a — all.|
|Список файлов пакета|rpm -ql package||
|Информация о пакете|rpm -qi package||
|Инфо о **неустановленном** пакете|rpm -qip package.rpm|Добавляется p — package file.|

### **Аналогия yum и apt**

|**Действие**|**apt**|**yum**|
|---|---|---|
|Установить пакет|apt install PACKAGE|yum install PACKAGE|
|Удалить пакет|apt remove PACKAGE|yum remove PACKAGE|
|Обновить пакет|apt install --upgrade PACKAGE|yum update PACKAGE|
|Обновить все пакеты|apt upgrade|yum update или yum upgrade|
|Инфо о пакете|apt show PACKAGE|yum info PACKAGE|
|Обновить список пакетов|apt update|yum updateinfo|

### **Пример использования yum updateinfo**

```
yum updateinfo
```

Вывод примера:

```
Updates Information Summary: available
    15 Security notice(s)
         3 Important Security notice(s)
         7 Moderate Security notice(s)
         2 Low Security notice(s)
    13 Bugfix notice(s)
     7 Enhancement notice(s)
     5 other notice(s)
```


### **Добавление репозитория (Простой способ)**

Пример установки EPEL-репозитория:
```
yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
```

### **Добавление репозитория (Ручной способ)**

Создать файл:
```
cat > /etc/yum.repos.d/Webtatic.repo << EOF
[webtatic]
name=Webtatic repo
baseurl=http://repo.webtatic.com/yum/centos/5/SRPMS/
enabled=1
gpgcheck=1
gpgkey=http://repo.webtatic.com/yum/RPM-GPG-KEY-webtatic-andy
EOF
```

Импортировать ключ:
```
rpm --import http://repo.webtatic.com/yum/RPM-GPG-KEY-webtatic-andy
```

Проверить:
```
yum updateinfo
```

### **Управление репозиториями**

|**Действие**|**Команда**|
|---|---|
|Список установленных репо|yum repolist|
|Удалить репозиторий|Удалить файл из /etc/yum.repos.d/ или удалить пакет через yum remove/rpm -e.|

### **dnf: Замена yum**
- DNF полностью совместим с yum-командами.
- Основная цель — повышение производительности и переписывание кода.
- Использует те же пакеты и синтаксис:

```
dnf install PACKAGE
dnf update
dnf remove PACKAGE
```

