# **Команда ls в Linux**

Команда ls используется для отображения списка файлов и директорий.
### **Основные ключи команды ls**

|**Ключ**|**Описание**|**Пример использования**|
|---|---|---|
|-l|Длинный формат вывода (права, владелец, размер, дата)|ls -l|
|-a|Показывать все файлы, включая скрытые (. начинаются)|ls -a|
|-h|Читаемый формат размера файлов (например, KB, MB) — используется с -l|ls -lh|
|-R|Рекурсивный вывод содержимого всех поддиректорий|ls -R|
|-S|Сортировка по размеру файлов (от большего к меньшему)|ls -S|
|-t|Сортировка по времени модификации (от новых к старым)|ls -t|
|-r|Инвертировать порядок сортировки|ls -lr|
|-d|Показать только сами каталоги, а не их содержимое|ls -d */|
|-1|Вывод по одному файлу на строку|ls -1|
|-i|Показать номер inode для каждого файла|ls -i|
|--color=auto|Раскрашивать вывод по типу файлов (обычно по умолчанию)|ls --color=auto|

---
### **Полезные комбинации ключей**

|**Команда**|**Что делает**|
|---|---|
|ls -la|Показывает все файлы и каталоги, включая скрытые, в длинном формате|
|ls -lhS|Список файлов по размеру в человекочитаемом формате|
|ls -ltr|Список файлов по дате изменения, от старых к новым|
|ls -R /etc|Рекурсивно показать все содержимое каталога /etc|
|ls -d */|Показать только каталоги в текущей папке|

# Идентификация и управление типами файлов в Linux

В Linux существует несколько типов файлов, каждый из которых обозначается специальным символом в выводе `ls -l`. Знание типов файлов помогает эффективно управлять системой.

### **Основные типы файлов:**

|Тип файла|Обозначение|Описание|
|---|---|---|
|Обычные файлы|`-`|Стандартные файлы: документы, скрипты, исполняемые файлы.|
|Каталоги|`d`|Папки, содержащие другие файлы и каталоги.|
|Символические ссылки|`l`|Указатели на другие файлы или каталоги.|
|Файлы символьных устройств|`c`|Устройства, работающие с потоками байтов (например, клавиатуры).|
|Файлы блочных устройств|`b`|Устройства, работающие с блоками данных (диски).|
|Сокеты|`s`|Способ обмена данными между процессами.|
|Именованные каналы (FIFO)|`p`|Способ передачи данных между процессами в режиме "первым пришёл — первым ушёл".|

---

# Заметка

- Файлы устройств (`c`, `b`), сокеты (`s`) и FIFO (`p`) обычно создаются системой или суперпользователем через `mknod`.