
## 1. Установка MetalLB в кластер

Устанавливаем MetalLB и все необходимые CRD, контроллеры и службы:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
````

После выполнения команды MetalLB будет установлен в namespace `metallb-system`.

---

## 2. Создание IP-пула и объявления L2Advertisement

Создаём файл `ip_address_pool.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ip-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.88.185-192.168.88.190
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2adv
  namespace: metallb-system
spec:
  ipAddressPools:
    - ip-pool
```

**Пояснения:**

- **IPAddressPool** — задаёт диапазон IP-адресов, которые MetalLB сможет раздавать сервисам типа `LoadBalancer`.
- **L2Advertisement** — сообщает, что эти адреса будут анонсироваться в локальной сети по L2 (ARP/NDP).

---

## 3. Применение конфигурации

Применяем конфигурацию:

```bash
kubectl apply -f ip_address_pool.yaml
```

---

## 4. Проверка работы

Создаём тестовый сервис типа `LoadBalancer`:

```bash
kubectl expose deployment nginx --port=80 --type=LoadBalancer
```

Проверяем назначенный IP:

```bash
kubectl get svc
```

IP должен быть из диапазона `192.168.88.185-192.168.88.190`.

Проверяем доступность сервиса по IP из локальной сети.

---

## 5. Полезные команды для диагностики

Проверить логи контроллеров:

```bash
kubectl logs -n metallb-system deploy/controller
kubectl logs -n metallb-system deploy/speaker
```

Проверить занятые IP и активные L2Advertisement:

```bash
kubectl get ipaddresspools -n metallb-system
kubectl get l2advertisements -n metallb-system
```
