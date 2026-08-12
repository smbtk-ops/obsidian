## 1. Проблема

Containerd накапливает неиспользуемые образы контейнеров в `/var/lib/containerd/`. Со временем корневой раздел заполняется. По дефолту kubelet начинает GC образов только при 85% заполнения диска.

---

## 2. Диагностика

### 2.1. Проверка места на ноде

```bash
df -h /
```

### 2.2. Поиск крупных директорий

```bash
sudo du -h --max-depth=1 /var/lib | sort -rh | head -10
```

### 2.3. Размер containerd

```bash
sudo du -sh /var/lib/containerd
```

### 2.4. Список неиспользуемых образов

```bash
sudo crictl images
```

---

## 3. Ручная очистка образов

Удаляет образы, не используемые запущенными контейнерами. Работающие поды не затрагиваются — при следующем деплое образ скачается заново.

```bash
sudo crictl --timeout 300s rmi --prune
```

| Параметр | Описание |
|----------|----------|
| `--timeout 300s` | Увеличенный таймаут, без него часто `context deadline exceeded` |
| `--prune` | Удалить только неиспользуемые образы |

---

## 4. Настройка автоматического GC в kubelet

### 4.1. Параметры

| Параметр | Дефолт | Описание |
|----------|--------|----------|
| `imageGCHighThresholdPercent` | 85 | При каком % заполнения диска начинать чистку |
| `imageGCLowThresholdPercent` | 80 | До какого % чистить |

### 4.2. Изменение порогов

Редактируем конфиг kubelet:

```bash
sudo vim /var/lib/kubelet/config.yaml
```

Добавляем параметры:

```yaml
imageGCHighThresholdPercent: 70
imageGCLowThresholdPercent: 50
```

Перезапуск kubelet:

```bash
sudo systemctl restart kubelet
```

Проверка:

```bash
sudo systemctl is-active kubelet
grep -E 'imageGC(High|Low)' /var/lib/kubelet/config.yaml
```

### 4.3. Применение на нескольких нодах

```bash
for ip in 181 182 183; do
  ssh ansible@10.10.1.$ip "\
    sudo sed -i '/^imageMinimumGCAge/a imageGCHighThresholdPercent: 70\nimageGCLowThresholdPercent: 50' /var/lib/kubelet/config.yaml && \
    sudo systemctl restart kubelet
done
```

---

## 5. Важно

- `crictl rmi --prune` безопасна — удаляет только образы без запущенных контейнеров
- После очистки при рестарте/деплое пода образ скачивается заново
- При изменении конфига через kubespray — параметры могут быть перезаписаны при следующем прогоне, нужно дублировать в inventory
- Путь к конфигу kubelet: `/var/lib/kubelet/config.yaml`