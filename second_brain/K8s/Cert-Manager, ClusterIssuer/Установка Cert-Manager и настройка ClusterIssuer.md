## 1. Установка Cert-Manager

Cert-Manager устанавливается через Helm из официальнего репозитория:

```bash
# Добавляем репозиторий
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Создаём namespace
kubectl create namespace cert-manager

# Устанавливаем CRD и Cert-Manager
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.14.5 \
  --set installCRDs=true
````

> **Важно:** `--set installCRDs=true` обязательно при первой установке, чтобы загрузить CustomResourceDefinitions — то есть зарегистрировать в кластере новые API-типы Kubernetes, с которыми работает cert-manager.  
> Без CRD кластер просто «не знает», что такое Issuer, ClusterIssuer, Certificate и т.д., и выпуск сертификатов не заработает.

---

## 2. Проверка установки

```bash
kubectl get pods -n cert-manager
```

Ожидаемый результат — три пода в состоянии **Running**:

- cert-manager
- cert-manager-cainjector
- cert-manager-webhook

---
## 3. Создание ClusterIssuer

`ClusterIssuer` — это глобальный объект, доступный во всём кластере.  
Ниже пример для **Let’s Encrypt** с **ACME-HTTP-01 challenge** через Ingress.
### 3.1. Пример ClusterIssuer для Let’s Encrypt (production)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # ACME сервер для production Let's Encrypt
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com      # ваш email для уведомлений
    privateKeySecretRef:
      name: letsencrypt-prod-key   # секрет для хранения приватного ключа
    solvers:
      - http01:
          ingress:
            class: nginx
```

**Примечание:** Для тестирования используйте staging-сервер Let's Encrypt:

```yaml
server: https://acme-staging-v02.api.letsencrypt.org/directory
```

---
## 4. Пример использования сертификата с ClusterIssuer

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-app-cert
  namespace: default
spec:
  secretName: my-app-tls          # секрет, который будет использовать Ingress
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: myapp.example.com
  dnsNames:
    - myapp.example.com
```

---
## 5. Пример Ingress с автоматическим сертификатом

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: my-app-tls
  rules:
    - host: myapp.example.com
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
## 6. Проверка выдачи сертификата

```bash
kubectl describe certificate my-app-cert -n default
kubectl get secret my-app-tls -n default
```

Если всё корректно — секрет будет содержать **tls.crt** и **tls.key**.

---
## 7. Для внутренних доменов без публичного DNS

Можно использовать **SelfSigned ClusterIssuer**:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
```

---
## 8. Дополнительно

- Для **DNS-challenge** можно настроить интеграцию с **Cloudflare**, **Route53** и другими DNS-провайдерами.
- Если у балансировщика только внутренний IP, но есть доступ извне через маршрутизатор — убедитесь, что **80/443** порты проброшены на балансировщик.
