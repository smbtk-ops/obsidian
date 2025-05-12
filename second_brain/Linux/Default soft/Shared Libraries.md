## **Что такое общие библиотеки**

|**Тип**|**Описание**|
|---|---|
|**Динамические библиотеки**|Подгружаются во время выполнения программы. Экономят память: могут использоваться несколькими процессами.|
|**Статические библиотеки**|Встраиваются в бинарник при компиляции. Получившийся файл больше по размеру. Часто используется в Go.|
|**Примеры библиотек**|libc, glibc, musl, Qt, GTK, boost и т.п.|
|**Типичные пути**|/lib, /lib64, /usr/lib, /usr/lib64, /usr/lib32|
|**Файл конфигурации**|/etc/ld.so.conf — добавляет нестандартные пути поиска библиотек|
|**Версионирование**|Возможны имена с версиями: libssl.so.1.1, libPocoCrypto.so.81|

## **Инструменты работы с библиотеками**

### **ldd — просмотр зависимостей бинарника**
```
ldd /usr/bin/curl
```

|**Полезно**|**Описание**|
|---|---|
|=> /usr/lib/...|Путь к найденной библиотеке|
|=> not found|Библиотека не найдена — возможна ошибка запуска|

### **LD_LIBRARY_PATH — временное добавление путей**
```
export LD_LIBRARY_PATH=/home/user/lib:/opt/lib
```
> Добавляет пути для поиска библиотек на время текущей сессии.

---
### **ldconfig — обновление кэша библиотек**
```
sudo ldconfig
```

|**Назначение**|**Описание**|
|---|---|
|Кеширует пути|Используется после изменений в /etc/ld.so.conf|
|Обновляет симлинки|Для .so файлов с версиями|

## **Типичная ошибка при запуске**
```
$ ./TogglDesktop
error while loading shared libraries: libPocoCrypto.so.81: cannot open shared object file: No such file or directory
```

📌 Решения:

- Установить нужную библиотеку
- Добавить путь к ней в LD_LIBRARY_PATH
- Либо прописать путь в /etc/ld.so.conf + ldconfig

---
## **Иерархия директорий библиотек**

|**Директория**|**Назначение**|
|---|---|
|/lib, /lib64|Системные библиотеки, доступные в ранней загрузке|
|/usr/lib, /usr/lib64|Основной каталог для большинства библиотек|
|/usr/lib32|32-битные библиотеки (для совместимости)|

## **Мини-шпаргалка**

|**Задача**|**Команда / Переменная**|
|---|---|
|Проверить зависимости бинаря|ldd /path/to/bin|
|Временно указать путь к .so|export LD_LIBRARY_PATH=/your/custom/path|
|Постоянно указать путь к .so|Добавить путь в /etc/ld.so.conf + ldconfig|
|Обновить кэш библиотек|sudo ldconfig|

----

- [Shared Libraries (linux docs)](https://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html)
- [Библиотеки Linux](https://losst.pro/biblioteki-linux)
- [Difference between Static and Shared libraries](https://www.geeksforgeeks.org/difference-between-static-and-shared-libraries/)
- [C standard library (wikipedia eng)](https://en.wikipedia.org/wiki/C_standard_library)
  
  -----
