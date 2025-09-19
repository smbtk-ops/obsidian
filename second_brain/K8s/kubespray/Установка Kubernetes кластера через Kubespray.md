
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
ingress_nginx_enabled: true
metallb_enabled: true
```

_(при желании можно включить и другие аддоны — список есть в этом же файле)._

Для использования metalLB
В файле inventory/k8s/group_vars/k8s_cluster/k8s-cluster.yml:

```yaml
kube_proxy_strict_arp: true
```

Для использования своего реджистри

В нексусе добавить docker-proxy репозитории:
docker-proxy-quay "https://quay.io"
docker-proxy-k8s   "https://registry.k8s.io"
и добавить их в  docker-images-group

В файле inventory/k8s/group_vars/all/all.yml добавить:

```
registry_host: "nexus.dev.mllive.by"

kube_image_repo: "{{ registry_host }}/docker-images-group"
gcr_image_repo: "{{ registry_host }}/docker-images-group"
docker_image_repo: "{{ registry_host }}/docker-images-group"
quay_image_repo: "{{ registry_host }}/docker-images-group"
```

---

## 3. Запуск установки

Выполняем установку кластера:

```bash
sudo env "PATH=$PATH" ansible-playbook -i inventory/k8s/inventory.yaml cluster.yml -u root -f 1 -b --extra-vars "@inventory/k8s/group_vars/k8s_cluster/addons.yml"
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

### Проблема:

Ошибка вида:
```
TASK [kubernetes/preinstall : Stop if access_ip is not pingable] **************************************************************************************************************************************************
fatal: [master1]: FAILED! => {"changed": false, "cmd": "ping -c1 192.168.88.191", "msg": "[Errno 2] No such file or directory: b'ping'", "rc": 2, "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}
fatal: [master2]: FAILED! => {"changed": false, "cmd": "ping -c1 192.168.88.192", "msg": "[Errno 2] No such file or directory: b'ping'", "rc": 2, "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}
fatal: [worker1]: FAILED! => {"changed": false, "cmd": "ping -c1 192.168.88.194", "msg": "[Errno 2] No such file or directory: b'ping'", "rc": 2, "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}
fatal: [worker2]: FAILED! => {"changed": false, "cmd": "ping -c1 192.168.88.195", "msg": "[Errno 2] No such file or directory: b'ping'", "rc": 2, "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}
fatal: [master3]: FAILED! => {"changed": false, "cmd": "ping -c1 192.168.88.193", "msg": "[Errno 2] No such file or directory: b'ping'", "rc": 2, "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}

```

### Решение:
Зайди на каждую ноду (master1, master2, master3, worker1, worker2).  
Для Ubuntu/Debian:

`sudo apt update`
`sudo apt install -y iputils-ping`

После этого снова запускаем установку:
```bash
sudo env "PATH=$PATH" ansible-playbook -i inventory/k8s/inventory.yaml cluster.yml -u root -f 1 -b --extra-vars "@inventory/k8s/group_vars/k8s_cluster/addons.yml"
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