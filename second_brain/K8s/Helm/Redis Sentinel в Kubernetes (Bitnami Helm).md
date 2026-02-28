
## 1. Обзор

**Bitnami Redis Helm chart** разворачивает Redis с поддержкой Sentinel для автоматического failover. Устанавливается через OCI registry (традиционный `helm repo add bitnami` больше не работает).

Архитектура: StatefulSet из N нод, каждая содержит 3 контейнера:
- **redis** — основной процесс (master или replica)
- **sentinel** — процесс Sentinel для мониторинга и failover
- **metrics** — redis-exporter для Prometheus метрик

---

## 2. Подготовка

### 2.1. Создать Secret с паролем

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: redis-myapp-secret
  namespace: myapp
type: Opaque
data:
  redis-password: <BASE64_PASSWORD>
```

```bash
echo -n 'your-password' | base64
kubectl apply -f redis-secret.yaml
```

---

## 3. Файл values.yaml

Минимальный пример для Redis Sentinel без persistence:

```yaml
architecture: replication

auth:
  enabled: true
  existingSecret: "redis-myapp-secret"
  existingSecretPasswordKey: "redis-password"

sentinel:
  enabled: true
  quorum: 2
  downAfterMilliseconds: 5000
  failoverTimeout: 60000
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
  configuration: |-
    sentinel resolve-hostnames yes
    sentinel announce-hostnames yes

master:
  persistence:
    enabled: false
    # С Longhorn — см. секцию 3.1
  resources:
    requests:
      cpu: 500m
      memory: 2Gi
    limits:
      cpu: 2
      memory: 6Gi
  configuration: |-
    maxmemory 5gb
    maxmemory-policy allkeys-lru
    save ""
    appendonly no
    tcp-keepalive 300

replica:
  replicaCount: 3
  persistence:
    enabled: false
    # С Longhorn — см. секцию 3.1
  resources:
    requests:
      cpu: 500m
      memory: 2Gi
    limits:
      cpu: 2
      memory: 6Gi
  configuration: |-
    maxmemory 5gb
    maxmemory-policy allkeys-lru
    save ""
    appendonly no
    tcp-keepalive 300

metrics:
  enabled: true
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 150m
      memory: 192Mi
  serviceMonitor:
    enabled: false
```

---

## 3.1. Persistence с Longhorn

Для сохранения данных между перезапусками подов используется StorageClass `longhorn`. Каждая нода Redis получит свой PVC.

### Изменения в values.yaml

Заменить блоки `persistence` для master и replica:

```yaml
master:
  persistence:
    enabled: true
    storageClass: "longhorn"
    size: 10Gi
    accessModes:
      - ReadWriteOnce

replica:
  persistence:
    enabled: true
    storageClass: "longhorn"
    size: 10Gi
    accessModes:
      - ReadWriteOnce
```

### Полный пример values.yaml с persistence

```yaml
architecture: replication

auth:
  enabled: true
  existingSecret: "redis-myapp-secret"
  existingSecretPasswordKey: "redis-password"

sentinel:
  enabled: true
  quorum: 2
  downAfterMilliseconds: 5000
  failoverTimeout: 60000
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
  configuration: |-
    sentinel resolve-hostnames yes
    sentinel announce-hostnames yes

master:
  persistence:
    enabled: true
    storageClass: "longhorn"
    size: 10Gi
    accessModes:
      - ReadWriteOnce
  resources:
    requests:
      cpu: 500m
      memory: 2Gi
    limits:
      cpu: 2
      memory: 6Gi
  configuration: |-
    maxmemory 5gb
    maxmemory-policy allkeys-lru
    tcp-keepalive 300

replica:
  replicaCount: 3
  persistence:
    enabled: true
    storageClass: "longhorn"
    size: 10Gi
    accessModes:
      - ReadWriteOnce
  resources:
    requests:
      cpu: 500m
      memory: 2Gi
    limits:
      cpu: 2
      memory: 6Gi
  configuration: |-
    maxmemory 5gb
    maxmemory-policy allkeys-lru
    tcp-keepalive 300

metrics:
  enabled: true
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 150m
      memory: 192Mi
  serviceMonitor:
    enabled: false
```

При включённом persistence убраны `save ""` и `appendonly no` — Redis будет использовать RDB/AOF по умолчанию для записи на диск.

### Проверка PVC после установки

```bash
kubectl get pvc -n myapp -l app.kubernetes.io/name=redis
```

Ожидаемый результат:

```
NAME                          STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   AGE
redis-data-redis-myapp-node-0 Bound    pvc-xxx...   10Gi       RWO            longhorn       5m
redis-data-redis-myapp-node-1 Bound    pvc-xxx...   10Gi       RWO            longhorn       5m
redis-data-redis-myapp-node-2 Bound    pvc-xxx...   10Gi       RWO            longhorn       5m
```

### Расширение тома

Longhorn поддерживает online expansion — увеличение без даунтайма:

```bash
kubectl patch pvc redis-data-redis-myapp-node-0 -n myapp \
  -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

