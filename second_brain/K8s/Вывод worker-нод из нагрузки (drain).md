## 1. Обзор

Процедура безопасного вывода worker-нод из нагрузки для обновления ОС, Kubernetes, обслуживания и т.д.

## 2. Предварительные проверки

### 2.1. Проверить состояние нод

```bash
kubectl get nodes -o wide
```

Все ноды должны быть `Ready`.

### 2.2. Проверить Longhorn volumes и реплики

```bash
# Статус volumes
kubectl -n longhorn-system get volumes.longhorn.io

# Убедиться что все volumes healthy
kubectl -n longhorn-system get volumes.longhorn.io -o json | \
  jq '.items[] | {name: .metadata.name, state: .status.state, robustness: .status.robustness, replicas: .spec.numberOfReplicas}'

# Распределение реплик по нодам
kubectl -n longhorn-system get replicas.longhorn.io -o json | \
  jq '.items[] | {volume: .spec.volumeName, node: .spec.nodeID, state: .status.currentState}'
```

Убедиться что:
- Все volumes в состоянии `attached` / `healthy`
- Каждый volume имеет реплики минимум на 2 разных нодах
- Нода, которую собираемся дрейнить, НЕ содержит единственную реплику какого-либо volume

### 2.3. Проверить Longhorn Drain Policy

```bash
kubectl -n longhorn-system get settings.longhorn.io node-drain-policy -o jsonpath='{.value}{"\n"}'
```

Должно быть `block-if-contains-last-replica` — не даст дрейнить ноду если на ней последняя реплика volume.

### 2.4. Проверить Reclaim Policy PV

```bash
kubectl get pv -o custom-columns='NAME:.metadata.name,RECLAIM:.spec.persistentVolumeReclaimPolicy,CLAIM:.spec.claimRef.namespace/.spec.claimRef.name'
```

**Все PV должны иметь Reclaim Policy = `Retain`**. Если стоит `Delete` — при случайном удалении PVC данные будут потеряны.

Исправление:
```bash
kubectl patch pv <PV_NAME> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

### 2.5. Проверить PVC Retention Policy StatefulSets

```bash
kubectl get statefulsets --all-namespaces -o json | \
  jq '.items[] | {namespace: .metadata.namespace, name: .metadata.name, pvcRetentionPolicy: .spec.persistentVolumeClaimRetentionPolicy}'
```

Должно быть `whenDeleted: Retain, whenScaled: Retain`. Если стоит `Delete` — drain может привести к удалению PVC.

### 2.6. Проверить что на ноде запущено

```bash
kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=<NODE_NAME>
```

Обратить внимание на:
- StatefulSet поды (базы данных) — будут эвакуированы, нужны реплики Longhorn на других нодах
- Single-replica Deployments — кратковременный даунтайм при пересоздании

---

## 3. Процедура drain

### 3.1. Порядок вывода нод

Дрейнить **строго по одной** ноде. Дождаться полного восстановления перед переходом к следующей.

Порядок не критичен если каждый Longhorn volume имеет реплики на 2+ нодах, но рекомендуется начинать с ноды, на которой меньше всего StatefulSet подов.

### 3.2. Cordon (запрет планирования)

```bash
kubectl cordon <NODE_NAME>
```

Новые поды не будут планироваться на эту ноду, существующие продолжат работать.

### 3.3. Drain (эвакуация подов)

```bash
kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data
```

Флаги:
- `--ignore-daemonsets` — пропускает DaemonSet поды (datadog-agent, longhorn-manager, speaker и т.д.)
- `--delete-emptydir-data` — разрешает удалять emptyDir данные (временные кеши, nginx temp)

**НЕ используй `--force`** без необходимости — он удаляет поды без контроллера (standalone pods). Если drain блокируется — разберись в причине, а не обходи её.

### 3.4. Что происходит при drain

1. Под эвакуируется с ноды
2. Контроллер (Deployment/StatefulSet) создаёт новый под на другой доступной ноде
3. Longhorn volume отсоединяется от старой ноды и подключается к новой (данные доступны через сетевую реплику)
4. `emptyDir` данные теряются (это нормально — они эфемерные)
5. **PVC и PV НЕ удаляются** — данные остаются

### 3.5. Проверка после drain

```bash
# Нода должна быть Ready,SchedulingDisabled
kubectl get nodes

# Все поды должны быть Running на других нодах
kubectl get pods --all-namespaces -o wide | grep -v Completed

# Longhorn volumes должны быть healthy
kubectl -n longhorn-system get volumes.longhorn.io -o json | \
  jq '.items[] | {name: .metadata.name, robustness: .status.robustness}'

