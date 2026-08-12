## 1. Общая схема

```
StorageClass → (dynamic provisioning) → PV ← PVC ← Pod
```

- **PersistentVolume (PV)** — ресурс хранилища в кластере (диск, Longhorn volume, NFS share и т.д.)
- **PersistentVolumeClaim (PVC)** — запрос на хранилище от пода. Привязывается к подходящему PV
- **StorageClass** — шаблон для автоматического создания PV (dynamic provisioning)

Pod не работает с PV напрямую — только через PVC.

---

## 2. PersistentVolume (PV)

### 2.1. Статусы PV

| Статус | Значение |
|---|---|
| `Available` | Свободен, не привязан к PVC, готов к использованию |
| `Bound` | Привязан к конкретному PVC |
| `Released` | PVC удалён, но PV не очищен (только при Reclaim Policy = Retain) |
| `Failed` | Автоматическая очистка/удаление не удалась |

### 2.2. Reclaim Policy

Определяет что происходит с PV **после удаления PVC**:

| Policy | Поведение | Данные | Когда использовать |
|---|---|---|---|
| `Delete` | PV и underlying storage удаляются автоматически | Потеряны | Dev/test, временные данные |
| `Retain` | PV остаётся в `Released`, данные сохраняются | Сохранены | Prod базы данных, важные данные |
| `Recycle` | Deprecated. Делал `rm -rf` и возвращал PV в `Available` | Удалены | Не использовать |

Проверка:

```bash
kubectl get pv -o custom-columns='NAME:.metadata.name,RECLAIM:.spec.persistentVolumeReclaimPolicy,CLAIM:.spec.claimRef.namespace/.spec.claimRef.name'
```

Изменение на Retain:

```bash
kubectl patch pv <PV_NAME> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

### 2.3. Access Modes

| Режим | Сокращение | Описание |
|---|---|---|
| `ReadWriteOnce` | RWO | Чтение/запись с одной ноды |
| `ReadOnlyMany` | ROX | Только чтение с нескольких нод |
| `ReadWriteMany` | RWX | Чтение/запись с нескольких нод |
| `ReadWriteOncePod` | RWOP | Чтение/запись только одним подом (K8s 1.27+) |

Longhorn по умолчанию поддерживает RWO. Для RWX нужен Longhorn NFS.

### 2.4. Reclaim Modes

| Режим | Описание |
|---|---|
| `Filesystem` | Volume монтируется как директория (по умолчанию) |
| `Block` | Volume доступен как raw block device |

---

## 3. PersistentVolumeClaim (PVC)

### 3.1. Статусы PVC

| Статус | Значение |
|---|---|
| `Pending` | Ожидает подходящий PV (нет доступного или StorageClass создаёт) |
| `Bound` | Привязан к PV, готов к использованию |
| `Lost` | PV, к которому был привязан, удалён или недоступен |
| `Terminating` | Удаление запущено, но finalizer блокирует (под ещё использует) |

### 3.2. Создание PVC (dynamic provisioning)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi
```

StorageClass `longhorn` автоматически создаст PV и Longhorn volume.

### 3.3. Привязка PVC к существующему PV

Если PV уже есть (например после восстановления):

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi
  volumeName: pvc-f0aa84ce-3b3e-4009-a9cd-73577d792b33
```

`volumeName` — имя конкретного PV для привязки.

---

## 4. StorageClass

### 4.1. Просмотр

```bash
# Все storage classes
kubectl get sc

# Подробности
kubectl describe sc longhorn
```

### 4.2. Пример StorageClass Longhorn

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "30"
  dataLocality: "disabled"
```

Ключевые параметры:
- `reclaimPolicy` — Reclaim Policy по умолчанию для создаваемых PV
- `allowVolumeExpansion` — разрешить увеличение размера PVC
- `volumeBindingMode` — `Immediate` (создать PV сразу) или `WaitForFirstConsumer` (ждать пока под запросит)
- `numberOfReplicas` — количество реплик Longhorn

### 4.3. Изменение Reclaim Policy по умолчанию

StorageClass определяет Reclaim Policy для **новых** PV. Уже созданные PV не меняются.

```bash
# Проверить текущую policy в StorageClass
kubectl get sc longhorn -o jsonpath='{.reclaimPolicy}{"\n"}'
```

Чтобы новые PV создавались с `Retain`:

```bash
kubectl patch sc longhorn -p '{"reclaimPolicy":"Retain"}'
```

