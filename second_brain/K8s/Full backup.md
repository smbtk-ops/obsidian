## 1. Цель и принципы

Чтобы восстановить **точную копию**, нужно бэкапить всё, что делает кластер уникальным:
1. **etcd** — база данных Kubernetes (все объекты API: Pod, Deployment, Secret, ConfigMap, CRD, RBAC и т.д.);
2. **PKI** — сертификаты и ключи (`/etc/kubernetes/pki`);
3. **манифесты статических подов** (`/etc/kubernetes/manifests`);
4. **конфигурации kubelet и kubeadm** (`/var/lib/kubelet`, `/etc/kubernetes/kubeadm.yaml`);
5. **persistent volumes** (PV/PVC) — данные приложений;
6. **ресурсы namespace-ов, ингрессов, сервисов, CRD**;
7. **бэкап YAML всех ресурсов** для независимости от etcd-дампа.

---

## 2. Полный цикл резервного копирования

### 2.1 Бэкап etcd

Если кластер создавался через kubeadm или kubespray, etcd работает как pod на control-plane.

**Команда для snapshot:**

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-snapshot-$(date +%F-%H-%M).db
```

**Проверить:**

```bash
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot-2025-11-10-10-00.db
```

Рекомендуется хранить **минимум 7 ротаций** и выгружать их на внешний storage (S3, MinIO, NFS и т.п.).

---

### 2.2 Бэкап PKI и конфигурации

```bash
tar czf /backup/k8s-pki-$(date +%F).tar.gz /etc/kubernetes/pki
tar czf /backup/k8s-manifests-$(date +%F).tar.gz /etc/kubernetes/manifests
tar czf /backup/kubeadm-config-$(date +%F).tar.gz /etc/kubernetes/kubeadm.yaml
```

---

### 2.3 Бэкап описаний ресурсов (YAML)

Даже если etcd-снапшот есть, YAML нужны для гибкости восстановления и клонирования в другой кластер.

**Массовый экспорт всех объектов:**
```bash
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  mkdir -p /backup/yaml/$ns
  kubectl get all,configmap,secret,pvc,serviceaccount,ingress,role,rolebinding,crd -n $ns -o yaml > /backup/yaml/$ns/resources.yaml
done
```

**Или одним инструментом:**
```bash
kubectl get all --all-namespaces -o yaml > /backup/yaml/all-resources.yaml
```

---

### 2.4 Бэкап Persistent Volume (данные)

Для PV возможны три подхода:

#### a) CSI Snapshot (современный способ)

Если StorageClass поддерживает snapshot:

```bash
kubectl create -f VolumeSnapshotClass.yaml
kubectl create -f VolumeSnapshot.yaml
```

Затем сохранить содержимое snapshot в S3/MinIO.

#### b) Rsync/Restic/Velero

- **Velero** — промышленный стандарт:
    
    ```bash
    velero install --provider aws --plugins velero/velero-plugin-for-aws:v1.8.0 \
      --bucket my-backup-bucket --backup-location-config region=us-east-1
    ```
    
    Бэкап кластера:
    
    ```bash
    velero backup create full-backup-$(date +%F) --include-namespaces all
    ```
    
    Восстановление:
    
    ```bash
    velero restore create --from-backup full-backup-2025-11-10
    ```
    

#### c) Rsync напрямую

```bash
rsync -a /var/lib/kubelet/pods/ /backup/volumes/
```

---

### 2.5 Автоматизация

Скрипт-cron на мастер-ноде (пример):

```bash
#!/bin/bash
set -e
BACKUP_DIR="/backup/$(date +%F-%H%M)"
mkdir -p $BACKUP_DIR

ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save $BACKUP_DIR/etcd.db

tar czf $BACKUP_DIR/k8s-pki.tar.gz /etc/kubernetes/pki
kubectl get all,cm,secret,svc,ingress,pvc -A -o yaml > $BACKUP_DIR/resources.yaml

