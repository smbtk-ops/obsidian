
## 1. Введение

**Cert-Manager** — это Kubernetes-контроллер, автоматизирующий получение, установку и обновление TLS-сертификатов (например, от Let's Encrypt).  
Он интегрируется с **Ingress-контроллером** (в частности, NGINX Ingress) и создает Kubernetes-секреты с приватным ключом и сертификатом.

---
## 2. Архитектура Cert-Manager

После установки Cert-Manager добавляет в кластер набор **CRD (Custom Resource Definitions):**

|Ресурс|Назначение|
|---|---|
|**Issuer**|Источник сертификатов (например, Let's Encrypt), применимый к конкретному namespace.|
|**ClusterIssuer**|То же самое, но действует на уровне всего кластера.|
|**CertificateRequest**|Запрос на получение сертификата.|
|**Order**|Временный объект, описывающий конкретный процесс запроса сертификата.|
|**Challenge**|Объект, создаваемый для валидации владения доменом (HTTP-01 или DNS-01).|
|**Certificate**|Итоговый объект, описывающий параметры выданного сертификата и хранящий ссылку на Secret.|

Cert-Manager хранит выданные сертификаты в **Secret**-объектах вида:
```yaml
data:
  tls.crt: <сертификат>
  tls.key: <приватный ключ>
type: kubernetes.io/tls
```

---

## 3. Методы валидации (Challenge Types)

Cert-Manager поддерживает два основных способа подтверждения владения доменом:
- **HTTP-01** — временно модифицирует Ingress, добавляя путь `/.well-known/acme-challenge/`.
- **DNS-01** — создаёт временную TXT-запись у DNS-провайдера (требуется поддерживаемый API).

---

## 4. Установка Cert-Manager через Helm

### 4.1. Добавление репозитория и обновление индексов
```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update
helm search repo jetstack
```

Ожидаемый результат:

```
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
jetstack/cert-manager                   v1.14.5         v1.14.5         A Helm chart for cert-manager
...
```

### 4.2. Установка Cert-Manager

```bash
helm upgrade --install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.14.5 \
  --set installCRDs=true \
  --set prometheus.enabled=false \
  --set webhook.timeoutSeconds=4
```

> Примечание: начиная с версии `1.15.x`, параметр `installCRDs=true` заменён на `--set crds.enabled=true`.

### 4.3. Проверка установки

```bash
kubectl -n cert-manager get all
```

Должны быть запущены поды:
```
pod/cert-manager
pod/cert-manager-cainjector
pod/cert-manager-webhook
```

---

## 5. Создание ClusterIssuer для Let's Encrypt

Создаём глобальный "выдавальщик" сертификатов — **ClusterIssuer**:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    email: devops@mail.com
    privateKeySecretRef:
      name: letsencrypt-private-key
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
    - http01:
        ingress:
          class: nginx
```

Применяем:

```bash
kubectl apply -f clusterissuer-letsencrypt.yaml
```

---

## 6. Настройка DNS-имени (пример с Cloudflare API)

Создаём A-запись для домена:

```bash
curl -X POST "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/dns_records" \
  -H "Authorization: Bearer <API_TOKEN>" \
  -H "Content-Type: application/json" \
  --data '{
    "type":"A",
    "name":"ingress.name",
    "content":"10.10.10.102",
    "proxied":false
  }'
```

- `name` — имя поддомена без основного домена.
- `content` — внешний IP адрес вашего **Ingress LoadBalancer** (`kubectl -n ingress-nginx get svc`).

---

## 7. Создание Ingress с автоматическим TLS

Пример манифеста:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: back
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt"
    acme.cert-manager.io/http01-ingress-class: "nginx"
spec:
  ingressClassName: nginx
  rules:
  - host: ingress.name.example.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: nginx-back
            port:
              number: 80
  tls:
  - hosts:
    - ingress.name.example.com
    secretName: tls-cert
```

После применения Cert-Manager создаст **HTTP-01 challenge** и выпишет сертификат.

---

## 8. Проверка результата

### Проверка наличия секрета:

```bash
kubectl get secrets
```

Должен появиться секрет:
```
NAME       TYPE                DATA   AGE
tls-cert   kubernetes.io/tls   2      5m
```

### Просмотр содержимого секрета:

```bash
kubectl get secret tls-cert -o yaml
```

Если в секрете присутствуют оба поля `tls.crt` и `tls.key`, сертификат успешно выдан.  
Если только `tls.key`, то проверка домена ещё не завершена.

---

## 9. Отладка и диагностика

### Проверка DNS-резолвинга

```bash
nslookup ingress.name.example.com
```

Если результат `NXDOMAIN` — запись ещё не обновилась у DNS-провайдера. Подождите 10–30 минут.

### Проверка Ingress и challenge

```bash
kubectl get ingresses
```

Если виден дополнительный Ingress `cm-acme-http-solver-xxxx`, значит Cert-Manager проходит проверку.

### Проверка логов

```bash
kubectl logs -n cert-manager deploy/cert-manager
```

Типичные ошибки:

- `no such host` — DNS не обновился.
- `connection refused` — ingress недоступен извне.
- `propagation check failed` — проверка владения доменом не завершена.

---

## 10. Обновление сертификатов

Let's Encrypt выдаёт сертификаты сроком на **90 дней**.  
Cert-Manager автоматически обновляет их за **30 дней** до истечения.

---

## 11. Типичные проблемы и решения

|Проблема|Причина|Решение|
|---|---|---|
|`NXDOMAIN`|DNS-запись не обновилась|Подождать, проверить TTL|
|`cm-acme-http-solver` не удаляется|Challenge завис|Удалить Ingress и Secret, пересоздать|
|Сертификат не создаётся|Неверный ingressClass или ClusterIssuer|Проверить аннотации и имя Issuer|
|`tls.key` без `tls.crt`|Проверка не завершена|Проверить логи и DNS-доступность|

---
