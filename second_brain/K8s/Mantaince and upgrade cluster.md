
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
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /
```

Репозитории можно глянуть тут:
https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/#verifying-if-the-kubernetes-package-repositories-are-used

3. Проверяем доступные версии:

```bash
sudo apt update
sudo apt-cache madison kubeadm
```

---

## 2. Команды для управления доступностью нод

|Команда|Действие|
|---|---|
|`kubectl cordon node01`|Запрещает планировщику назначать новые поды на узел.|
|`kubectl drain node01 --ignore-daemonsets`|Переносит все поды (кроме DaemonSet) на другие узлы.|
|`kubectl drain node01 --ignore-daemonsets --force`|То же, но включает удаление подов без контроллеров.|
|`kubectl uncordon node01`|Разрешает планировщику снова назначать поды на узел.|

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
  
  
  
