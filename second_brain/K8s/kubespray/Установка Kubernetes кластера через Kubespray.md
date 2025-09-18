
## 1. Создание инвентарного файла

Создаём файл `./inventory/k8s/inventory.yaml` и добавляем описание мастер и воркер-нод:

```yaml
all:
  hosts:
    master1:
      ansible_host: 192.168.88.190
      ip: 192.168.88.190
      access_ip: 192.168.88.190
    master2:
      ansible_host: 192.168.88.191
      ip: 192.168.88.191
      access_ip: 192.168.88.191
    master3:
      ansible_host: 192.168.88.192
      ip: 192.168.88.192
      access_ip: 192.168.88.192
    worker1:
      ansible_host: 192.168.88.193
      ip: 192.168.88.193
      access_ip: 192.168.88.193
    worker2:
      ansible_host: 192.168.88.194
      ip: 192.168.88.194
      access_ip: 192.168.88.194

  children:
    kube_control_plane:
      hosts:
        master1:
        master2:
        master3:
    kube_node:
      hosts:
        worker1:
        worker2:
    etcd:
      hosts:
        master1:
        master2:
        master3:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```

---

## 2. Настройка аддонов

Открываем файл `inventory/k8s/group_vars/k8s_cluster/addons.yml` и активируем необходимые дополнения:

```yaml
helm_enabled: true
metrics_server_enabled: true
local_path_provisioner_enabled: true
metallb_enabled: true
```

_(при желании можно включить и другие аддоны — список есть в этом же файле)._

---

## 3. Запуск установки

Выполняем установку кластера:

```bash
ansible-playbook -i inventory/k8s/inventory.yaml cluster.yml -u root -b --extra-vars "@inventory/k8s/group_vars/k8s_cluster/addons.yml"
```

---

## 4. Возможные проблемы и их решение

### Проблема:

Ошибка вида:

```
Файл существует: .../kubespray/inventory/k8s/credentials/<hash>.ansible_lockfile
```

### Решение:
!!!Данное решение можно использовать перед первым запуском что бы сразу избежать возможной ошибки!!!

Открываем `inventory/k8s/group_vars/k8s_cluster/k8s-cluster.yml` и заменяем блок `kubeadm_certificate_key` на:

```yaml
kubeadm_certificate_key: >-
  {{
    (lookup('file', credentials_dir + '/kubeadm_certificate_key.creds')
    if lookup('pipe', 'test -f ' + credentials_dir + '/kubeadm_certificate_key.creds && echo yes || echo no') == 'yes'
    else lookup('password', credentials_dir + '/kubeadm_certificate_key.creds length=64 chars=hexdigits'))
    | trim | lower
  }}
```

После этого снова запускаем установку:

```bash
ansible-playbook -i inventory/k8s/inventory.yaml cluster.yml -u root -b --extra-vars "@inventory/k8s/group_vars/k8s_cluster/addons.yml"
```

---

### **Последние шаги:**

Скопировать конфиг файл:
```
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Для проверки:
`kubectl get nodes`

Сделай отдельный YAML-файл, например `metallb-layer2.yaml`:
```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ip-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.88.195-192.168.88.199  # Указать интересующий пул
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

Применить его:
`kubectl apply -f metallb-layer2.yaml`