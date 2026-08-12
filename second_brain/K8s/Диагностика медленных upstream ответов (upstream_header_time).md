# Диагностика медленных upstream ответов (upstream_header_time)

Руководство по поиску и устранению проблем, когда `upstream_response_time >= 1s`, но `upstream_connect_time = 0`. TCP-соединение устанавливается мгновенно, но бэкенд долго обрабатывает запрос.

Связанный документ: [[Диагностика TCP SYN ретрансмитов (upstream_connect_time)]] — для случаев когда `upstream_connect_time >= 1s`.

---

## 1. Определение типа проблемы

### 1.1. Ключевые метрики nginx

| Метрика | Что показывает |
|---------|---------------|
| `upstream_connect_time` | Время установления TCP-соединения с бэкендом |
| `upstream_header_time` | Время от отправки запроса до получения первого байта ответа |
| `upstream_response_time` | Полное время обработки запроса бэкендом |

### 1.2. Два разных сценария

| Сценарий | Признаки | Причина |
|----------|---------|--------|
| Сетевая проблема | `upstream_connect_time >= 1s` | TCP SYN retransmit, conntrack overflow, backlog overflow |
| Бэкенд тормозит | `upstream_connect_time = 0`, `upstream_header_time >= 1s` | CFS throttling, lock contention, перегрузка I/O |

Этот документ покрывает **второй сценарий**.

### 1.3. Фильтрация в Datadog

```
# медленные ответы бэкенда (не сеть)
source:nginx.<service> @upstream_response_time:>=1 @upstream_connect_time:<0.01
```

---

## 2. Причины медленного upstream_header_time

### 2.1. CFS CPU Throttling (самая частая причина в K8s)

Kubernetes использует Linux CFS (Completely Fair Scheduler) для enforce CPU limits. Когда контейнер превышает CPU квоту за CFS-период (100ms), ядро **замораживает все процессы** в контейнере до следующего периода.

Для пода с `limits.cpu: "6"` квота = 600ms CPU-времени на каждые 100ms стенного времени. Если Go runtime с десятками горутин, GC и I/O completion кратковременно требует больше — весь контейнер замерзает.

Эффект каскадный:
```
CFS throttle → часть запросов замедляется
  → горутины живут дольше → больше горутин одновременно
    → больше CPU demand → больше throttling
      → DB соединения заняты дольше → lock wait растёт
        → ещё больше горутин → положительная обратная связь
```

### 2.2. InnoDB Row Lock Contention

Если бэкенд делает SELECT и UPDATE на одну таблицу одновременно (особенно на одном MySQL-сервере), SELECT ждёт снятия row lock от UPDATE. При высоком RPS это добавляет сотни миллисекунд.

### 2.3. Goroutine Saturation (Go-специфично)

Go HTTP server принимает запрос (TCP connection OK, `upstream_connect_time = 0`), но все горутины заняты I/O. Новый запрос стоит в очереди пока горутина не освободится. Время ожидания = разница между `upstream_response_time` и внутренним duration приложения.

### 2.4. Внешние зависимости

Синхронные вызовы на критическом пути (Kafka sync write, RabbitMQ blocking publish, HTTP-вызовы к другим сервисам) без таймаутов. Если зависимость тормозит — запрос висит.

---

## 3. Диагностика: CFS CPU Throttling

### 3.1. Проверка cpu.stat

```bash
# для всех подов deployment
for POD in $(kubectl get pods -n <namespace> -l app=<app> -o jsonpath='{.items[*].metadata.name}'); do
    echo "=== $POD ==="
    kubectl exec -n <namespace> "$POD" -- \
        grep -E 'nr_periods|nr_throttled|throttled_usec' /sys/fs/cgroup/cpu.stat
done
```

Пример вывода:
```
nr_periods 3966915
nr_throttled 3901
throttled_usec 1043141141
```

| Поле | Описание |
|------|----------|
| `nr_periods` | Общее количество CFS-периодов (каждый 100ms) |
| `nr_throttled` | Сколько раз контейнер был заморожен |
| `throttled_usec` | Суммарное время заморозки (микросекунды) |

### 3.2. Оценка throttle rate

```
throttle_rate = nr_throttled / nr_periods × 100%
avg_throttle_duration = throttled_usec / nr_throttled / 1000 (ms)
```

| throttle_rate | Оценка |
|--------------|--------|
| < 0.01% | Норма |
| 0.01% - 0.1% | Заметные задержки под нагрузкой |
| > 0.1% | Серьёзная проблема, нужно менять limits |

### 3.3. Мониторинг в Datadog

```
container.cpu.throttled.periods{kube_namespace:<ns>}
container.cpu.throttled.time{kube_namespace:<ns>}
```

Эти метрики собираются из того же `cpu.stat` но с привязкой к временной шкале — видно точное время throttling.

### 3.4. Решение

**Вариант A (рекомендуется)**: убрать CPU limits, оставить requests:
```yaml
resources:
  requests:
    cpu: "6"
    memory: "256Mi"
  limits:
    memory: "256Mi"
    # cpu limit убран — нет CFS throttling
```

QoS станет Burstable. HPA с `averageUtilization` продолжит работать — он считает процент от `requests.cpu`, не от `limits.cpu`.

**Вариант B**: увеличить limit с запасом:
```yaml
resources:
  requests:
    cpu: "6"
  limits:
    cpu: "10"  # запас 60% для всплесков
```

---

## 4. Диагностика: InnoDB Row Lock Contention

