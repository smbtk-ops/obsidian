# Инструкция по установке и настройке Kubernetes-кластера на Ubuntu

> Все шаги выполняются **на всех серверах**, если не указано иное.  
> Рекомендуется использовать одинаковые версии Ubuntu и ядра Linux.

---

## 1. Отключение swap

```bash
sudo sed -i '/swap/s/^/#/' /etc/fstab
sudo swapoff -a
```

Проверьте в `/etc/fstab`, что swap-разделы закомментированы.

---

## 2. Подключение модулей ядра

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
vxlan
EOF
```

---

## 3. Полезная документация по модулям ядра

- `man modprobe.d` — конфигурация modprobe
- `man systemd-modules-load` — автозагрузка модулей
- **OverlayFS** — [документация](https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html)
- **Bridge Netfilter** — [документация](https://www.kernel.org/doc/Documentation/networking/bridge.txt)
- **IPVS** — [IP Virtual Server HOWTO](http://www.linuxvirtualserver.org/software/ipvs.html)
- **nf_conntrack** — [Netfilter Conntrack](https://wiki.nftables.org/wiki-nftables/index.php/Conntrack)
- **VXLAN** — [VXLAN documentation](https://datatracker.ietf.org/doc/html/rfc7348)

---

## 4. Установка containerd

```bash
sudo apt update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y containerd.io
```

---

## 5. Настройка containerd

```bash
containerd config default | sudo tee /etc/containerd/config.toml
```

В конфиге (`/etc/containerd/config.toml`) установить:

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

Применить изменения:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## 6. Настройка sysctl

```bash
cat << EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-arptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF

sudo sysctl --system
```

---

## 7. Установка nerdctl

```bash
wget https://github.com/containerd/nerdctl/releases/download/v1.7.6/nerdctl-full-1.7.6-linux-amd64.tar.gz
sudo tar Cxzvvf /usr/local nerdctl-full-1.7.6-linux-amd64.tar.gz
```

---

## 8. Установка kubelet, kubeadm, kubectl (Версию заменить на актуальную)

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

---

## 9. Инициализация кластера (Master Node)(Версию заменить на актуальную)

Файл `kubeadm.yaml`:

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.30.0
networking:
  serviceSubnet: "10.96.0.0/16"
  podSubnet: "192.168.0.0/16"
  dnsDomain: "cluster.local"

---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
```

Запуск инициализации:

```bash
sudo kubeadm init --config kubeadm.yaml
```

Настройка kubectl:

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 10. Переключение kube-proxy на IPVS

```bash
kubectl edit configmap -n kube-system kube-proxy
```

Изменить на:

```yaml
mode: "ipvs"
ipvs:
  strictARP: true
  scheduler: "wrr" # варианты: rr, wrr, sh
```

---

## 11. Установка Calico

```bash
sudo ufw disable
sudo mkdir -p /etc/NetworkManager/conf.d/
sudo tee /etc/NetworkManager/conf.d/calico.conf <<EOF
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico;interface-name:vxlan-v6.calico;interface-name:wireguard.cali;interface-name:wg-v6.cali
EOF
sudo systemctl restart systemd-networkd
```

Установка манифестов (Версию заменить на актуальную):

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
```

Редактируем `custom-resources.yaml`:

```yaml
spec:
  calicoNetwork:
    ipPools:
      - blockSize: 26
        cidr: 192.168.0.0/16 # должен совпадать с podSubnet в kubeadm.yaml
```

Применяем:

```bash
kubectl apply -f custom-resources.yaml
```

---

## 12. Добавление Worker Node

```bash
kubeadm join 192.168.100.6:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---

## 13. Добавление Master Node

```bash
kubeadm token create --print-join-command
kubeadm init phase upload-certs --upload-certs
```

Выполнить join с ключом:

```bash
kubeadm join 192.168.0.10:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <cert-key>
```

### Где взять ключи для `kubeadm join` (control plane и worker)

### 1. **Token (`--token <token>`)**

- **Что это:** Временный токен для присоединения ноды к кластеру.
- **Где взять:**
- На мастер-ноде, которая уже в кластере:
```bash
kubeadm token list
```
- Если токен истёк (по умолчанию — 24 часа), создаём новый:
```
kubeadm token create
```

Результат будет в формате:

```   
abcdef.0123456789abcdef
```


---

### 2. **CA hash (`--discovery-token-ca-cert-hash sha256:<hash>`)**

- **Что это:** Хэш публичного ключа корневого сертификата CA кластера.
- **Где взять:**
- На мастер-ноде:
```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt \
openssl rsa -pubin -outform der 2>/dev/null \
openssl dgst -sha256 -hex | sed 's/^.* //'
```

- Вы получите строку типа:

```
3a3f12c8e9a1b6c82e7afc6b6e19e5a07c5dfbcb676f4b8c7a283d4144d8b3f2
```

- В команду `join` подставляете так:

```
--discovery-token-ca-cert-hash sha256:3a3f12c8e9a1b6c82e7afc6b6e19e5a07c5dfbcb676f4b8c7a283d4144d8b3f2
```


---

### 3. **Certificate key (`--certificate-key <cert-key>`)**

- **Что это:** Ключ для безопасной передачи сертификатов между control plane нодами.
    
- **Где взять:**
    
    - Генерируется при выполнении:
        
        ```bash
        kubeadm init phase upload-certs --upload-certs
        ```
        
    - Результат будет:
        
        ```
        W+1j3F1W2d6+XWgqHf2yC0+Q0T8nE1ZfS8L5fM2YxFQ=
        ```
        
    - Этот ключ действителен ограниченное время (по умолчанию — 2 часа).
        
    - Используется **только** при добавлении новых master/control plane нод.
        

---

## Куда класть эти ключи?

- **Никуда сохранять в систему не нужно** — они просто вставляются в команду `kubeadm join` на **новой ноде**.
    
- Пример для **worker**:
    
    ```bash
    kubeadm join 192.168.0.10:6443 \
      --token abcdef.0123456789abcdef \
      --discovery-token-ca-cert-hash sha256:3a3f12c8e9a1b6c82e7afc6b6e19e5a07c5dfbcb676f4b8c7a283d4144d8b3f2
    ```
    
- Пример для **control plane**:
    
    ```bash
    kubeadm join 192.168.0.10:6443 \
      --token abcdef.0123456789abcdef \
      --discovery-token-ca-cert-hash sha256:3a3f12c8e9a1b6c82e7afc6b6e19e5a07c5dfbcb676f4b8c7a283d4144d8b3f2 \
      --control-plane --certificate-key W+1j3F1W2d6+XWgqHf2yC0+Q0T8nE1ZfS8L5fM2YxFQ=
    ```
    

---

Если хотите, я могу прямо сейчас сделать **готовый генератор этих ключей с командами для вашей вики**, чтобы вы всегда могли за 2–3 команды получить токен, хэш и cert-key и сразу вставить в `join`. Хотите, сделаю?
