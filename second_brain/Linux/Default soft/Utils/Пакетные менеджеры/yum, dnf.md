
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

| **Действие**                      | **Команда**          | **Примечание**                     |
| --------------------------------- | -------------------- | ---------------------------------- |
| Установить пакет                  | rpm -i package.rpm   | Аналог dpkg -i.                    |
| Удалить пакет                     | rpm -e package       | В отличие от deb, используется -e. |
| Показать все установленные пакеты | rpm -qa              | q — query, a — all.                |
| Список файлов пакета              | rpm -ql package      |                                    |
| Информация о пакете               | rpm -qi package      |                                    |
| Инфо о **неустановленном** пакете | rpm -qip package.rpm | Добавляется p — package file.      |

### **Основные ключи команды yum**

| **Ключ / Команда**      | **Назначение**                                               |
| ----------------------- | ------------------------------------------------------------ |
| install <пакет>         | Устанавливает указанный пакет                                |
| remove <пакет>          | Удаляет указанный пакет и его зависимости                    |
| update                  | Обновляет все установленные пакеты                           |
| update <пакет>          | Обновляет только указанный пакет                             |
| upgrade                 | Обновляет пакеты с удалением устаревших                      |
| list                    | Показывает список доступных и установленных пакетов          |
| list installed          | Список всех установленных пакетов                            |
| list available          | Список всех доступных пакетов в репозиториях                 |
| info <пакет>            | Подробная информация о пакете                                |
| search <строка>         | Поиск пакетов по ключевому слову                             |
| provides <файл>         | Показывает, какой пакет содержит указанный файл              |
| clean all               | Удаляет все кэшированные данные                              |
| makecache               | Создаёт локальный кэш метаданных                             |
| repolist                | Показывает включённые репозитории                            |
| groupinstall "<группа>" | Устанавливает группу пакетов (например, “Development Tools”) |
| groupremove "<группа>"  | Удаляет группу пакетов                                       |

### **Часто используемые флаги команды yum**

| **Флаг**             | **Назначение**                                                                       |
| -------------------- | ------------------------------------------------------------------------------------ |
| -y                   | Автоматически отвечает «yes» на все запросы (полностью без участия пользователя)     |
| -q                   | Тихий режим (минимум вывода)                                                         |
| -v                   | Подробный вывод (verbose)                                                            |
| --enablerepo=<repo>  | Включает указанный репозиторий для выполнения текущей операции                       |
| --disablerepo=<repo> | Отключает указанный репозиторий                                                      |
| --nogpgcheck         | Отключает проверку GPG-подписи пакетов                                               |
| --exclude=<пакет>    | Исключает пакет(ы) из установки или обновления                                       |
| --showduplicates     | Показывает все доступные версии пакета                                               |
| --downloadonly       | Скачивает пакет без установки                                                        |
| --skip-broken        | Пропускает пакеты с неудовлетворёнными зависимостями                                 |
| --security           | Ограничивает обновления только до тех, что отмечены как обновления безопасности      |
| --setopt=<параметр>  | Устанавливает опцию для текущей команды (например, proxy, timeout и др.)             |
| --installroot=<dir>  | Устанавливает альтернативный корень для установки (например, для chroot/контейнеров) |

### **Примеры использования команды yum**

#### **Установка пакета**
```
yum install nginx
```

#### **Удаление пакета**
```
yum remove nginx
```

#### **Обновление всех пакетов**
```
yum update
```

#### **Информация о пакете**
```
yum info httpd
```

#### **Поиск пакета по ключевому слову**
```
yum search ftp
```

#### **Очистка кэша**
```
yum clean all
```

#### **Установка группы пакетов**
```
yum groupinstall "Development Tools"
```

#### **Просмотр активных репозиториев**
```
yum repolist
```

#### **Создание локального кэша метаданных**
```
yum makecache
```

### **Примеры использования флагов**

#### **Установить пакет без подтверждений**
```
yum install -y vim
```

#### **Обновить систему с отключением конкретного репозитория**
```
yum update --disablerepo=epel
```

#### **Установить пакет только из определённого репозитория**
```
yum install --enablerepo=epel htop
```

#### **Скачать пакет без установки**
```
yum install --downloadonly nginx
```

#### **Пропустить проблемные зависимости при обновлении**
```
yum update --skip-broken
```

#### **Установить пакет без проверки GPG-подписи**
```
yum install --nogpgcheck somepackage.rpm
```

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

----

- [How To Extract an RPM Package Files Without Installing It](https://www.cyberciti.biz/tips/how-to-extract-an-rpm-package-without-installing-it.html)
- [20 Practical Examples of RPM Commands in Linux](https://www.tecmint.com/20-practical-examples-of-rpm-commands-in-linux/)
- [Yum, шпаргалка (habr)](https://habr.com/ru/post/301292/)
- [How to add a Yum repository](https://www.redhat.com/sysadmin/add-yum-repository)
- [DNF vs YUM: Differences between DNF and YUM Package Managers](https://bytexd.com/dnf-vs-yum/)