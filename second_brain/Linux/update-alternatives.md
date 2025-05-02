### **Что такое альтернатива в контексте update-alternatives?**

**Альтернатива** — это механизм в Linux (особенно в Debian/Ubuntu), позволяющий управлять **разными версиями одной и той же команды или программы**.

#### **Пример:**

У тебя может быть установлено несколько редакторов:
- /usr/bin/vim
- /usr/bin/nano
- /usr/bin/micro
И у всех у них может быть общее имя: editor

В этом случае ты можешь управлять, **какой именно редактор будет запускаться, когда ты вводишь просто editor**, через систему update-alternatives.

### **Как это работает?**

За каждой “альтернативой” стоит **символическая ссылка (symlink)**.

Например:
```
editor -> /etc/alternatives/editor -> /usr/bin/nano
```

То есть:
1. /usr/bin/editor — это основная точка вызова.
2. /etc/alternatives/editor — это промежуточная ссылка, которую update-alternatives управляет.
3. /usr/bin/nano — это фактический исполняемый файл.

### **Зачем это нужно?**
- Выбор версии интерпретатора Python: python2 vs python3
- Переключение между gcc и clang
- Использование разных JVM: openjdk-11 vs openjdk-17
- Настройка editor по умолчанию

### **Пример из жизни**

#### **Установлены:**
- /usr/bin/python2.7
- /usr/bin/python3.10

#### **Хочешь, чтобы python вызывал python3.10 :**

```
sudo update-alternatives --install /usr/bin/python python /usr/bin/python3.10 1
sudo update-alternatives --install /usr/bin/python python /usr/bin/python2.7 2
sudo update-alternatives --config python
```
Ты увидишь список, выберешь нужное — и всё, теперь python будет вести на нужную версию.

### **Ключи команды update-alternatives**

|**Ключ**|**Описание**|**Пример**|
|---|---|---|
|--install|Добавить новую альтернативу|sudo update-alternatives --install /usr/bin/editor editor /usr/bin/vim 100|
|--remove|Удалить альтернативу|sudo update-alternatives --remove editor /usr/bin/vim|
|--remove-all|Удалить все альтернативы по имени|sudo update-alternatives --remove-all editor|
|--auto|Включить автоматический режим выбора альтернативы|sudo update-alternatives --auto editor|
|--set|Вручную установить конкретную альтернативу|sudo update-alternatives --set editor /usr/bin/nano|
|--config|Запустить интерактивный выбор альтернативы|sudo update-alternatives --config editor|
|--display|Показать информацию об альтернативе|update-alternatives --display editor|
|--get-selections|Показать все альтернативы и выбранные значения|update-alternatives --get-selections|
|--query|Вывести информацию о конкретной альтернативе|update-alternatives --query editor|
|--list|Показать все пути, установленные как альтернативы|update-alternatives --list editor|
|--help|Показать справку|update-alternatives --help|
|--version|Показать версию утилиты|update-alternatives --version|

### **Пояснение к ключевым командам:**

#### **--install**
```
sudo update-alternatives --install /usr/bin/python python /usr/bin/python3.10 1
```
> Добавляет альтернативу python, ссылающуюся на /usr/bin/python3.10, приоритет 1.

#### **--config**
```
sudo update-alternatives --config python
```
> Откроется список всех альтернатив python, где можно выбрать нужную вручную.

#### **--set**
```
sudo update-alternatives --set python /usr/bin/python3.10
```
> Устанавливает python на конкретную версию.


### **Как узнать, какие “общие имена” (группы альтернатив) уже существуют?**

```
update-alternatives --get-selections
```

**Пример вывода:**
```
editor                      auto    /usr/bin/nano
awk                         auto    /usr/bin/gawk
x-www-browser               auto    /usr/bin/firefox
```
Здесь editor, awk, x-www-browser — это те самые “общие имена” (группы альтернатив).

### **Как посмотреть, какие конкретные пути входят в группу?**

```
update-alternatives --display editor
```

**Пример:**
```
editor - auto mode
  link best version is /usr/bin/vim.basic
  link currently points to /usr/bin/vim.basic
  link editor is /usr/bin/editor
  slave editor.1.gz is /usr/share/man/man1/editor.1.gz
/usr/bin/nano - priority 40
/usr/bin/vim.basic - priority 30
```
То есть ты видишь, что editor — это общее имя, а к нему подключены nano и vim.basic.

### **Как создать свою группу альтернатив?**

```
sudo update-alternatives --install /usr/bin/myeditor myeditor /usr/bin/vim 100
sudo update-alternatives --install /usr/bin/myeditor myeditor /usr/bin/nano 50
```
Теперь у тебя появится **группа с именем myeditor**, которую можно настраивать и переключать:
```
sudo update-alternatives --config myeditor
```
