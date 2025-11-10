## Назначение скрипта

Скрипт предназначен для автоматической очистки старых Docker-тегов в **Nexus Repository Manager 3** в заданном репозитории.  
Он оставляет заданное количество последних тегов (по дате изменения), а все остальные **удаляет вместе с их ассетами**.

---
## Основные изменения и улучшения

|Раздел|Что сделано|Причина|
|---|---|---|
|**Чтение конфигурации**|Теперь креды (`NEXUS_USER`, `NEXUS_PASS`) читаются из внешнего файла `/opt/nexus/sonatype-work/nexus3/etc/scripts.env`|Чтобы не хранить пароли в коде|
|**Аутентификация**|Авторизация выполняется через HTTP Basic (`Authorization: Basic ...`)|Универсальный метод для REST API Nexus|
|**Постраничная загрузка ассетов**|Добавлен цикл `while (true)` с `continuationToken`|Nexus возвращает ассеты постранично|
|**Группировка ассетов**|Ассеты группируются по имени образа (`imageName`)|Чтобы вычислить старые теги для каждого образа|
|**Сортировка по дате**|Теги сортируются по `lastModified`|Гарантированно сохраняются самые свежие версии|
|**Логирование**|Подробные сообщения через `log.info`|Видно, какие теги удаляются и какие остаются|
|**Безопасное удаление**|Добавлена проверка кода ответа (204, 404)|Для стабильной работы и отладки|

---
## Структура окружения

Файл `scripts.env` должен содержать:
```bash
NEXUS_USER=admin
NEXUS_PASS=SuperSecretPassword
```

Расположить по пути:
```
/opt/nexus/sonatype-work/nexus3/etc/scripts.env
```

Права на файл:
```bash
chmod 600 /opt/nexus/sonatype-work/nexus3/etc/scripts.env
chown nexus:nexus /opt/nexus/sonatype-work/nexus3/etc/scripts.env
```

---
## Как использовать в Nexus

1. Зайди в Nexus под администратором.
2. Перейди в **Administration → System → Scripts**.
3. Создай новый скрипт:
    
    - **Name:** `cleanup-docker-keep-5`
    - **Type:** `groovy`
    - **Content:** вставь полный код из блока ниже.
4. Сохрани и запусти задачу через **Tasks → Create task → “Run script”**.
5. Укажи:
    - **Script name:** `cleanup-docker-keep-5`
    - **Schedule type:** например, _Daily_ или _Manual run_.
6. Запусти задачу вручную и проверь лог `/opt/nexus/sonatype-work/nexus3/log/tasks/script-*.log`.

---

## Настройки, которые можно менять

|Параметр|По умолчанию|Описание|
|---|---|---|
|`baseUrl`|`http://127.0.0.1:8081/service/rest`|REST API Nexus|
|`repository`|`docker-images-hosted`|Имя репозитория|
|`keepCount`|`5`|Сколько тегов хранить|
|`scripts.env`|`/opt/nexus/sonatype-work/nexus3/etc/scripts.env`|Путь к кредам|

---
##  Полный актуальный код
```groovy
// === Конфигурация ===
def baseUrl    = "http://127.0.0.1:8081/service/rest"
def repository = "docker-images-hosted"

// === Читаем креды из внешнего файла ===
def envFile = new File("/opt/nexus/sonatype-work/nexus3/etc/scripts.env")
if (!envFile.exists()) {
    log.error("Файл с переменными окружения не найден: ${envFile}")
    return
}

def creds = [:]
envFile.eachLine { line ->
    if (line && !line.startsWith("#") && line.contains("=")) {
        def (key, value) = line.split("=", 2)
        creds[key.trim()] = value.trim()
    }
}

def user = creds["NEXUS_USER"]
def password = creds["NEXUS_PASS"]

if (!user || !password) {
    log.error("Не удалось получить креды из scripts.env")
    return
}

log.info("→ Использую креды из scripts.env (${user})")

// === Количество тегов, которые нужно оставить ===
def keepCount  = 5

// === Вспомогательные функции ===
def getJson = { endpoint ->
    def url = new java.net.URL("${baseUrl}${endpoint}")
    def conn = (java.net.HttpURLConnection) url.openConnection()
    conn.setRequestMethod("GET")
    conn.setRequestProperty("Authorization", "Basic " + "${user}:${password}".bytes.encodeBase64().toString())
    conn.setRequestProperty("Accept", "application/json")
    conn.connect()
    if (conn.responseCode != 200) {
        throw new RuntimeException("Ошибка GET ${endpoint}: ${conn.responseCode}")
    }
    def text = conn.inputStream.text
    new groovy.json.JsonSlurper().parseText(text)
}

def deleteAsset = { id ->
    def url = new java.net.URL("${baseUrl}/v1/assets/${id}")
    def conn = (java.net.HttpURLConnection) url.openConnection()
    conn.setRequestMethod("DELETE")
    conn.setRequestProperty("Authorization", "Basic " + "${user}:${password}".bytes.encodeBase64().toString())
    conn.connect()
    def code = conn.responseCode
    if (code == 204) {
        log.info("Удалён ассет ${id}")
    } else if (code == 404) {
        log.info("Не найден ассет ${id}")
    } else {
        log.warn("Ошибка удаления ${id}: ${code}")
    }
}

// === Получаем все ассеты ===
def allAssets = []
def token = null
log.info("→ Получение ассетов из '${repository}'...")

while (true) {
    def url = "/v1/assets?repository=${repository}" + (token ? "&continuationToken=${token}" : "")
    def result = getJson(url)
    allAssets.addAll(result.items)
    token = result.continuationToken
    if (!token) break
}

log.info("Всего ассетов: ${allAssets.size()}")

// === Группировка по имени образа ===
def grouped = [:]
for (asset in allAssets) {
    def path = asset.path ?: ""
    def imageName = "unknown"
    if (path.contains("/")) {
        imageName = path.substring(0, path.lastIndexOf("/"))
    }
    if (!grouped.containsKey(imageName)) {
        grouped[imageName] = []
    }
    grouped[imageName] << asset
}

// === Определяем старые теги ===
def toDelete = []
for (entry in grouped) {
    def imageName = entry.key
    def assets = entry.value.findAll { it.path.contains("/manifests/") && !it.path.contains("sha256:") }
    if (assets.isEmpty()) continue

    def sorted = assets.sort { a, b -> b.lastModified <=> a.lastModified }
    if (sorted.size() > keepCount) {
        def oldManifests = sorted.drop(keepCount)
        log.info("Образ '${imageName}': всего тегов=${sorted.size()}, сохраняем=${keepCount}, удаляем=${oldManifests.size()}")
        toDelete.addAll(oldManifests)
    }
}

log.info("Всего manifest-файлов к удалению: ${toDelete.size()}")

// === Удаляем ===
for (asset in toDelete) {
    def id = asset.id.tokenize("/").last()
    log.info("Удаляю ${asset.path} (${id})")
    deleteAsset(id)
}

log.info("Очистка завершена. Оставлено по ${keepCount} тегов на каждый Docker-образ.")
```

---

## Проверка результата

После запуска задачи:
1. Перейди в **Tasks → Cleanup logs**.
2. Убедись, что строки вида  
    `Образ 'slot-front': всего тегов=12, сохраняем=5, удаляем=7` присутствуют.
3. Открой **Browse → docker-images-hosted** и проверь, что остались только 5 последних тегов на образ.
