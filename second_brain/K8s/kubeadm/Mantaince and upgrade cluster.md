
# Инструкция по обновлению Kubernetes с помощью kubeadm

> **Рекомендуется** обновляться только на **одну минорную версию за раз**.  
> Например: с `1.19` → `1.20` → `1.21`, а не напрямую `1.19` → `1.21`.

---

## 1. Подготовка репозиториев (все ноды)

1. Создаём файл:

```bash
sudo nano /etc/apt/sources.list.d/kubernetes.list
```

2. Добавляем необходимые версии:

```plaintext
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Репозитории можно глянуть тут:
https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/#verifying-if-the-kubernetes-package-repositories-are-used

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

3. Проверяем доступные версии:

```bash
sudo apt update
sudo apt-cache madison kubeadm
```

---

## 2. Команды для управления доступностью нод

| Команда                                                                   | Действие                                             |
| ------------------------------------------------------------------------- | ---------------------------------------------------- |
| `kubectl cordon node01`                                                   | Запрещает планировщику назначать новые поды на узел. |
| `kubectl drain node01 --ignore-daemonsets`                                | Переносит все поды (кроме DaemonSet) на другие узлы. |
| `kubectl drain node01 --ignore-daemonsets --force`                        | То же, но включает удаление подов без контроллеров.  |
| `kubectl uncordon node01`                                                 | Разрешает планировщику снова назначать поды на узел. |
| `kubectl drain node01 --ignore-daemonsets --force --delete-emptydir-data` |                                                      |

Что произойдёт при drain:                                                                                                                                                                                          
                                                                                                                                                                                                                     
  1. Replica count >= 2 — Longhorn перестроит реплику на другой ноде автоматически. Данные не потеряются, volume останется доступным.                                                                                
  2. Replica count = 1 и она на этой ноде — volume станет degraded/unavailable, workload с этим PVC не запустится на другой ноде.         
                                                                                                                                                                                                                     
  Что делать перед drain:                                                                                                                                                                                            
                                                                                                                                                                                                                     
  # Проверь что все volumes healthy                                                                                                                                                                                  
  kubectl -n longhorn-system get volumes.longhorn.io                                                                                                                                                                 
                                                                                                                                                                                                                     
  Убедись что:                                                                                                                                                                                                       
  - У volumes numberOfReplicas >= 2                                                                                                                                                                                  
  - Статус robustness: healthy (реплики есть на других нодах)                                                                                                                                                        
                                                             
  Настройка в Longhorn:                                                                                                                                                                                              
                                                                                                                                                                                                                     
  В Longhorn UI → Settings → Node Drain Policy — определяет поведение при drain:                                                                                                                                     
  - block-if-contains-last-replica — не даст дрейнить если на ноде последняя реплика (безопасно)                                                                                                                     
  - allow-if-replica-is-stopped — разрешит если реплика остановлена                                                                                                                                                  
  - always-allow — разрешит всегда (опасно)                                                                                               
                                                                                                                                                                                                                     
  Рекомендация: перед обновлением worker-ноды проверь что все volume имеют реплики на других нодах. Если нет — временно увеличь replica count или добавь реплику вручную через Longhorn UI. 
---

## 3. Обновление **первого** control plane узла

### 3.1. Обновляем пакеты `kubeadm`, `kubelet`, `kubectl`

```bash
sudo apt-mark unhold kubeadm kubelet kubectl && \
sudo apt-get update && \
sudo apt-get install -y kubeadm='1.34.x-*' kubelet='1.34.x-*' kubectl='1.34.x-*' && \
sudo apt-mark hold kubeadm kubelet kubectl
```

### 3.2. Проверяем версию

```bash
kubeadm version
```

### 3.3. Планируем апгрейд

```bash
sudo kubeadm upgrade plan
```

> Можно добавить `--certificate-renewal=false`, если не хотите обновлять сертификаты.

### 3.4. Применяем апгрейд

```bash
sudo kubeadm upgrade apply v1.34.x
```

**Успешное сообщение:**

```
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.34.x". Enjoy!
```

---

## 4. Обновление CNI (если требуется)

Проверьте документацию вашего CNI.  
Для Calico: [Calico Requirements](https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements#kubernetes-requirements).

---

## 5. Обновление **остальных** control plane узлов

Выполните **те же шаги**, но вместо:

```bash
sudo kubeadm upgrade apply v1.34.x
```

используйте:

```bash
sudo kubeadm upgrade node
```

Плагин CNI можно **не обновлять**, если он развёрнут как DaemonSet.

---

## 6. Обновление worker-нод

1. На **мастер-ноде**:

```bash
kubectl drain <node-to-drain> --ignore-daemonsets --force
```

2. На **worker-ноде** обновляем `kubelet` и `kubectl`:

```bash
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && \
sudo apt-get install -y kubelet='1.34.x-*' kubectl='1.34.x-*' && \
sudo apt-mark hold kubelet kubectl
```

3. Перезапускаем `kubelet`:

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

4. Освобождаем ноду на **мастер-ноде**:

```bash
kubectl uncordon <node-to-uncordon>
```

---

### Проверка **на мастер ноде**:

kubectl get no -o wide

## Полезные ссылки

- 📄 [Официальная инструкция kubeadm upgrade](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

- 📄 [Управление сертификатами Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/)

- 📄 [Calico Kubernetes Requirements](https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements#kubernetes-requirements)

---

## 7. Ротация сертификатов (без апгрейда кластера)

Сертификаты K8s выдаются на 1 год. CA — на 10 лет. Etcd (kubespray) — на 100 лет.

### 7.1. Проверка сроков

```bash
# Все K8s сертификаты (kubeadm)
kubeadm certs check-expiration