Или пересоздать StorageClass (reclaimPolicy нельзя менять patch в некоторых версиях):

```bash
kubectl get sc longhorn -o yaml > sc-longhorn.yaml
# Изменить reclaimPolicy: Retain
kubectl delete sc longhorn
kubectl apply -f sc-longhorn.yaml
```

---

## 5. Жизненный цикл

### 5.1. Создание и привязка

```
PVC создан (Pending) → StorageClass создаёт PV → PVC Bound ↔ PV Bound → Pod монтирует
```

### 5.2. Удаление PVC при Reclaim Policy = Delete

```
PVC удалён → PV удалён → Longhorn volume удалён → данные потеряны
```

### 5.3. Удаление PVC при Reclaim Policy = Retain

```
PVC удалён → PV переходит в Released → данные сохранены
```

Повторная привязка:

```bash
# 1. Очистить claimRef (PV перейдёт из Released в Available)
kubectl patch pv <PV_NAME> -p '{"spec":{"claimRef":null}}'

# 2. Создать новый PVC с volumeName = имя PV
kubectl apply -f new-pvc.yaml
```

### 5.4. Защита от удаления (finalizers)

PVC имеет finalizer `kubernetes.io/pvc-protection` — Kubernetes не удалит PVC пока его использует хотя бы один под. PVC перейдёт в `Terminating` и будет ждать.

PV имеет finalizer `kubernetes.io/pv-protection` — аналогично, PV не удалится пока к нему привязан PVC.

Принудительное удаление (снятие finalizer):

```bash
kubectl patch pvc <PVC_NAME> -n <NS> -p '{"metadata":{"finalizers":null}}'
kubectl patch pv <PV_NAME> -p '{"metadata":{"finalizers":null}}'
```

---

## 6. Расширение PVC

Если StorageClass имеет `allowVolumeExpansion: true`:

```bash
kubectl patch pvc <PVC_NAME> -n <NS> -p '{"spec":{"resources":{"requests":{"storage":"30Gi"}}}}'
```

Для Longhorn расширение происходит онлайн (без остановки пода). Уменьшить размер нельзя.

---

## 7. StatefulSet и PVC

### 7.1. volumeClaimTemplates

StatefulSet создаёт отдельный PVC для каждого пода автоматически:

```yaml
apiVersion: apps/v1
kind: StatefulSet
spec:
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: longhorn
        resources:
          requests:
            storage: 20Gi
```

Имя PVC: `{volumeClaimTemplate.name}-{statefulset.name}-{ordinal}` → `data-mydb-0`

### 7.2. PVC Retention Policy

```yaml
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain   # при удалении StatefulSet
    whenScaled: Retain     # при scale down
```

| Значение | Поведение |
|---|---|
| `Retain` | PVC сохраняется (по умолчанию) |
| `Delete` | PVC удаляется автоматически |

Проверка:

```bash
kubectl get statefulsets --all-namespaces -o json | \
  jq '.items[] | {ns: .metadata.namespace, name: .metadata.name, policy: .spec.persistentVolumeClaimRetentionPolicy}'
```

Для prod баз всегда `Retain` / `Retain`.

---

## 8. Полезные команды

```bash
# Все PV с деталями
kubectl get pv -o wide

# Все PVC во всех namespace
kubectl get pvc --all-namespaces

# Найти PV по имени PVC
kubectl get pvc <NAME> -n <NS> -o jsonpath='{.spec.volumeName}{"\n"}'

# Найти PVC по имени PV
kubectl get pv <PV_NAME> -o jsonpath='{.spec.claimRef.namespace}/{.spec.claimRef.name}{"\n"}'

# Размер используемого storage
kubectl get pvc --all-namespaces -o custom-columns='NS:.metadata.namespace,NAME:.metadata.name,SIZE:.spec.resources.requests.storage,SC:.spec.storageClassName,STATUS:.status.phase'

# Массовая смена Reclaim Policy на Retain
kubectl get pv -o name | xargs -I {} kubectl patch {} -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

---

## 9. Рекомендации для prod

1. **Reclaim Policy = Retain** для всех PV с данными
2. **numberOfReplicas >= 2** в Longhorn StorageClass
3. **StatefulSet PVC Retention = Retain/Retain**
4. **Longhorn Node Drain Policy = block-if-contains-last-replica**
5. Регулярно проверять `kubectl get pv` — нет ли `Released` PV (забытые данные)
6. Перед удалением PVC убедиться что данные зарезервированы
