Команда chown (change owner) в Linux используется для изменения владельца и/или группы файла или директории.

### **Основные ключи команды chown**

| **Ключ**                               | **Описание**                                                    | **Пример использования**                                |
| -------------------------------------- | --------------------------------------------------------------- | ------------------------------------------------------- |
| -c                                     | Показать изменения для каждого обработанного файла              | chown -c user file.txt                                  |
| -f                                     | Игнорировать сообщения об ошибках                               | chown -f user file.txt                                  |
| -v                                     | Вывести подробную информацию о каждом файле                     | chown -v user file.txt                                  |
| -R                                     | Рекурсивно менять владельца для директорий и их содержимого     | chown -R user folder/                                   |
| --dereference                          | Изменять владельца цели ссылки, а не самой символической ссылки | chown --dereference user linkname                       |
| --no-dereference (-h)                  | Изменять владельца самой символической ссылки                   | chown -h user symlink                                   |
| --from=CURRENT_OWNER:<br>CURRENT_GROUP | Менять владельца только если текущий владелец/группа совпадает  | chown --from=olduser:oldgroup newuser:newgroup file.txt |
| --reference=FILE                       | Установить владельца и группу по образцу другого файла          | chown --reference=reference.txt file.txt                |

---
### **Формат команды**
```
chown [опции] [новый_владелец][:новая_группа] файл/директория
```
- новый_владелец — имя нового пользователя
- новая_группа — (необязательно) имя новой группы
- : — разделяет владельца и группу

### **Примеры использования команды chown**

#### **1. Изменить владельца файла**
```
chown username file.txt
```

#### **2. Изменить владельца и группу файла**
```
chown username:groupname file.txt
```

#### **3. Рекурсивное изменение владельца в директории**
```
chown -R username folder/
```

#### **4. Изменить только группу файла**
```
chown :groupname file.txt
```

#### **5. Изменить владельца файла на владельца другого файла**
```
chown --reference=anotherfile.txt file.txt
```

#### **6. Изменить владельца только если совпадает старый**
```
chown --from=olduser:oldgroup newuser:newgroup file.txt
```
Изменит владельца и группу только если текущий владелец olduser и группа oldgroup.