rclone copy $BACKUP_DIR remote:cluster-backups/$(hostname)/
```

---

## 3. Восстановление полной копии

### 3.1 Разворачиваем чистый кластер
Создаём пустой Kubernetes (той же версии).
### 3.2 Восстанавливаем etcd
1. Останавливаем kube-apiserver (или kubelet, если статические поды).
2. Восстанавливаем снапшот:
  ```bash
   systemctl stop kubelet
   ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
   --data-dir=/var/lib/etcd
   ```
    
3. Копируем PKI:
  ```bash
   tar xzf /backup/k8s-pki.tar.gz -C /etc/kubernetes/
   ```

4. Запускаем kubelet:

  ```bash
   systemctl start kubelet
   ```

5. Проверяем `kubectl get nodes`.

---
### 3.3 Восстановление вручную из YAML (альтернатива)
Если кластер новый (другая инфраструктура):
```bash
kubectl apply -f /backup/yaml/all-resources.yaml
```

При необходимости заменить StorageClass и IP-адреса.

### 3.4 Восстановление PVC

Если использовался Velero:
```bash
velero restore create --from-backup full-backup-2025-11-10
```

Или вручную через rsync или snapshot.

---
### 3.5 Проверка целостности
- `kubectl get nodes,pods -A` — все поды должны быть в состоянии Running;
- `kubectl describe pod` — сверяем конфиги;
- проверяем DNS, ingress, secrets;
- тестируем приложение.

---

## 4. Минимально достаточный набор для полной копии

|Компонент|Что сохраняем|Где восстанавливаем|
|---|---|---|
|etcd snapshot|`etcd-snapshot.db`|`/var/lib/etcd`|
|Сертификаты PKI|`/etc/kubernetes/pki`|`/etc/kubernetes/pki`|
|kubeadm config|`/etc/kubernetes/kubeadm.yaml`|kubeadm init/restore|
|Манифесты static pods|`/etc/kubernetes/manifests`|те же|
|YAML ресурсов|`/backup/yaml/all-resources.yaml`|`kubectl apply`|
|Persistent Volume|snapshot / rsync / velero|восстановление storage|

- Версия `etcdctl` и `etcd` должна совпадать.
- Регулярно тестируйте **recovery на отдельном стенде**.
- Бэкап храните **вне кластера**.
- Для продакшена — используйте **Velero + Restic + S3/MinIO**.
- Для bare-metal — добавьте **node-level бэкап** (tar `/etc`, `/var/lib/kubelet`).

---


## Полный план резервного копирования Kubernetes-кластера

### Что необходимо сохранить

Кластер состоит из нескольких критичных компонентов:

**Control Plane данные:**

- etcd — хранилище всех объектов кластера (deployments, services, configmaps, secrets)
- Сертификаты PKI для всех компонентов
- Конфигурационные файлы kubeadm, kubelet

**Workload данные:**

- Persistent Volumes — данные приложений
- Состояние приложений (databases, stateful sets)

**Конфигурация инфраструктуры:**

- Манифесты всех ресурсов
- Helm charts и releases
- Custom Resource Definitions
- RBAC политики

### Пошаговый план резервирования

**Шаг 1: Резервное копирование etcd**

etcd — сердце кластера. Без него восстановление невозможно.

Сначала определи где работает etcd:

```bash
kubectl get pods -n kube-system | grep etcd
```

Для резервирования используй встроенную утилиту etcdctl:

```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

Проверь целостность снимка:

```bash
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot-*.db --write-out=table
```

Если у тебя multi-master setup, делай бекап с любого master-узла — данные идентичны.

**Шаг 2: Сохранение PKI инфраструктуры**

Без сертификатов кластер не поднимется:

```bash
tar -czf /backup/pki-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
  /etc/kubernetes/pki/ \
  /etc/kubernetes/*.conf
```

Это включает:

- ca.crt, ca.key — корневой CA
- apiserver.crt, apiserver.key
- apiserver-kubelet-client.crt
- front-proxy-ca.crt
- sa.key, sa.pub — для service accounts
- admin.conf, controller-manager.conf, scheduler.conf

**Шаг 3: Экспорт всех манифестов ресурсов**

Используй инструмент для массового экспорта:

```bash
mkdir -p /backup/manifests
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  mkdir -p /backup/manifests/$ns
  for resource in $(kubectl api-resources --verbs=list --namespaced -o name); do
    kubectl get $resource -n $ns -o yaml > /backup/manifests/$ns/$resource.yaml 2>/dev/null
  done
done
```

Для cluster-scoped ресурсов:

```bash
mkdir -p /backup/manifests/cluster
for resource in $(kubectl api-resources --verbs=list --namespaced=false -o name); do
  kubectl get $resource -o yaml > /backup/manifests/cluster/$resource.yaml 2>/dev/null
done
```

**Шаг 4: Резервирование Persistent Volumes**

Для каждого PV нужна своя стратегия:

Если используешь CSI драйверы (например AWS EBS, GCE PD), создавай snapshots через провайдера:

```bash
kubectl get pv -o json | jq -r '.items[] | select(.spec.csi) | .metadata.name' | while read pv; do
  kubectl patch pv $pv -p '{"spec":{"csi":{"volumeAttributes":{"snapshot":"true"}}}}'
done
```

Для generic volumes используй Velero (далее подробнее).

**Шаг 5: Установка и настройка Velero**

