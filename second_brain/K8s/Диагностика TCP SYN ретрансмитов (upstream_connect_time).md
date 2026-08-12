
Руководство по поиску и устранению проблем с TCP SYN retransmit в связке nginx → MetalLB → K8s backend. Типичный симптом — `upstream_connect_time >= 1s` в логах nginx при статусе 200.

---

## 1. Суть проблемы

### 1.1. Что такое SYN retransmit

При установлении TCP-соединения клиент отправляет SYN-пакет и ожидает SYN-ACK от сервера. Если ответ не приходит, ядро Linux ждёт **1 секунду** и повторяет SYN — это и есть SYN retransmit.

Количество повторов определяется параметром `net.ipv4.tcp_syn_retries` (дефолт 6). Таймауты между попытками удваиваются:

| Попытка | Таймаут | Суммарное ожидание |
|---------|---------|-------------------|
| 1 | 1 сек | 1 сек |
| 2 | 2 сек | 3 сек |
| 3 | 4 сек | 7 сек |
| 4 | 8 сек | 15 сек |
| 5 | 16 сек | 31 сек |
| 6 | 32 сек | 63 сек |

Если в логах nginx `upstream_connect_time` стабильно чуть больше 1 секунды (1.016, 1.020) — это практически всегда один SYN retransmit. Если ~3 секунды — два ретрансмита, и так далее.

### 1.2. Почему SYN теряется

Основные причины потери SYN или SYN-ACK:

| Причина | Где происходит | Метрика для проверки |
|---------|---------------|---------------------|
| Переполнение listen backlog | Бэкенд-сервер | `ListenOverflows`, `ListenDrops` |
| Conntrack table overflow | Любая нода с NAT | `dmesg \| grep conntrack` |
| Сетевые потери | Между нодами | `TCPSynRetrans` на обеих сторонах |
| Нет keepalive — слишком много новых соединений | nginx | Количество TIME_WAIT |
| MetalLB L2 single-node bottleneck | Elected speaker node | `TCPSynRetrans` на одной ноде >> остальных |

---

## 2. Обнаружение проблемы

### 2.1. Nginx access log

В nginx логах с форматом `apm` или кастомным, включающим upstream-метрики:

```nginx
log_format apm '$remote_addr - $remote_user [$time_local] '
               '"$request" $status $body_bytes_sent '
               'upstream_connect_time=$upstream_connect_time '
               'upstream_header_time=$upstream_header_time '
               'upstream_response_time=$upstream_response_time '
               'request_time=$request_time';
```

Фильтрация записей с долгим connect time:

```bash
# из файла лога
awk '$0 ~ /upstream_connect_time=/ {
    match($0, /upstream_connect_time=([0-9.]+)/, a);
    if (a[1]+0 >= 1.0) print
}' /log/nginx/access.log

# количество ретрансмитов за последний час
awk -v threshold=1.0 '$0 ~ /upstream_connect_time=/ {
    match($0, /upstream_connect_time=([0-9.]+)/, a);
    if (a[1]+0 >= threshold) count++
} END {print count+0}' /log/nginx/access.log
```

В Datadog — фильтр `@upstream_connect_time:>=1` для соответствующего сервиса.

### 2.2. Метрики ядра (/proc/net/netstat)

Ключевые счётчики в `/proc/net/netstat` (TcpExt):

| Метрика | Что показывает |
|---------|---------------|
| `TCPSynRetrans` | Количество повторно отправленных SYN-пакетов |
| `TCPTimeouts` | Общее количество TCP таймаутов |
| `ListenOverflows` | Переполнения listen backlog (серверная сторона) |
| `ListenDrops` | Дропы на listen-сокетах |
| `TCPBacklogDrop` | Дропы из-за переполнения backlog приложения |
| `TCPTimeWaitOverflow` | Переполнение таблицы TIME_WAIT |

---

## 3. Диагностические команды

### 3.1. nstat — основной инструмент

`nstat` — утилита из пакета `iproute2`, читает счётчики из `/proc/net/netstat` и `/proc/net/snmp`.

```bash
# все метрики ретрансмитов (накопленные с момента загрузки)
nstat -az | grep -iE 'SynRetrans|Timeout|Overflow|Drop|Listen'

# только SYN retransmit
nstat -az TcpExtTCPSynRetrans

# наблюдение в реальном времени (delta каждые N секунд)
# второй столбец — прирост за интервал
nstat TcpExtTCPSynRetrans 5

# мониторинг через watch
watch -n5 'nstat -az TcpExtTCPSynRetrans'
```

Пример вывода `nstat -az`:

```
TcpExtTCPSynRetrans     66239      0.0
```

