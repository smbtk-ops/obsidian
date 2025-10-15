## Зачем нужен

- **Ingress** — объект, который описывает правила маршрутизации HTTP(S) трафика на сервисы в кластере.
- **Ingress Controller** — процессор Ingress-объектов (например, `ingress-nginx`), который применяет правила и проксирует трафик к backend-сервисам.

## Предварительные условия

- Доступный Kubernetes-кластер (kubectl работает).
    
- Helm установлен.
    
- Для публикации через `LoadBalancer`:
    - В облаке — облачный LB-контроллер.
    - В on-prem — установлен **MetalLB** с пулом адресов.
- Пространство имён `ingress-nginx` будет создано автоматически командами ниже.
---

## Установка через Helm repo (быстрый старт)

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

Проверка:
```bash
kubectl -n ingress-nginx get deployments,svc
kubectl get svc -n ingress-nginx ingress-nginx-controller --watch
```

Ожидаем `EXTERNAL-IP` у `service/ingress-nginx-controller` (пример: `192.168.0.161`). В on-prem его выдаёт MetalLB, в облаках — нативный контроллер.

---
## Кастомизация через values.yaml

Выгрузите базовые значения и отредактируйте:
```bash
helm show values ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx > values.yaml
```

Ключевые параметры, которые часто меняют (фрагмент `values.yaml`):
```yaml
controller:
  kind: Deployment
  replicaCount: 2
  nodeSelector:
    kubernetes.io/os: linux

  ingressClassResource:
    default: false

  metrics:
    enabled: false
    serviceMonitor:
      enabled: false
    prometheusRule:
      enabled: false

  hostPort:
    enabled: false
    ports:
      http: 80
      https: 443

defaultBackend:
  enabled: false

tcp: {}
udp: {}
```

Применение с кастомными значениями:
```bash
helm upgrade --install ingress-nginx ingress-nginx \
  -f values.yaml \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

---

## Установка из GitHub (удобно для офлайн/корп. контура)

Вариант 1 — исходники с каталогом charts:

```bash
git clone https://github.com/kubernetes/ingress-nginx/ -b helm-chart-4.10.1
# далее установка из charts/ по стандартной команде helm install/upgrade
```

Вариант 2 — готовый архив конкретной версии чарта:
```bash
wget https://github.com/kubernetes/ingress-nginx/releases/download/helm-chart-4.10.1/ingress-nginx-4.10.1.tgz
helm install ingress-nginx ingress-nginx-4.10.1.tgz -n ingress-nginx --create-namespace
```

> Плюсы подхода: версионирование своих правок в Git, независимость от внешнего репозитория.

---
## Варианты публикации контроллера наружу

- **LoadBalancer** — предпочтительно (MetalLB или облачный LB).
- **NodePort** — слушает порты на всех нодах, затем трафик уходит на поды контроллера.
- **HostPort** — открывает порт только на нодах с подом контроллера; обычно требует внешний балансировщик (haproxy/nginx).

Рекомендация: для on-prem — MetalLB и выделенный адрес (например, `192.168.0.161`) с записью в DNS.

---

## Базовый пример публикации сервиса через Ingress

Манифесты: Deployment + Service + Ingress (namespace `default`).
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: nginx
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-svc
                port:
                  number: 80
```

Поведение: весь трафик на `EXTERNAL-IP` контроллера (например, `http://192.168.0.161/`) попадёт в `nginx-svc`.

---

## Пример маршрутизации по subpath и host

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-service-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: my.hostname.com
      http:
        paths:
          - path: /my-service
            pathType: Prefix
            backend:
              service:
                name: nginx-svc
                port:
                  number: 80
```

Доступ: `https://my.hostname.com/my-service`.

---

## Версии API и IngressClass

Актуальный API Ingress — `networking.k8s.io/v1`.  
IngressClass позволяет указать, какой контроллер обрабатывает объект:
```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx
```

Если сделать класс по умолчанию, можно не указывать `ingressClassName` в каждом Ingress.

---

## TLS для Ingress (kubernetes.io/tls)

Создайте секрет c сертификатом и ключом:
```bash
kubectl create secret tls tls-secret --cert=tls.cert --key=tls.key -n default
```

Ingress с TLS и host:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubia
  namespace: default
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myhost.example.com
      secretName: tls-secret
  rules:
    - host: myhost.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-svc
                port:
                  number: 80
