## 1. Установка Cert-Manager
*(Если уже установлен — пропустите)*

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
kubectl create namespace cert-manager
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.14.5 \
  --set installCRDs=true
````

---
## 2. Вариант 1 — Полностью самоподписанный сертификат (SelfSigned)

Если нужен сертификат **без цепочки CA**, например, для тестовых сервисов:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
```

**Сертификат:**

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-cert
  namespace: default
spec:
  secretName: internal-tls
  issuerRef:
    name: selfsigned-cluster-issuer
    kind: ClusterIssuer
  commonName: internal.local
  dnsNames:
    - internal.local
    - svc.cluster.local
```

> **Минус:** Браузеры будут ругаться, если сертификат не доверен.
---
## 3. Вариант 2 — Собственный локальный CA
_(все сертификаты будут доверены внутри сети)_
### 3.1. Создаём Root CA

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-root-ca
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-root-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: "Internal Root CA"
  secretName: internal-root-ca
  issuerRef:
    name: selfsigned-root-ca
    kind: ClusterIssuer
```

### 3.2. Создаём ClusterIssuer на основе CA

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca-issuer
spec:
  ca:
    secretName: internal-root-ca
```

### 3.3. Выпуск сертификата от CA

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-internal-app
  namespace: default
spec:
  secretName: my-internal-app-tls
  issuerRef:
    name: internal-ca-issuer
    kind: ClusterIssuer
  commonName: "myapp.internal"
  dnsNames:
    - myapp.internal
    - myapp.svc.cluster.local
```

---
## 4. Подключение сертификата к Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "internal-ca-issuer"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.internal
      secretName: my-internal-app-tls
  rules:
    - host: myapp.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
```

---

## 5. Установка доверия к сертификату

Получаем CA:

```bash
kubectl get secret internal-root-ca -n cert-manager -o jsonpath='{.data.ca\.crt}' | base64 -d > internal-ca.crt
```

Добавляем его в доверенные сертификаты:

- **Linux:** `/usr/local/share/ca-certificates/` → `update-ca-certificates`
- **Windows:** Панель управления → Доверенные корневые центры
- **MacOS:** Keychain Access → System → Import

---

Такой подход идеален, если у балансировщика только внутренний IP, но есть нормальный домен, проброшенный через маршрутизатор — в этом случае внутри сети всё будет работать по доверенным сертификатам.