- Первый столбец — имя метрики
- Второй — абсолютное значение (с момента загрузки или последнего сброса)
- Третий — скорость (events/sec), `0.0` если используется `-a` (absolute)

### 3.2. Conntrack

```bash
# текущее количество записей vs лимит
sysctl net.netfilter.nf_conntrack_max
cat /proc/sys/net/netfilter/nf_conntrack_count

# проверка overflow в логах ядра
dmesg | grep -i "conntrack"
dmesg | grep -i "table full"

# детальная статистика conntrack
conntrack -S
```

### 3.3. Listen queue (backlog)

```bash
# ss показывает Recv-Q (текущая очередь) и Send-Q (максимальный размер)
# если Recv-Q приближается к Send-Q — backlog переполняется
ss -ltn | grep :8090

# пример вывода:
# State  Recv-Q Send-Q Local Address:Port
# LISTEN 0      32768  *:8090
#        ^      ^^^^^
#        |      максимальный backlog (= min(somaxconn, app_backlog))
#        текущая очередь (0 = ок, если растёт — проблема)
```

### 3.4. Количество соединений

```bash
# ESTABLISHED соединения к upstream
ss -tn state established | grep "10.10.1.177:8090" | wc -l

# TIME_WAIT соединения (индикатор отсутствия keepalive)
ss -tn state time-wait | grep "10.10.1.177:8090" | wc -l

# если TIME_WAIT >> ESTABLISHED — keepalive не работает
```

### 3.5. SYN flood warnings

```bash
# ядро логирует, если обнаруживает SYN flooding
dmesg | grep -i "SYN flood"
# пример: "TCP: request_sock_TCP: Possible SYN flooding on port 8090. Sending cookies."
```

### 3.6. Sysctl параметры

```bash
# socket/TCP
sysctl net.core.somaxconn              # макс. backlog (дефолт 128, рекомендуется 32768)
sysctl net.core.netdev_max_backlog     # очередь сетевого интерфейса
sysctl net.ipv4.tcp_max_syn_backlog    # макс. SYN backlog
sysctl net.ipv4.tcp_syn_retries        # кол-во попыток SYN (дефолт 6)
sysctl net.ipv4.tcp_tw_reuse           # повторное использование TIME_WAIT (1 = вкл)
sysctl net.ipv4.tcp_fin_timeout        # таймаут FIN-WAIT-2

# conntrack
sysctl net.netfilter.nf_conntrack_max
sysctl net.netfilter.nf_conntrack_buckets
```

---

## 4. Анализ результатов

### 4.1. Чек-лист диагностики

Порядок проверки — от самого вероятного к менее вероятному:

**Шаг 1. Проверить nginx keepalive**

```bash
# много TIME_WAIT = нет keepalive, каждый запрос создаёт новое соединение
ss -tn state time-wait | grep "<upstream_ip>:<port>" | wc -l
```

Если TIME_WAIT > 100 — keepalive скорее всего не настроен.

**Шаг 2. Проверить ListenOverflows на бэкенде**

```bash
nstat -az | grep -E 'ListenOverflows|ListenDrops'
```

Ненулевые значения = backlog переполняется → увеличить `somaxconn` и backlog приложения.

**Шаг 3. Проверить conntrack**

```bash
# утилизация conntrack таблицы
echo "$(cat /proc/sys/net/netfilter/nf_conntrack_count) / $(sysctl -n net.netfilter.nf_conntrack_max)" | bc -l

# если > 0.8 (80%) — нужно увеличивать
```

**Шаг 4. Проверить TCPSynRetrans по нодам**

```bash
# запустить на всех нодах кластера
for node in worker1 worker2 worker3; do
    echo "=== $node ==="
    ssh root@$node "nstat -az TcpExtTCPSynRetrans"
done
```

Если одна нода имеет значение в десятки раз больше — это MetalLB L2 bottleneck (весь трафик через одну ноду).

### 4.2. Типичные паттерны

**Паттерн 1: Нет keepalive**
- Симптомы: много TIME_WAIT, `upstream_connect_time ~1s` периодически
- Причина: каждый HTTP-запрос = новый TCP handshake
- Решение: добавить upstream блок с `keepalive` в nginx

**Паттерн 2: Backlog overflow**
- Симптомы: `ListenOverflows > 0`, `dmesg` показывает SYN flooding
- Причина: `somaxconn` слишком мал или приложение не успевает `accept()`
- Решение: увеличить `somaxconn`, оптимизировать приложение

**Паттерн 3: Conntrack overflow**
- Симптомы: `dmesg | grep conntrack` показывает "table full, dropping packet"
- Причина: `nf_conntrack_max` слишком мал для объёма NAT-трафика
- Решение: увеличить `nf_conntrack_max` и `nf_conntrack_buckets`