```

Проверка:

- Порт 80 должен редиректить/отдавать 200 в зависимости от конфигурации.
- Порт 443 должен отдавать сертификат из `tls-secret`. Если видите «default kubernetes…», проверьте секрет и SAN в сертификате через `openssl`.

---

## Полезные аннотации: примеры

### Вставка «сырого» конфигурационного фрагмента NGINX (server-snippet)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-snippet
  annotations:
    nginx.ingress.kubernetes.io/server-snippet: |
      set $agentflag 0;
      if ($http_user_agent ~* "(Mobile)") { set $agentflag 1; }
      if ($agentflag = 1) { return 301 https://m.example.com; }
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-svc
                port:
                  number: 80
```

### Basic-Auth для Ingress

Создание секрета:
```bash
htpasswd -c auth sergei
kubectl create secret generic basic-auth --from-file=auth -n default
kubectl get secret basic-auth -n default -o yaml
```

Ingress с basic-auth:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress-auth
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: Authentication Required
spec:
  ingressClassName: nginx
  rules:
    - host: blue.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-svc
                port:
                  number: 80
```

Проверка:

```bash
curl -D - -s -o /dev/null -H 'Host: blue.example.com' http://192.168.0.161/
curl -u login:password -D - -s -o /dev/null -H 'Host: blue.example.com' http://192.168.0.161/
```

---

## Отладка и типичные проблемы

### 1) Ingress не обрабатывается контроллером

- Проверьте адрес:
```bash
   kubectl get ingress -A
```
Если колонка `ADDRESS` пустая, подождите 1–2 минуты. Если пусто и дальше — проверяем класс и конфигурацию контроллера.
- Существует ли указанный класс:
  ```bash
  kubectl get ingressclasses
   ```
    Должен быть `nginx` с `controller: k8s.io/ingress-nginx`.
- Совпадает ли класс у контроллера:
  ```bash
  kubectl -n ingress-nginx get deploy ingress-nginx-controller -o yaml | grep -A15 args
  ```
Должны быть аргументы вида:
```
   --controller-class=k8s.io/ingress-nginx
   --ingress-class=nginx
```
- Пересоздайте проблемный Ingress и смотрите логи контроллера:
  ```bash
  kubectl -n ingress-nginx logs deploy/ingress-nginx-controller
   ```

### 2) Адрес есть, но сервис не отвечает

- В Ingress проверьте имя `service.name` и `port.number`. Namespace должен совпадать с сервисом.
- В Service проверьте `selector` и соответствующие метки на подах. Проверьте `targetPort`.
- В Pod проверьте `containerPort` и что приложение действительно слушает нужный порт.
- Проверка сетевой доступности:
  ```bash
  kubectl -n default exec -it deploy/nginx -- sh -c "apt-get update && apt-get install -y curl || true; curl -I http://127.0.0.1:80 || true"
   ```

### 3) TLS не тот (виден «default kubernetes…»)

- Проверьте имя секрета и его namespace.
- Проверьте соответствие ключа и сертификата:
 ```bash
   openssl x509 -in tls.cert -noout -text | grep -A1 "Subject Alternative Name"
   openssl rsa -in tls.key -check -noout
   ```
- Убедитесь, что host в `rules.hosts` совпадает с SAN в сертификате.
    

### 4) Нет `EXTERNAL-IP` у сервиса контроллера

- On-prem: проверьте, что установлен MetalLB, пул адресов корректный и включает подсеть нод не пересекающимся способом.
- Облако: проверьте квоты и состояние LB-ресурса у провайдера.

---
## Полезные команды

```bash
helm list -n ingress-nginx
helm get all ingress-nginx -n ingress-nginx
kubectl describe ingress -A
kubectl -n ingress-nginx get pods -o wide
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller
kubectl get svc -n ingress-nginx ingress-nginx-controller -o wide
```

---

## Рекомендации по прод-эксплуатации

- Минимум **2 реплики** контроллера.
- Выделенные ноды под Ingress (taints/tolerations, nodeSelector/affinity).
- Метрики и алёрты: включить `controller.metrics.enabled: true` и интеграцию с Prometheus (`serviceMonitor`, `prometheusRule`).
- Чёткое управление классами: либо `ingressClassName` в каждом объекте, либо один дефолтный класс через `IngressClass` с аннотацией `is-default-class: "true"`.
- DNS: укажите A/AAAA записи хостов на `EXTERNAL-IP` контроллера (пример: `192.168.0.161`).

---