Повторить для каждой ноды (node-0, node-1, node-2).

### Важно

- `size` должен быть больше ожидаемого объёма данных в Redis (RDB дамп + AOF + full sync файлы)
- При `replicaCount: 3` создаётся 3 PVC — убедиться что в Longhorn достаточно свободного места
- Longhorn `defaultReplicaCount: 2` означает что каждый PVC хранится на 2 воркерах — фактически потребуется `size * replicaCount * 2` дискового пространства

---

## 4. Установка

```bash
helm install redis-myapp oci://registry-1.docker.io/bitnamicharts/redis \
  --namespace myapp \
  --values redis-values.yaml \
  --wait --timeout 5m
```

Проверка:

```bash
kubectl get pods -n myapp -l app.kubernetes.io/name=redis -w
```

Ожидаемый результат — все ноды `3/3 Running`.

---

## 5. Обновление

```bash
helm upgrade redis-myapp oci://registry-1.docker.io/bitnamicharts/redis \
  --namespace myapp \
  --values redis-values.yaml
```

Helm выполнит rolling update StatefulSet (node-2 → node-1 → node-0).

---

## 6. Подключение

### 6.1. Из приложения внутри кластера

```
redis+sentinel://:PASSWORD@redis-myapp:26379?sentinel_master_name=mymaster&sentinel_password=PASSWORD
```

Параметр `sentinel_password` обязателен — Bitnami использует единый пароль для data-нод и sentinel.

### 6.2. Проверка вручную

```bash
# Получить пароль
export REDIS_PASSWORD=$(kubectl get secret -n myapp redis-myapp-secret \
  -o jsonpath="{.data.redis-password}" | base64 -d)

# Проверить sentinel master
kubectl exec -n myapp redis-myapp-node-0 -c sentinel -- \
  redis-cli -p 26379 -a $REDIS_PASSWORD sentinel master mymaster

# Проверить данные
kubectl exec -n myapp redis-myapp-node-0 -c redis -- \
  redis-cli -a $REDIS_PASSWORD INFO memory
```

---

## 7. Troubleshooting

### 7.1. Pods Evicted — ephemeral storage

**Симптом:** Redis поды циклически перезапускаются, в events:

```
Warning  Evicted  Pod ephemeral local storage usage exceeds the total limit of containers 2Gi
```

**Причина:** при `persistence.enabled: false` данные пишутся в emptyDir (ephemeral storage). При репликации Redis записывает RDB дамп + AOF файлы на диск, даже если `appendonly no` в конфиге — Redis автоматически включает AOF при full sync с мастера.

Bitnami chart по умолчанию ставит `ephemeral-storage: 2Gi` лимит на metrics контейнер. Суммарный лимит пода = 2Gi, что легко превышается при объёме данных >1GB.

**Решение:** явно задать `metrics.resources` без `ephemeral-storage` и не задавать `ephemeral-storage` на остальных контейнерах:

```yaml
metrics:
  enabled: true
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 150m
      memory: 192Mi
```

Диагностика:

```bash
# Проверить events
kubectl get events -n myapp --field-selector reason=Evicted

# Проверить использование диска
kubectl exec -n myapp redis-myapp-node-0 -c redis -- du -sh /data/

# Проверить лимиты на поде
kubectl get pod redis-myapp-node-0 -n myapp -o yaml | grep ephemeral-storage
```

### 7.2. NOAUTH HELLO при подключении

Ошибка аутентификации на sentinel. Убедиться что в URL указан `sentinel_password`.

### 7.3. Sentinel не определяет master

```bash
kubectl exec -n myapp redis-myapp-node-0 -c sentinel -- \
  redis-cli -p 26379 -a $REDIS_PASSWORD sentinel master mymaster

kubectl logs -n myapp redis-myapp-node-0 -c sentinel --tail=50
```

---

## 8. Удаление

```bash
helm uninstall redis-myapp --namespace myapp
```

Секреты удаляются отдельно:

```bash
kubectl delete secret redis-myapp-secret -n myapp
```

---

## 9. Полезные команды

| Действие | Команда |
|----------|---------|
| Кол-во ключей | `kubectl exec -c redis redis-myapp-node-0 -- redis-cli -a $PASS DBSIZE` |
| Память | `kubectl exec -c redis redis-myapp-node-0 -- redis-cli -a $PASS INFO memory` |
| Конфиг | `kubectl exec -c redis redis-myapp-node-0 -- redis-cli -a $PASS CONFIG GET appendonly` |
| Sentinel status | `kubectl exec -c sentinel redis-myapp-node-0 -- redis-cli -p 26379 -a $PASS sentinel master mymaster` |
| Helm ревизии | `helm history redis-myapp -n myapp` |
| Откат | `helm rollback redis-myapp <revision> -n myapp` |
