
## 1. Что такое Helm

**Helm** — это пакетный менеджер для Kubernetes, который:
- собирает все манифесты приложения в единый пакет (чарт);
- умеет устанавливать, обновлять и откатывать релизы;
- позволяет параметризовать манифесты с помощью шаблонов и файла `values.yaml`.

Helm работает аналогично `apt` или `yum`, но для Kubernetes.  
Каждый «пакет» — это **чарт**, содержащий шаблоны Deployment, Service, Ingress и пр.

---
## 2. Установка Helm

`https://helm.sh/docs/intro/install/#from-apt-debianubuntu`
### Вариант 1 — из системного репозитория
```bash
sudo apt-get install curl gpg apt-transport-https --yes
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
helm version
```

Пример вывода:
```
version.BuildInfo{Version:"v3.14.4", GitCommit:"81c902a123462fd4052bc5e9aa9c513c4c8fc142"}
```

### Вариант 2 — скачивание последнего бинарника (рекомендуется)
```bash
curl -sLO https://get.helm.sh/helm-v3.19.0-linux-amd64.tar.gz
tar -zxvf helm-v3.19.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
chmod +x /usr/local/bin/helm
helm version
```

### Включение автодополнения
```bash
mkdir -p /etc/bash_completion.d
helm completion bash > /etc/bash_completion.d/helm
source <(helm completion bash)
```

---

## 3. Основные команды Helm

|Команда|Назначение|
|---|---|
|`helm create chart_name`|Создаёт структуру нового чарта|
|`helm install`|Устанавливает чарт в кластер|
|`helm upgrade --install`|Обновляет или устанавливает чарт|
|`helm template`|Генерирует YAML из шаблонов|
|`helm repo add`|Добавляет репозиторий чартов|
|`helm pull / push`|Скачивает / загружает чарт|
|`helm list`|Список установленных релизов|
|`helm rollback`|Откатывает релиз на предыдущую версию|
|`helm uninstall`|Удаляет релиз|
|`helm history`|Просмотр истории релизов|

---

## 4. Создание собственного чарта

### 4.1. Создание структуры

```bash
helm create my-service
cd my-service
tree .
```

Вывод:

```
my-service/
├── charts/
├── Chart.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── _helpers.tpl
│   ├── NOTES.txt
│   └── tests/test-connection.yaml
└── values.yaml
```

---

## 5. Работа с `values.yaml`

Файл `values.yaml` содержит переменные, которые будут подставлены в шаблоны.  
Пример (сокращённый):

```yaml
replicaCount: 1

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: ""

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: ""
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
```

Использование в шаблоне (`templates/service.yaml`):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.service.name | default .Chart.Name }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
  selector:
    app.kubernetes.io/name: {{ .Chart.Name }}
```

---

## 6. Рендеринг шаблонов

Перед установкой всегда можно посмотреть, какие YAML-файлы Helm создаст:
```bash
helm template -f values.yaml my-service .
```

или сразу применить:
```bash
helm template -f values.yaml my-service . | kubectl apply -f -
```

---

## 7. Установка чарта в кластер

```bash
helm upgrade --install \
  -n test-helm \
  --create-namespace \
  -f values.yaml \
  my-release .
```

Пример результата:
```
Release "my-release" has been upgraded. Happy Helming!
NAME: my-release
NAMESPACE: test-helm
STATUS: deployed
```

Проверка:
```bash
kubectl -n test-helm get all
```

---

## 8. Изменение чарта и обновление релиза

Допустим, добавим имя сервиса в `values.yaml`:
```yaml
service:
  name: my-service
  type: ClusterIP
  port: 80
```

Изменим шаблон `templates/service.yaml`:
```yaml
metadata:
  name: {{ .Values.service.name }}
```

и в `templates/ingress.yaml` заменим строку:
```gotemplate
{{- $fullName := include "my-service.fullname" . -}}
```

на:
```gotemplate
{{- $fullName := .Values.service.name -}}
```

и включим Ingress:
```yaml
ingress:
  enabled: true
  className: "nginx"
```

Перезапустим релиз:
```bash
helm upgrade --install -n test-helm -f values.yaml my-release .
```

Проверим результат:
```bash
kubectl -n test-helm get svc,ingress --show-labels
```

---

## 9. Полезные приёмы

- Посмотреть значения у установленного релиза:
  ```bash
  helm get values my-release -n test-helm
   ```
- Проверить изменения перед обновлением:
  ```bash
  helm diff upgrade my-release . -f values.yaml
   ```
- Удалить релиз:
  ```bash
  helm uninstall my-release -n test-helm
   ```
- Откатить:
  ```bash
  helm rollback my-release 1
   ```

---

## 10. Дополнительные возможности

- **helm-secrets** — шифрование значений в `values.yaml`
- **helmfile / helmwave** — инструменты для оркестрации множества чартов
- **helm lint** — проверка синтаксиса шаблонов
- **helm package** — сборка чарта в `.tgz`

---