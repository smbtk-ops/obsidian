# Очистка диска на сервере с Jenkins

Сервер: `192.168.88.25`

---

## 1. Диагностика

Проверка занятого места:

```bash
df -h /
du -h --max-depth=1 / 2>/dev/null | sort -rh | head -15
du -h --max-depth=1 /var/lib/jenkins 2>/dev/null | sort -rh | head -15
```

---

## 2. Очистка ws-cleanup мусора Jenkins

Jenkins плагин `ws-cleanup` при неудачной очистке оставляет папки `workspace_ws-cleanup_*` внутри job-ов. Они могут накапливаться и занимать гигабайты.

### 2.1. Поиск и оценка размера

```bash
find /var/lib/jenkins/jobs/ -maxdepth 2 -name 'workspace_ws-cleanup_*' -type d | wc -l
du -sh /var/lib/jenkins/jobs/*/workspace_ws-cleanup_* 2>/dev/null
```

### 2.2. Удаление

```bash
find /var/lib/jenkins/jobs/ -maxdepth 2 -name 'workspace_ws-cleanup_*' -type d -exec rm -rf {} +
```

---

## 3. Очистка workspace Jenkins

`/var/lib/jenkins/workspace/` — склонированные репозитории, используемые при сборках. Удаление безопасно — при следующем билде Jenkins заново склонирует нужный репозиторий.

```bash
rm -rf /var/lib/jenkins/workspace/*
```

---

## 4. Ограничение systemd journal логов

По умолчанию journal может расти неограниченно.

### 4.1. Установка постоянного лимита

В `/etc/systemd/journald.conf`:

```bash
sed -i 's/^#\?SystemMaxUse=.*/SystemMaxUse=1G/' /etc/systemd/journald.conf
```

### 4.2. Применение и очистка

```bash
systemctl restart systemd-journald
journalctl --vacuum-size=1G
```

### 4.3. Проверка

```bash
journalctl --disk-usage
```