Velero — индустриальный стандарт для бекапов Kubernetes. Поддерживает работу с S3, Azure Blob, GCS.

Установка:

```bash
wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz
tar -xvf velero-v1.12.0-linux-amd64.tar.gz
sudo mv velero-v1.12.0-linux-amd64/velero /usr/local/bin/
```

Настройка для S3 (подставь свои данные):

```bash
cat > credentials-velero <<EOF
[default]
aws_access_key_id = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY
EOF

velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket kubernetes-backups \
  --secret-file ./credentials-velero \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --use-volume-snapshots=true
```

Создай полный бекап:

```bash
velero backup create full-cluster-backup \
  --include-namespaces '*' \
  --snapshot-volumes=true \
  --include-cluster-resources=true
```

Настрой расписание для автоматизации:

```bash
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces '*' \
  --snapshot-volumes=true \
  --ttl 720h
```

**Шаг 6: Резервирование Helm releases**

Если используешь Helm:

```bash
mkdir -p /backup/helm
helm list --all-namespaces -o json > /backup/helm/releases.json

helm list --all-namespaces --short | while read release; do
  ns=$(helm list --all-namespaces | grep $release | awk '{print $2}')
  helm get values $release -n $ns > /backup/helm/${release}-values.yaml
  helm get manifest $release -n $ns > /backup/helm/${release}-manifest.yaml
done
```

**Шаг 7: Сохранение конфигурации узлов**

На каждом узле кластера:

```bash
tar -czf /backup/node-config-$(hostname)-$(date +%Y%m%d).tar.gz \
  /etc/kubernetes/ \
  /var/lib/kubelet/config.yaml \
  /etc/systemd/system/kubelet.service.d/
```

**Шаг 8: Документирование инфраструктуры**

Создай текстовый файл с критичной информацией:

```bash
cat > /backup/cluster-info.txt <<EOF
Cluster Name: $(kubectl config current-context)
API Server: $(kubectl config view -o jsonpath='{.clusters[0].cluster.server}')
Kubernetes Version: $(kubectl version --short)
Nodes:
$(kubectl get nodes -o wide)

CNI Plugin: $(kubectl get pods -n kube-system | grep -E 'calico|flannel|weave|cilium' | head -1)

Ingress Controller:
$(kubectl get pods -n ingress-nginx -o wide 2>/dev/null || echo "Not found")

Storage Classes:
$(kubectl get sc)

Backup Date: $(date)
EOF
```

### План восстановления на чистой системе

**Этап 1: Подготовка инфраструктуры**

Разверни чистые серверы с той же ОС и версией. Установи базовые зависимости:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

Установи container runtime (containerd):

```bash
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

Установи kubeadm, kubelet, kubectl той же версии что была:

```bash
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet=1.28.0-00 kubeadm=1.28.0-00 kubectl=1.28.0-00
sudo apt-mark hold kubelet kubeadm kubectl
```

**Этап 2: Восстановление Control Plane с etcd**

Восстанови сертификаты первым делом:

```bash
sudo tar -xzf /backup/pki-backup-*.tar.gz -C /
```

Инициализируй кластер без создания новых данных:

```bash
sudo kubeadm init phase preflight
sudo kubeadm init phase kubelet-start
```

Останови etcd если он запустился:

```bash
sudo systemctl stop etcd
```

Восстанови снимок etcd:

```bash
sudo ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot-*.db \
  --data-dir=/var/lib/etcd \
  --name=$(hostname) \
  --initial-cluster=$(hostname)=https://$(hostname -i):2380 \
  --initial-advertise-peer-urls=https://$(hostname -i):2380
```

Запусти static pod etcd:

```bash
sudo kubeadm init phase etcd local \
  --config=/etc/kubernetes/kubeadm-config.yaml
```

Запусти остальные control plane компоненты:

```bash
sudo kubeadm init phase control-plane all
sudo kubeadm init phase upload-config all
sudo kubeadm init phase upload-certs --upload-certs
sudo kubeadm init phase mark-control-plane
sudo kubeadm init phase bootstrap-token
```

Настрой kubeconfig:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Этап 3: Присоединение worker узлов**

На worker узлах восстанови конфигурацию:

```bash
sudo tar -xzf /backup/node-config-*.tar.gz -C /
```

Получи join команду:

```bash
kubeadm token create --print-join-command
```

Выполни join на каждом worker:

```bash
sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

**Этап 4: Восстановление сетевого плагина**

Установи CNI (например Calico):

```bash
kubectl apply -f /backup/manifests/kube-system/calico.yaml
```

Или для Cilium:

```bash
kubectl apply -f /backup/manifests/kube-system/cilium.yaml
```

Проверь что узлы Ready:

```bash
kubectl get nodes
```

**Этап 5: Восстановление CRD и операторов**

Сначала восстанови Custom Resource Definitions:

```bash
kubectl apply -f /backup/manifests/cluster/customresourcedefinitions.yaml
```

Жди пока CRD зарегистрируются:

```bash
kubectl wait --for condition=established --timeout=60s crd --all
```

Установи операторы (cert-manager, prometheus-operator и т.д.):

```bash
kubectl apply -f /backup/manifests/cert-manager/
kubectl apply -f /backup/manifests/monitoring/
```

**Этап 6: Восстановление через Velero**

Установи Velero на новом кластере:

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket kubernetes-backups \
  --secret-file ./credentials-velero \
  --backup-location-config region=us-east-1
```

Проверь доступность бекапов:

```bash
velero backup get
```

Восстанови все:

```bash
velero restore create --from-backup full-cluster-backup
```

Следи за прогрессом:

```bash
velero restore describe <restore-name>
velero restore logs <restore-name>
```

**Этап 7: Восстановление Persistent Volumes**

Если использовались CSI snapshots:

```bash
kubectl apply -f /backup/manifests/volumesnapshots/
```

PVC автоматически подключатся к восстановленным volumes.

Для volumes без снапшотов восстанови данные вручную:

```bash
kubectl exec -it <pod-name> -- bash
# Внутри пода перемести данные из резервной копии
```

**Этап 8: Восстановление Helm releases**

Переустанови charts с сохраненными values:

```bash
cat /backup/helm/releases.json | jq -r '.[] | "\(.name) \(.namespace) \(.chart)"' | while read name ns chart; do
  helm install $name $chart \
    --namespace $ns \
    --create-namespace \
    --values /backup/helm/${name}-values.yaml
done
```

**Этап 9: Верификация**

Проверь что все поды работают:

```bash
kubectl get pods --all-namespaces
```

Проверь services и endpoints:

```bash
kubectl get svc,ep --all-namespaces
```

Проверь ingress маршруты:

```bash
kubectl get ingress --all-namespaces
```

Проверь работу приложений:

```bash
kubectl run test-pod --image=busybox --rm -it -- wget -O- http://your-service
```

### Автоматизация процесса

Создай cron-скрипт для ежедневных бекапов:

```bash
cat > /usr/local/bin/k8s-backup.sh <<'EOF'
#!/bin/bash
set -e

BACKUP_DIR="/backup/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

echo "Starting etcd backup..."
ETCDCTL_API=3 etcdctl snapshot save $BACKUP_DIR/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

echo "Backing up PKI..."
tar -czf $BACKUP_DIR/pki-backup.tar.gz /etc/kubernetes/pki/ /etc/kubernetes/*.conf

echo "Exporting manifests..."
mkdir -p $BACKUP_DIR/manifests
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  mkdir -p $BACKUP_DIR/manifests/$ns
  for resource in $(kubectl api-resources --verbs=list --namespaced -o name); do
    kubectl get $resource -n $ns -o yaml > $BACKUP_DIR/manifests/$ns/$resource.yaml 2>/dev/null || true
  done
done

echo "Triggering Velero backup..."
velero backup create auto-backup-$(date +%Y%m%d-%H%M%S) \
  --include-namespaces '*' \
  --snapshot-volumes=true

echo "Uploading to S3..."
aws s3 sync $BACKUP_DIR s3://your-backup-bucket/kubernetes/$(date +%Y%m%d)/

echo "Cleaning up old backups (>30 days)..."
find /backup/ -type d -mtime +30 -exec rm -rf {} +

echo "Backup completed successfully!"
EOF

chmod +x /usr/local/bin/k8s-backup.sh

echo "0 2 * * * /usr/local/bin/k8s-backup.sh >> /var/log/k8s-backup.log 2>&1" | sudo crontab -
```

### Критичные моменты

**Версионность:** Kubernetes очень чувствителен к версиям. Восстанавливай на точно такую же версию kubeadm/kubelet/kubectl.

**Сертификаты:** Если сертификаты истекли, перед восстановлением обнови их через kubeadm certs renew all.

**Storage Classes:** Убедись что storage provisioners доступны перед восстановлением PVC.

**Secrets:** etcd snapshot содержит все secrets, включая токены и пароли. Храни бекапы зашифрованными.

**Network policies:** После восстановления сети могут работать некорректно если IP адреса изменились. Проверь pod CIDR и service CIDR.

**LoadBalancers:** Внешние балансировщики нужно перенастроить на новые IP адресов узлов.

Этот план покрывает disaster recovery сценарий когда нужно восстановить кластер с нуля. Тестируй процесс восстановления регулярно на отдельном окружении.