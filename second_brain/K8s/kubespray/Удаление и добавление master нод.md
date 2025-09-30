## 1. Удаление мастер-ноды

### 1.1. Подготовка инвентаря

В `inventory/k8s/inventory.yaml` временно закомментировать или удалить ноду из блока `all`, чтобы Ansible не пытался её опрашивать:

```yaml
# master1:
#   ansible_host: 192.168.88.193
#   ip: 192.168.88.193
#   access_ip: 192.168.88.193
```

### 1.2. Запуск playbook на удаление

```bash
sudo env "PATH=$PATH" ansible-playbook -i inventory/k8s/inventory.yaml remove-node.yml \
  -u root -b \
  -e reset_nodes=false \
  -e allow_ungraceful_removal=true \
  -e ignore_assert_errors=yes \
  -e node=master1
```

### 1.3. Чистка в Kubernetes

Проверить, не осталась ли нода в кластере:

```bash
kubectl get nodes
```

Если удалённая нода всё ещё есть в списке:

```bash
kubectl delete node master1
```

### 1.4. Чистка инвентаря

После успешного удаления окончательно удалить закомментированные строки из файла `inventory.yaml`.

---

## 2. Добавление новой мастер-ноды

### 2.1. Добавление в `kube_control_plane`

Добавить **в конец списка**:

```yaml
kube_control_plane:
  hosts:
    master3:
    master2:
    master1:
```

### 2.2. Добавление в `etcd`

Аналогично, добавить **в конец списка**:

```yaml
etcd:
  hosts:
    master3:
    master2:
    master1:
```

Запуск:

```bash
sudo env "PATH=$PATH" ansible-playbook -i inventory/k8s/inventory.yaml cluster.yml \
  -u root -b \
  --limit=etcd,kube_control_plane \
  -e ignore_assert_errors=yes \
  -e etcd_retries=10
```

---

## 3. Проверка состояния ETCD

На 2 мастер-ноде выполнить:

```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/member-master2.pem \
  --key=/etc/ssl/etcd/ssl/member-master2-key.pem \
  member list
```

Убедиться, что новые мастера отображаются в составе кворума, а старой удалённой ноды нет.

---


# Если нода по каким то причинам не удаляется австоматически:

### Шаг 1. Выведи ноду из расписания
```bash
kubectl cordon master2
kubectl drain master2 --ignore-daemonsets
```

### Шаг 2. Удали её из кластера Kubernetes
```bash
kubectl delete node master2
```

### Шаг 3. Удали ноду из инвентаря Kubespray

В `inventory/k8s/inventory.yaml` нужно убрать блок:
```yaml
master2:
  ansible_host: 192.168.88.192
  ip: 192.168.88.192
  access_ip: 192.168.88.192
```

Удаляем мертвую ноду из ETCD кластера:
```shell
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379   --cacert=/etc/ssl/etcd/ssl/ca.pem   --cert=/etc/ssl/etcd/ssl/member-master1.pem   --key=/etc/ssl/etcd/ssl/member-master1-key.pem   member list # тут получаем айди удаляемой ноды ETCD
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379   --cacert=/etc/ssl/etcd/ssl/ca.pem   --cert=/etc/ssl/etcd/ssl/member-master1.pem   --key=/etc/ssl/etcd/ssl/member-master1-key.pem   member remove 719be21e733cfe96
```

На всех мастерах выполняем удаление ETCD из конфигов
```shell
nano /etc/kubernetes/manifests/kube-apiserver.yaml

- --etcd-servers=https://192.168.88.191:2379,https://192.168.88.193:2379,https://192.168.88.192:2379 # В данной строке удаляем мертвую ноду etcd
```

```shell
nano /etc/etcd.env

ETCD_INITIAL_CLUSTER=etcd1=https://192.168.88.191:2380,etcd2=https://192.168.88.193:2380,etcd3=https://192.168.88.192:2380 # В данной строке удаляем мертвую ноду etcd
```

### Шаг 4. Обновить конфигурацию кластера Kubespray
```bash
sudo env "PATH=$PATH" ansible-playbook -e ignore_assert_errors=yes -i inventory/k8s/inventory.yaml cluster.yml -u root -b
```

-e ignore_assert_errors=yes устанавливается для того что бы игнорировать ошибку etcd что не хорошо иметь 2 ноды в кластере необходимо нечетное количество

Это почистит конфигурацию etcd и control-plane от удалённого мастера.  
Если мастер полностью "мертвый" (невозможно выполнить drain), можно сразу `kubectl delete node` и убрать его из inventory

---

## 2. Добавление мастер-ноды обратно

### Шаг 1. Верни ноду в inventory

Добавь блок обратно в `inventory/mycluster/inventory.yaml`:
```yaml
master2:
  ansible_host: 192.168.88.192
  ip: 192.168.88.192
  access_ip: 192.168.88.192
```

### Шаг 2. Убедись, что на новой ноде чистая ОС

### Шаг 3. Прогони cluster.yml для добавления

```bash
sudo env "PATH=$PATH" ansible-playbook -i inventory/k8s/inventory.yaml cluster.yml -u root -b --limit=etcd,kube_control_plane -e ignore_assert_errors=yes -e etcd_retries=10
```

Проверить на всех мастерах данные конфиги что бы в них была новая нода ETCD:
```shell
nano /etc/kubernetes/manifests/kube-apiserver.yaml
nano /etc/etcd.env
```

---

## 3. Проверка состояния

После удаления/добавления всегда проверяй:
```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
kubectl get endpoints -n default kubernetes
```

- Все мастера должны быть в статусе **Ready**.
- В `etcd` должно быть нужное количество членов:
```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/member-master2.pem \
  --key=/etc/ssl/etcd/ssl/member-master2-key.pem \
  member list
```