# Все файлы PKI подробно
for cert in /etc/kubernetes/pki/*.crt; do
  echo -n "$(basename $cert): "
  openssl x509 -enddate -noout -in $cert
done

# Сертификаты в kubeconfig файлах
for conf in /etc/kubernetes/*.conf; do
  echo -n "$(basename $conf): "
  grep client-certificate-data $conf | awk '{print $2}' | \
    base64 -d | openssl x509 -enddate -noout 2>/dev/null || echo 'no client cert'
done

# Etcd сертификаты (kubespray хранит в /etc/ssl/etcd/ssl/)
for cert in /etc/ssl/etcd/ssl/*.pem; do
  case $cert in *-key*) continue;; esac
  echo -n "$(basename $cert): "
  openssl x509 -enddate -noout -in $cert
done

# SAN apiserver (проверить что VIP включён)
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A1 'Subject Alternative Name'
```

### 7.2. Процедура обновления

Выполняется **по одному мастеру**, начиная с последнего (master3 → master2 → master1).

```bash
# 1. Бэкап сертификатов
mkdir -p /etc/kubernetes/conf.bak
cp -r /etc/kubernetes/pki /etc/kubernetes/pki.bak
cp /etc/kubernetes/*.conf /etc/kubernetes/conf.bak/

# 2. Обновление всех сертификатов
kubeadm certs renew all

# 3. Рестарт kubelet (пересоздаст static pods с новыми сертификатами)
systemctl restart kubelet
```

> `super-admin.conf` существует только на master1 (где выполнялся `kubeadm init`). На master2/3 будет `MISSING!` — это нормально.

### 7.3. Проверка после обновления каждого мастера

```bash
# 1. Сертификаты обновились
kubeadm certs check-expiration

# 2. Control plane pods работают
crictl pods | grep -E "apiserver|controller|scheduler|etcd"

# 3. API отвечает
curl -sk https://127.0.0.1:6443/healthz

# 4. Все ноды Ready
kubectl get nodes

# 5. Etcd здоров
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  endpoint health
```

Если все 5 пунктов ок — переходить к следующему мастеру.

### 7.4. Обновление kubeconfig на клиентах

После обновления всех мастеров — скопировать новый `admin.conf` с master1:

```bash
scp root@<master1-ip>:/etc/kubernetes/admin.conf ~/.kube/config
```

Обновить также в:
- Lens (kubeconfigs/)
- Jenkins (если использует kubeconfig)
- CI/CD пайплайны

> Старый kubeconfig продолжит работать до истечения старого сертификата (тот же CA), но лучше обновить.

### 7.5. Откат (если что-то пошло не так)

```bash
# Восстановить бэкап
cp -r /etc/kubernetes/pki.bak/* /etc/kubernetes/pki/
cp /etc/kubernetes/conf.bak/* /etc/kubernetes/
systemctl restart kubelet
```

Старые сертификаты остаются валидными до исходного срока. Два других мастера при этом не затронуты.

### 7.6. Что обновляет `kubeadm certs renew`, а что нет

| Компонент | Обновляется | Примечание |
|-----------|------------|------------|
| apiserver, front-proxy, kubeconfigs | Да | Основные K8s сертификаты |
| CA (ca.crt, front-proxy-ca.crt) | Нет | Действуют 10 лет, обновляются при апгрейде через kubespray |
| Etcd сертификаты | Нет | Управляются kubespray, выданы на 100 лет |
| Kubelet client cert | Нет | Автоматическая ротация через bootstrap token |

### 7.7. Лог проведённых ротаций

| Дата | Кластер | Было | Стало | Кто |
|------|---------|------|-------|-----|
| 2026-04-06 | DEV (192.168.88.191-193) | 28 янв 2027 | 6 апр 2027 | kubeadm certs renew all |
| 2026-04-06 | PROD (10.10.1.171-173) | 5 дек 2026 | 6 апр 2027 | kubeadm certs renew all |

