Vertical Pod Autoscaler помогает:

- анализировать реальное потребление ресурсов;
- оптимизировать requests/limits;
- повышать утилизацию нод и снижать издержки.
#### 1. Установка VPA
```bash
git clone https://github.com/kubernetes/autoscaler/
cd autoscaler
git checkout vpa-release-1.0
cd vertical-pod-autoscaler
REGISTRY=registry.k8s.io/autoscaling TAG=1.0.0 ./hack/vpa-process-yamls.sh apply
```

Создаются все необходимые CRD, сервисы и контроллеры (`vpa-updater`, `vpa-recommender`, `vpa-admission-controller`).

---

#### 2. Создание тестового Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alpine
spec:
  replicas: 2
  selector:
    matchLabels:
      app: alpine
  template:
    metadata:
      labels:
        app: alpine
    spec:
      containers:
        - name: alpine
          image: alpine
          command: ["ping"]
          args: ["8.8.8.8"]
          resources:
            requests:
              cpu: 50m
              memory: 50Mi
            limits:
              cpu: 50m
              memory: 50Mi
```

Простой pod с минимальными ресурсами.

---
#### 3. Создание VerticalPodAutoscaler
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: alpine-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: alpine
  updatePolicy:
    updateMode: "Off"
```

Режим `Off` — только сбор рекомендаций, без изменений подов.

---
#### 4. Проверка рекомендаций
```bash
kubectl get vpa alpine-vpa -o yaml
```

В секции `status.recommendation` отображаются новые значения CPU и памяти.

---
#### 5. Включение автоматического пересоздания подов
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: alpine-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: alpine
  updatePolicy:
    updateMode: "Recreate"
```

---
#### 6. Тест нагрузки
```bash
kubectl exec -it <имя-pod> -- dd if=/dev/urandom of=/dev/zero
kubectl -n kube-system logs -f deploy/vpa-updater
```

VPA пересоздаст pod с новыми лимитами на ресурсы.

---
#### 7. Важные замечания

- Для работы нужен установленный **metrics-server**.
- На **production** рекомендуется режим `Off` — чтобы не было рестартов подов в рабочее время.
- Используйте релиз VPA, совместимый с версией API Kubernetes.
- Если при установке возникнут ошибки генерации сертификатов — возьмите скрипт из другой версии VPA.

---