**Паттерн 4: MetalLB L2 bottleneck**
- Симптомы: `TCPSynRetrans` на одной ноде в десятки раз больше, чем на остальных
- Причина: MetalLB L2 направляет 100% трафика VIP через одну ноду
- Решение: nginx keepalive, или NodePort upstream, или BGP mode

---

## 5. Решения

### 5.1. Nginx keepalive (наибольший эффект)

Без keepalive каждый HTTP-запрос создаёт новое TCP-соединение:

```
запрос 1: SYN → SYN-ACK → ACK → HTTP → FIN
запрос 2: SYN → SYN-ACK → ACK → HTTP → FIN  # новое соединение
запрос 3: SYN → SYN-ACK → ACK → HTTP → FIN  # ещё одно
```

С keepalive одно соединение обслуживает сотни запросов:

```
соединение: SYN → SYN-ACK → ACK
  запрос 1: HTTP request → HTTP response
  запрос 2: HTTP request → HTTP response  # то же соединение
  ...
  запрос 1000: HTTP request → HTTP response
FIN
```

Конфигурация:

```nginx
upstream backend {
    server 10.10.1.177:8090;

    keepalive 64;              # до 64 idle-соединений в пуле
    keepalive_requests 1000;   # до 1000 запросов на одно соединение
    keepalive_timeout 60s;     # закрывать idle-соединение через 60 сек
}

server {
    location / {
        proxy_pass http://backend;

        # обязательно для работы keepalive:
        proxy_http_version 1.1;          # keepalive требует HTTP/1.1
        proxy_set_header Connection "";  # очистить Connection: close от клиента

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Параметры keepalive:**

| Директива | Значение | Описание |
|-----------|----------|----------|
| `keepalive N` | 32-128 | Количество idle-соединений в пуле на каждый worker process. Подбирается под нагрузку. Слишком мало — соединения будут закрываться и открываться заново. Слишком много — будут висеть без дела |
| `keepalive_requests N` | 1000 | Максимум запросов через одно соединение. После достижения лимита соединение закрывается и создаётся новое. Предотвращает утечки памяти в backend |
| `keepalive_timeout Ns` | 60s | Время жизни idle-соединения. Если за это время запросов не было — соединение закрывается |

`proxy_http_version 1.1` и `proxy_set_header Connection ""` — обязательны. Без них keepalive не работает, потому что HTTP/1.0 по умолчанию закрывает соединение после каждого ответа, а `Connection: close` от клиента принудительно закроет upstream-соединение.

### 5.2. Sysctl тюнинг

Рекомендуемые значения для K8s нод и серверов с nginx:

```bash
# /etc/sysctl.d/99-tcp-tune.conf

# Максимальный backlog listen-сокетов
# Дефолт 128 — критически мало для нагруженных сервисов
net.core.somaxconn = 32768

# Очередь входящих пакетов на сетевом интерфейсе
net.core.netdev_max_backlog = 65535

# Максимальный SYN backlog (полуоткрытые соединения)
net.ipv4.tcp_max_syn_backlog = 16384

# Разрешить повторное использование TIME_WAIT сокетов
net.ipv4.tcp_tw_reuse = 1

# Время ожидания FIN-WAIT-2 (дефолт 60, можно уменьшить)
net.ipv4.tcp_fin_timeout = 5

# Не замедлять TCP после idle
net.ipv4.tcp_slow_start_after_idle = 0

# Conntrack — для нод с kube-proxy/NAT
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_buckets = 262144
```

Применение:

```bash
sysctl --system
```

### 5.3. MetalLB L2 bottleneck — варианты обхода

MetalLB в L2 mode направляет весь трафик на одну ноду. Варианты решения:

**Вариант A: NodePort upstream в nginx (без MetalLB)**

Вместо обращения к VIP `10.10.1.177:8090`, nginx обращается напрямую к NodePort на всех воркерах:

```nginx
upstream backend {
    server 10.10.1.181:30090;  # worker1
    server 10.10.1.182:30090;  # worker2
    server 10.10.1.183:30090;  # worker3

    keepalive 64;
    keepalive_requests 1000;
    keepalive_timeout 60s;
}
```

Трафик распределяется равномерно по всем нодам.

**Вариант B: MetalLB BGP mode**

Если маршрутизатор поддерживает BGP, MetalLB может анонсировать VIP со всех нод. Маршрутизатор использует ECMP для распределения трафика:

```yaml
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: router
  namespace: metallb-system
spec:
  myASN: 64500
  peerASN: 64501
  peerAddress: 10.10.1.1

---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: bgp-advert
  namespace: metallb-system
spec:
  ipAddressPools:
    - primary
