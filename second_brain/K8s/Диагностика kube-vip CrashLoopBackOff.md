# Диагностика kube-vip CrashLoopBackOff

## 1. Симптомы

- Поды `kube-vip-master1` и `kube-vip-master3` в статусе CrashLoopBackOff
- Множественные рестарты kube-vip на всех мастерах
- Лидерство постоянно "прыгает" между мастерами

---

## 2. Порядок диагностики

### 2.1. Проверить статус подов

```bash
kubectl get pods -n kube-system -o wide | grep kube-vip
```

Обращать внимание на:
- колонку `RESTARTS` и время последнего рестарта
- все ли поды в Running или есть CrashLoopBackOff

### 2.2. Проверить логи kube-vip (текущие и предыдущего краша)

```bash
kubectl logs kube-vip-master1 -n kube-system --tail=50
kubectl logs kube-vip-master1 -n kube-system --previous --tail=50
```

Ключевые ошибки:
- `Failed to update lock optimistically: context deadline exceeded` — kube-vip не смог обновить lease через API server
- `failed to renew lease: context deadline exceeded` — lease renewal таймаут
- `lost leadership, restarting kube-vip` — потеря лидерства, pod крашится (by design)

### 2.3. Проверить конфигурацию lease

```bash
kubectl get lease plndr-cp-lock -n kube-system -o yaml
```

Обращать внимание на:
- `leaseDurationSeconds` — при значении 5 (дефолт kube-vip) очень мало запаса
- `leaseTransitions` — количество смен лидера, растущее число = нестабильность
- `holderIdentity` — кто сейчас лидер

### 2.4. Проверить параметры kube-vip pod

```bash
kubectl describe pod kube-vip-master1 -n kube-system
```

Ключевые env-переменные:
- `vip_leaseduration: 5` — время жизни lease (секунды)
- `vip_renewdeadline: 3` — таймаут обновления lease
- `vip_retryperiod: 1` — интервал повторных попыток

Значения 5/3/1 — агрессивные. Стандартные для k8s: 15/10/2.

### 2.5. Проверить логи kube-apiserver

```bash
kubectl logs kube-apiserver-master1 -n kube-system --tail=50
```

Ключевые ошибки:
- `grpc: addrConn.createTransport failed to connect` — API server не может подключиться к etcd
- `transport: authentication handshake failed: context canceled` — TLS handshake с etcd не проходит
- `apiserver was unable to write a JSON response: http: Handler timeout` — API server таймаутится на запросах
- `Post-timeout activity ... method="PUT" path=".../leases/plndr-cp-lock"` — конкретно обновление lease kube-vip таймаутится

Если есть ошибки подключения к etcd — проблема глубже, в etcd.

### 2.6. Проверить здоровье etcd

etcd может работать как systemd-сервис (kubespray) или static pod.

```bash
# Проверить статус сервиса (kubespray)
ssh root@<master-ip> "systemctl status etcd"

# Member list
ETCDCTL_API=3 etcdctl \
  --endpoints=https://10.10.1.171:2379,https://10.10.1.172:2379,https://10.10.1.173:2379 \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/node-master1.pem \
  --key=/etc/ssl/etcd/ssl/node-master1-key.pem \
  member list -w table

# Endpoint health
ETCDCTL_API=3 etcdctl \
  --endpoints=https://10.10.1.171:2379,https://10.10.1.172:2379,https://10.10.1.173:2379 \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/node-master1.pem \
  --key=/etc/ssl/etcd/ssl/node-master1-key.pem \
  endpoint health -w table
```

### 2.7. Проверить логи etcd на clock drift

```bash
ssh root@<master-ip> "journalctl -u etcd --since '5 min ago' --no-pager | grep clock.drift"
```

Если видим `prober found high clock drift` — проблема в рассинхронизации часов между мастерами.

### 2.8. Проверить синхронизацию времени

```bash
# Сравнить время на всех мастерах
for h in 10.10.1.171 10.10.1.172 10.10.1.173; do
  echo -n "$h: "; ssh root@$h "date -u '+%Y-%m-%d %H:%M:%S.%N'"
done

# Проверить статус NTP на каждом мастере
ssh root@<master-ip> "timedatectl timesync-status"
ssh root@<master-ip> "systemctl status systemd-timesyncd"
```

Обращать внимание на:
- `Offset` — отклонение от NTP сервера
- `Frequency` — скорость дрейфа (ppm)
- Таймауты подключения к NTP серверам в логах

---

## 3. Цепочка причин (данный инцидент)

```
NTP sync failure на master2 (таймауты ntp.ubuntu.com)
  → Clock drift ~3.2 секунды между мастерами
    → etcd TLS handshake failures (сертификаты чувствительны к времени)
      → API server не может надёжно общаться с etcd
        → Запросы на обновление lease таймаутятся
          → kube-vip теряет лидерство за 5 секунд
            → CrashLoopBackOff
```

---

## 4. Исправление

### 4.1. Немедленно — принудительная синхронизация времени

```bash
ssh root@<master-ip> "systemctl restart systemd-timesyncd"
```

Проверка после рестарта:

```bash
ssh root@<master-ip> "timedatectl timesync-status"
# Offset должен уменьшиться
```

Если timesyncd не делает step (медленный slew):

```bash
ssh root@<master-ip> "timedatectl set-ntp false && timedatectl set-ntp true"
```

### 4.2. Долгосрочно — переключить на chrony

systemd-timesyncd ненадёжен при больших drift'ах (делает slew вместо step). chrony справляется лучше.

```bash
apt install chrony
systemctl disable --now systemd-timesyncd
systemctl enable --now chrony
```

### 4.3. Опционально — увеличить lease таймеры kube-vip

Файл: `/etc/kubernetes/manifests/kube-vip.yaml` на каждом мастере.

```yaml
env:
  - name: vip_leaseduration
    value: "15"          # было 5
  - name: vip_renewdeadline
    value: "10"          # было 3
  - name: vip_retryperiod
    value: "2"           # было 1
```

Pod пересоздастся автоматически (static pod).

---

## 5. Проверка после исправления

```bash
# etcd больше не логирует clock drift
ssh root@<master-ip> "journalctl -u etcd --since '5 min ago' --no-pager | grep clock.drift"

# etcd cluster healthy
etcdctl endpoint health -w table

# kube-vip не рестартуется
kubectl get pods -n kube-system -l k8s-app=kube-vip -o wide

# Lease стабильно обновляется, leaseTransitions не растёт
kubectl get lease plndr-cp-lock -n kube-system -o yaml
```