# PVC должны быть Bound (НЕ Terminating)
kubectl get pvc --all-namespaces
```

### 3.6. Выполнить обслуживание

Обновление ОС, Kubernetes, containerd и т.д.

### 3.7. Uncordon (возврат ноды)

```bash
kubectl uncordon <NODE_NAME>
```

Проверить что нода вернулась:
```bash
kubectl get nodes
# Статус должен быть Ready (без SchedulingDisabled)
```

### 3.8. Дождаться восстановления Longhorn реплик

```bash
# Подождать пока все volumes станут healthy
kubectl -n longhorn-system get volumes.longhorn.io -o json | \
  jq '.items[] | {name: .metadata.name, robustness: .status.robustness}'
```

Longhorn автоматически пересоздаст реплики на вернувшейся ноде. Дождаться `healthy` перед drain следующей ноды.

---

## 4. Возможные проблемы

### 4.1. Drain блокируется — PodDisruptionBudget

```
error: cannot evict pod: would violate PodDisruptionBudget
```

Проверить PDB:
```bash
kubectl get pdb --all-namespaces
```

Решение: убедиться что у сервиса достаточно реплик на других нодах.

### 4.2. Drain блокируется — local storage

```
cannot delete Pods with local storage (use --delete-emptydir-data to override)
```

Добавить `--delete-emptydir-data`. Это затрагивает только emptyDir (временные данные), PVC не трогает.

### 4.3. PVC в статусе Terminating после drain

Причины:
- StatefulSet имеет `persistentVolumeClaimRetentionPolicy.whenScaled: Delete`
- Кто-то вручную удалил PVC

Восстановление:
```bash
# 1. Защитить данные — сменить reclaim policy на Retain
kubectl patch pv <PV_NAME> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

# 2. Снять finalizer чтобы PVC завершил удаление
kubectl -n <NAMESPACE> patch pvc <PVC_NAME> -p '{"metadata":{"finalizers":null}}'

# 3. Очистить claimRef на PV
kubectl patch pv <PV_NAME> -p '{"spec":{"claimRef":null}}'

# 4. Пересоздать PVC с привязкой к тому же PV
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <PVC_NAME>
  namespace: <NAMESPACE>
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: <SIZE>
  storageClassName: longhorn
  volumeName: <PV_NAME>
EOF
```

### 4.4. Longhorn volume в состоянии degraded

Volume потерял реплику. Если осталась хотя бы одна running реплика — данные в безопасности. Longhorn автоматически пересоздаст недостающую реплику после uncordon.

Если volume `faulted` (нет running реплик) — нужно вернуть ноду с последней репликой в строй (uncordon).

### 4.5. Pod не может запуститься на новой ноде — nodeAffinity/nodeSelector

Проверить:
```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

Если pod привязан к конкретной ноде через nodeSelector — он не сможет переехать. Нужно либо убрать привязку, либо дрейнить другие ноды первыми.

---

## 5. Краткая шпаргалка (копировать и выполнять)

### 5.1. Проверки перед drain

```bash
# 1. Ноды Ready?
kubectl get nodes -o wide

# 2. Longhorn volumes healthy + распределение реплик
kubectl -n longhorn-system get volumes.longhorn.io -o json | \
  jq '.items[] | {name: .metadata.name, robustness: .status.robustness, replicas: .spec.numberOfReplicas}'

kubectl -n longhorn-system get replicas.longhorn.io -o json | \
  jq '.items[] | {volume: .spec.volumeName, node: .spec.nodeID, state: .status.currentState}'

# 3. Reclaim Policy = Retain?
kubectl get pv -o custom-columns='NAME:.metadata.name,RECLAIM:.spec.persistentVolumeReclaimPolicy'

# Если Delete — исправить:
# kubectl patch pv <PV_NAME> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

### 5.2. Drain

```bash
kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data
```

### 5.3. Проверки после drain

```bash
# Нода SchedulingDisabled
kubectl get nodes

# Нет застрявших подов
kubectl get pods --all-namespaces -o wide | grep -v "Running\|Completed\|NAMESPACE"

# PVC все Bound
kubectl get pvc --all-namespaces

# Longhorn degraded допустимо, faulted — нет
kubectl -n longhorn-system get volumes.longhorn.io -o json | \
  jq '.items[] | {name: .metadata.name, robustness: .status.robustness}'
```

### 5.4. Обслуживание ноды

Обновление ОС, Kubernetes, containerd и т.д.

### 5.5. Возврат ноды и переход к следующей

```bash
# Вернуть ноду
kubectl uncordon <NODE_NAME>

# Дождаться healthy volumes перед drain следующей ноды
watch 'kubectl -n longhorn-system get volumes.longhorn.io -o json | \
  jq ".items[] | {name: .metadata.name, robustness: .status.robustness}"'
```

---

## 6. Чеклист перед drain

```
[ ] Все ноды Ready
[ ] Все Longhorn volumes healthy
[ ] Каждый volume имеет реплики на 2+ нодах
[ ] PV Reclaim Policy = Retain
[ ] StatefulSet PVC Retention = Retain
[ ] Longhorn Drain Policy = block-if-contains-last-replica
[ ] Дрейним строго по одной ноде
[ ] После uncordon дождались healthy volumes перед следующим drain
```