```

---

## 6. Мониторинг после применения fix

### 6.1. Замер до/после

```bash
# до применения keepalive — зафиксировать значения
ssh root@<node> "nstat -az TcpExtTCPSynRetrans"
# записать число, например: 66239

# применить keepalive, nginx -t && nginx -s reload

# через сутки
ssh root@<node> "nstat -az TcpExtTCPSynRetrans"
# записать число, например: 66450
# разница = 211 за сутки (было тысячи в день → стало сотни)
```

### 6.2. Наблюдение в реальном времени

```bash
# delta каждые 5 секунд (второй столбец = прирост)
nstat TcpExtTCPSynRetrans 5

# все ключевые метрики
watch -n5 'nstat -az | grep -iE "SynRetrans|ListenOverflow|ListenDrop|TCPTimeout"'
```

### 6.3. Проверка что keepalive работает

```bash
# на nginx-сервере:
# ESTABLISHED должен вырасти (постоянные соединения)
ss -tn state established | grep "<upstream>:<port>" | wc -l

# TIME_WAIT должен снизиться (меньше закрытых соединений)
ss -tn state time-wait | grep "<upstream>:<port>" | wc -l
```

До keepalive: TIME_WAIT >> ESTABLISHED
После keepalive: ESTABLISHED ~ keepalive count, TIME_WAIT ~ 0

### 6.4. В Datadog / логах nginx

После применения fix отслеживать:
- Фильтр `@upstream_connect_time:>=1` — количество должно резко снизиться
- Среднее `upstream_connect_time` — должно стать < 10ms

---

## 7. Диагностический скрипт

Готовый скрипт для запуска на любой ноде — проверяет все описанные метрики:

```bash
#!/bin/bash
# TCP SYN Retransmit Diagnostics
set -e

echo "=========================================="
echo "  Host: $(hostname) | Date: $(date)"
echo "=========================================="

echo ""
echo "--- Sysctl ---"
for p in net.core.somaxconn net.core.netdev_max_backlog \
         net.ipv4.tcp_max_syn_backlog net.ipv4.tcp_syn_retries \
         net.ipv4.tcp_tw_reuse net.ipv4.tcp_fin_timeout; do
    printf "%-35s %s\n" "$p:" "$(sysctl -n $p 2>/dev/null || echo 'N/A')"
done

echo ""
echo "--- Conntrack ---"
max=$(sysctl -n net.netfilter.nf_conntrack_max 2>/dev/null || echo 0)
cur=$(cat /proc/sys/net/netfilter/nf_conntrack_count 2>/dev/null || echo 0)
pct=0
[ "$max" -gt 0 ] 2>/dev/null && pct=$((cur * 100 / max))
echo "count/max: ${cur}/${max} (${pct}%)"

echo ""
echo "--- TCP Retransmit Counters ---"
nstat -az 2>/dev/null | grep -E 'TCPSynRetrans|TCPTimeouts|ListenOverflows|ListenDrops|TCPBacklogDrop' || \
    echo "nstat not available"

echo ""
echo "--- dmesg warnings ---"
dmesg 2>/dev/null | grep -iE "conntrack|syn flood|table full" | tail -5 || \
    echo "clean"

echo ""
echo "--- Listen queues (Recv-Q / Send-Q) ---"
ss -ltn 2>/dev/null | head -1
ss -ltn 2>/dev/null | awk 'NR>1 && $2>0' || echo "all queues empty"
```

Запуск на всех нодах кластера:

```bash
B64=$(base64 < diag-retransmits.sh)
for node in 10.10.1.181 10.10.1.182 10.10.1.183; do
    echo "====== $node ======"
    ssh root@$node "echo '$B64' | base64 -d | bash"
done
```

---

## 8. Важные замечания

- Счётчики `TCPSynRetrans`, `ListenOverflows` и др. — **накопительные** с момента загрузки ноды. Для оценки скорости нужно брать два замера и вычислять разницу, либо использовать `nstat` без флага `-a` (показывает delta с последнего вызова)
- `somaxconn = 128` (дефолт Linux) — критически мало для любого нагруженного сервиса. Если на сервере не применён sysctl-тюнинг, это первое что нужно проверить
- `ListenOverflows = 0` не означает отсутствие проблемы. SYN может теряться на уровне сети, conntrack или MetalLB до того, как дойдёт до listen-сокета
- В MetalLB L2 mode **один** speaker pod анонсирует VIP. Проверяется через `nstat` — нода с аномально высоким `TCPSynRetrans` = elected speaker
- Nginx `keepalive` в upstream — самая эффективная мера. Снижает количество TCP handshakes на порядки и соответственно вероятность SYN retransmit
