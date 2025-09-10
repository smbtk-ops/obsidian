## 1. Добавление репозитория Helm

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

---
## 2. Установка Ingress Controller

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

**Пояснения:**
- `upgrade --install` — повторный запуск обновит релиз, если он уже существует.
- `--namespace ingress-nginx` — ресурсы будут размещены в выделенном пространстве имён.
- `--create-namespace` — создаёт namespace, если он ещё не существует.

---
## 3. Проверка установки

```bash
kubectl get pods -n ingress-nginx
```

Ожидаемый вывод:

```
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-7d8f7c5b9f-wp4n6   1/1     Running   0          2m
```

---
## 4. Получение внешнего IP

```bash
kubectl get svc -n ingress-nginx
```

> ⚠ Если внешний IP не назначен (bare-metal кластер), используйте **[[MetalLB в L2-режиме в Kubernetes|MetalLB]]** или аналогичный балансировщик.

---

## 6. Тестирование работы Ingress

### 6.1 Создание тестового приложения

```bash
kubectl create deployment web --image=nginx --port=80
kubectl expose deployment web --port=80 --target-port=80
```

### 6.2 Создание тестового Ingress

Сохраняем файл `web-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: example.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

Применяем:

```bash
kubectl apply -f web-ingress.yaml
```

---

## 7. Доступ к приложению

### 7.1 Добавляем запись в `/etc/hosts`

```bash
<EXTERNAL_IP> example.local
```

### 7.2 Проверяем в браузере

```
http://example.local
```