### 4.1. Проверка метрик MySQL

```sql
-- количество ожиданий lock (накопительный счётчик)
SHOW GLOBAL STATUS LIKE 'Innodb_row_lock_waits';

-- среднее время ожидания lock (ms)
SHOW GLOBAL STATUS LIKE 'Innodb_row_lock_time_avg';

-- текущее количество соединений
SHOW GLOBAL STATUS LIKE 'Threads_connected';

-- текущие активные запросы
SHOW GLOBAL STATUS LIKE 'Threads_running';

-- slow queries
SHOW GLOBAL STATUS LIKE 'Slow_queries';
```

### 4.2. Оценка

| Метрика | Норма | Проблема |
|---------|-------|---------|
| Innodb_row_lock_waits | < 1000 | > 100,000 |
| Innodb_row_lock_time_avg | < 5ms | > 20ms |
| Threads_connected | < 200 | > 500 |

### 4.3. Проверка текущих блокировок

```sql
-- активные запросы (не Sleep)
SHOW PROCESSLIST;

-- или через information_schema
SELECT id, time, state, LEFT(info, 100) as query
FROM information_schema.processlist
WHERE command != 'Sleep' AND time > 1
ORDER BY time DESC LIMIT 20;
```

### 4.4. Решение

- Разделить READ и WRITE на разные MySQL-хосты (read replica для SELECT, master для UPDATE)
- `SELECT` только нужные поля вместо `SELECT *`
- Ограничить пул соединений: `SetMaxOpenConns(50-100)` вместо 0

---

## 5. Диагностика: Goroutine Saturation (Go)

### 5.1. Сравнение internal duration vs upstream_response_time

Если приложение логирует внутренний duration обработки:

| internal duration | upstream_response_time | Диагноз |
|-------------------|----------------------|---------|
| 100ms | 100ms | Норма |
| 100ms | 1000ms | Запрос ждал 900ms в очереди перед обработкой |
| 800ms | 800ms | Сама обработка медленная (DB, внешние вызовы) |

### 5.2. Мониторинг горутин

Datadog Runtime Metrics:
```
runtime.go.num_goroutine{service:<name>}
```

Если число горутин растёт и не падает — утечка горутин (обычно из-за blocking call без таймаута).

### 5.3. Решение

- Добавить таймауты на все внешние вызовы (`context.WithTimeout`)
- Вынести не-критичные вызовы в горутины (`go func()`)
- Ограничить пулы соединений (DB, Redis)

---

## 6. Диагностика: Внешние зависимости

### 6.1. APM трейсы (Datadog)

```
Datadog → APM → Traces
Фильтр: service:<name>, @duration:>500ms
```

Waterfall view трейса показывает каждый span (MySQL query, Redis call, Kafka publish, HTTP call) и сколько каждый занял. Самый длинный span = узкое место.

### 6.2. На что обращать внимание

| Проблема в трейсе | Причина |
|-------------------|--------|
| Один SQL-запрос занимает > 100ms | Отсутствие индекса или lock contention |
| Kafka/RabbitMQ publish > 100ms | Синхронная запись, брокер перегружен |
| HTTP-вызов к другому сервису > 500ms | Тот сервис тоже тормозит |
| Большой gap между spans | CFS throttling или GC pause |

---

## 7. Чек-лист диагностики

При обнаружении `upstream_connect_time = 0` + `upstream_header_time >= 1s`:

**Шаг 1. CFS Throttling**
```bash
kubectl exec -n <ns> <pod> -- grep nr_throttled /sys/fs/cgroup/cpu.stat
```
Если `nr_throttled > 0` — убрать/увеличить CPU limits.

**Шаг 2. Сравнить internal duration vs nginx upstream_response_time**
Если разница большая — запросы стоят в очереди (goroutine saturation).

**Шаг 3. APM трейс**
Найти самый медленный span в waterfall.

**Шаг 4. MySQL метрики**
```sql
SHOW GLOBAL STATUS LIKE 'Innodb_row_lock_waits';
SHOW GLOBAL STATUS LIKE 'Threads_connected';
```

**Шаг 5. Проверить добавление подов**
Если добавление подов сразу помогает — подтверждение что проблема в capacity/throttling, не в инфраструктуре.

---

## 8. Реальный кейс: maxline-slotegrator (03.2026)

### Симптомы
- `upstream_response_time >= 1s` при `upstream_connect_time = 0.000`
- Internal duration сервиса: 100-150ms
- Добавление 2 подов (с 6 до 8) через HPA сразу устраняло проблему
- В Datadog визуально похоже на TCP retransmit, но `connect_time = 0`

### Диагностика
1. CFS `cpu.stat` показал: pod qcxfn throttled 3,901 раз за 4 дня (17 мин суммарной заморозки)
2. APM трейс (p98, 773ms): `SELECT * FROM user WHERE id = ?` занимал 615ms из-за InnoDB row lock
3. MySQL: `Innodb_row_lock_waits = 317,279`, `Threads_connected = 559`

### Корневая причина
CFS throttling (`cpu limits = requests = 6`, Guaranteed QoS) замораживал контейнер при всплесках CPU. Это запускало каскад: горутины копились → MySQL-соединения заняты дольше → row lock wait рос → ещё больше горутин.

### Решение
1. Убран CPU limit в deployment (оставлен только requests.cpu: 6)
2. В очереди: ограничение MySQL пула соединений, таймаут на RabbitMQ PublishBlocking, разделение read/write MySQL
