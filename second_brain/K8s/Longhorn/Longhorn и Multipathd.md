## 1. Проблема

На Ubuntu 24.04 (и других дистрибутивах с включённым `multipathd`) Longhorn не может отформатировать новые тома. Pod остаётся в `ContainerCreating`, а в events ошибка:

```
MountVolume.MountDevice failed for volume "pvc-xxx":
  format of disk "/dev/longhorn/pvc-xxx" failed:
  /dev/longhorn/pvc-xxx is apparently in use by the system;
  will not make a filesystem here!
```

---

## 2. Причина

`multipathd` — сервис для управления multipath I/O (несколько путей к одному SAN-диску). На серверах без SAN storage он не нужен, но по умолчанию включён в Ubuntu.

Когда Longhorn создаёт блочное устройство (`/dev/longhorn/pvc-xxx`), `multipathd` обнаруживает его и захватывает — создаёт device-mapper устройство (`/dev/dm-X`). После этого `mkfs` отказывается форматировать оригинальное устройство, потому что оно уже "in use".

Проверить, захвачены ли устройства:

```bash
sudo multipath -ll
```

Если в выводе есть устройства Longhorn (IET, VIRTUAL-DISK) — `multipathd` мешает.

---

## 3. Решение

### 3.1. Вариант A: blacklist устройств в multipath.conf

Добавить blacklist для блочных устройств, которые не являются SAN:

```bash
sudo tee /etc/multipath.conf > /dev/null << 'EOF'
defaults {
    user_friendly_names yes
}

blacklist {
    devnode "^sd[a-z0-9]+"
}
EOF

sudo systemctl restart multipathd
```

Проверить, что устройства освобождены:

```bash
sudo multipath -ll
# Должен быть пустой вывод
```

Использовать этот вариант, если `multipathd` нужен для других устройств (например, SAN с fibre channel).

### 3.2. Вариант B: полное отключение multipathd

Если SAN storage не используется (типичный случай для bare-metal K8s):

```bash
sudo systemctl disable --now multipathd.socket multipathd.service
```

Для всех воркеров через Ansible:

```bash
ansible workers -m shell -a "systemctl disable --now multipathd.socket multipathd.service" -u ansible --become
```

---

## 4. Применение на все ноды кластера

Важно применить fix на **все worker-ноды**, а не только на ту, где произошла ошибка. Longhorn может разместить том на любой ноде.

### 4.1. Через Ansible (blacklist)

```bash
ansible workers -m copy -a "content='defaults {\n    user_friendly_names yes\n}\n\nblacklist {\n    devnode \"^sd[a-z0-9]+\"\n}\n' dest=/etc/multipath.conf" -u ansible --become

ansible workers -m shell -a "systemctl restart multipathd" -u ansible --become
```

### 4.2. Проверка на всех нодах

```bash
ansible workers -m shell -a "multipath -ll" -u ansible --become
```

---

## 5. Восстановление зависшего тома

Если pod уже завис с ошибкой форматирования после применения fix:

```bash
# 1. Масштабировать StatefulSet в 0
kubectl scale statefulset <name> -n <namespace> --replicas=0

# 2. Дождаться удаления пода
kubectl get pods -n <namespace> -w

# 3. Удалить проблемный PVC
kubectl delete pvc <pvc-name> -n <namespace>

# 4. Пересоздать PVC
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <pvc-name>
  namespace: <namespace>
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi
EOF

# 5. Масштабировать обратно
kubectl scale statefulset <name> -n <namespace> --replicas=1
```

---

## 6. Диагностика

| Команда | Назначение |
|---|---|
| `kubectl describe pod <pod>` | Проверить events (FailedMount) |
| `kubectl get pvc -n <ns>` | Проверить статус PVC (Bound/Pending) |
| `kubectl get volumes.longhorn.io -n longhorn-system` | Статус Longhorn томов |
| `sudo multipath -ll` | Проверить захваченные устройства на ноде |
| `sudo multipathd show status` | Статус демона multipathd |
| `lsblk` | Дерево блочных устройств (видно dm- устройства) |
