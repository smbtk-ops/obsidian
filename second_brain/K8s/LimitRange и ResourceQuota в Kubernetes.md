#### **1. LimitRange — ограничения на уровне контейнера и пода**

`LimitRange` задаёт минимальные, максимальные и дефолтные значения ресурсов (CPU, память, хранилище) для контейнеров, подов и PVC в пределах **namespace**.
##### **Основное назначение**

- Применяется автоматически ко всем создаваемым подам.
- Предотвращает чрезмерное потребление ресурсов одним контейнером.
- Устанавливает значения по умолчанию, если разработчик их не указал.

##### **Типичный пример:**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: example-limits
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: 100m
      memory: 50Mi
    default:
      cpu: 200m
      memory: 100Mi
    min:
      cpu: 50m
      memory: 50Mi
    max:
      cpu: 1
      memory: 1Gi
  - type: PersistentVolumeClaim
    min:
      storage: 1Gi
    max:
      storage: 10Gi
```

##### **Принципы:**

- Если в манифесте пода не указаны `resources.requests` или `resources.limits` — применяются значения `defaultRequest` и `default`.
- `min` и `max` задают рамки возможных значений.
- `maxLimitRequestRatio` ограничивает отношение лимита к запросу (например, лимит не может превышать запрос более чем в 10 раз).

---

#### **2. ResourceQuota — ограничения на уровне namespace**
`ResourceQuota` регулирует общий объём ресурсов, доступных **всему неймспейсу**.
##### **Основное назначение**
- Контролирует, сколько CPU, памяти, PVC, подов и других объектов может быть создано в namespace.
- Не задаёт лимиты подам напрямую, но ограничивает общий “бюджет”.

##### **Типичный пример:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
spec:
  hard:
    requests.cpu: 2
    requests.memory: 2Gi
    limits.cpu: 4
    limits.memory: 4Gi
    persistentvolumeclaims: 5
    requests.storage: 500Gi
    pods: 10
    secrets: 20
```

##### **Возможности:**

- Можно задавать квоты по **StorageClass**:
```yaml
   standard.storageclass.storage.k8s.io/requests.storage: 300Gi
   ssd.storageclass.storage.k8s.io/requests.storage: 0
   ```
→ команда не сможет использовать SSD-диски.
- Можно ограничить количество объектов:
  ```yaml
  services.loadbalancers: 1
  services.nodeports: 2
  ```
- Можно ограничивать по QoS-классам:
  ```yaml
   scopes:
   - BestEffort
   - NotTerminating
   ```

---
#### **3. Практическое применение**

1. **Создайте namespace:**
  ```bash
   kubectl create ns team-a
   ```
2. **Примените LimitRange:**
  ```bash
   kubectl apply -f limitrange.yaml -n team-a
   ```
3. **Примените ResourceQuota:**
  ```bash
   kubectl apply -f quota.yaml -n team-a
   ```
4. **Проверьте квоты:**
  ```bash
   kubectl describe quota -n team-a
   ```

---
#### **4. Рекомендации**
- Всегда задавайте `requests` и `limits` явно — это улучшает QoS и прогнозируемость нагрузки.
- Следите за поведением init-контейнеров — их ресурсы учитываются в квоте.
- При проблемах с созданием подов — смотрите `kubectl describe rs` или `kubectl describe quota`.

---
#### **Кратко**

| Сущность          | Уровень применения | Контролирует            | Тип ограничения                                            |
| ----------------- | ------------------ | ----------------------- | ---------------------------------------------------------- |
| **LimitRange**    | Namespace          | Контейнеры / поды / PVC | Минимум, максимум, значения по умолчанию                   |
| **ResourceQuota** | Namespace          | Все ресурсы и объекты   | Общий лимит по CPU, памяти, хранилищу, количеству объектов |

---

## Пример:

## 1. Манифест ResourceQuota — `ns-limit.yaml`

Этот объект ограничивает **общий бюджет ресурсов в namespace `default`**:  
не более 4 CPU, 4 Gi памяти и 20 подов.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ns-limit
  namespace: default
spec:
  hard:
    requests.cpu: 4
    requests.memory: 4Gi
    limits.cpu: 4
    limits.memory: 4Gi
    pods: 20
```

**Пояснение:**

- `requests.*` — гарантированный минимальный объём, который можно запрашивать.
- `limits.*` — верхний предел для суммарного потребления в namespace.
- `pods: 20` — суммарное число одновременно существующих подов в `default`.

Применение:
```bash
kubectl apply -f ns-limit.yaml
```

Проверка:
```bash
kubectl describe quota ns-limit -n default
```

---

## 2. Манифест LimitRange — `pod-limit.yaml`

Этот объект ограничивает **ресурсы на уровне каждого пода / контейнера**,  
и задаёт минимальные `requests` и максимальные `limits`.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pod-limit
  namespace: default
spec:
  limits:
  - type: Container
    min:
      cpu: 200m
      memory: 128Mi
    max:
      cpu: 1
      memory: 1Gi
    defaultRequest:
      cpu: 200m
      memory: 128Mi
    default:
      cpu: 1
      memory: 1Gi
```

**Пояснение:**

- `type: Container` — ограничение применяется к каждому контейнеру.
- `min` — нельзя запрашивать меньше 0.2 CPU и 128 Mi.
- `max` — нельзя выделить больше 1 CPU и 1 Gi памяти.
- `defaultRequest` и `default` — значения по умолчанию, если разработчик их не указал.

Применение:
```bash
kubectl apply -f pod-limit.yaml
```

Проверка:
```bash
kubectl describe limitrange pod-limit -n default
```

---

## 3. Проверка результата

После применения обоих объектов:
```bash
kubectl get resourcequota,limitrange -n default
```

Ожидаемый вывод:
```
NAME               AGE
resourcequota/ns-limit   ...
limitrange/pod-limit     ...
```

Создайте тестовый под без указанных ресурсов:
```bash
kubectl run test-pod --image=nginx
```

Затем проверьте, что Kubernetes подставил лимиты автоматически:
```bash
kubectl get pod test-pod -o jsonpath='{.spec.containers[0].resources}'
```

Результат должен содержать:
```
"requests":{"cpu":"200m","memory":"128Mi"},
"limits":{"cpu":"1","memory":"1Gi"}
```

