## 1. Что это и из чего состоит

Мониторинг prod-кластера построен на community Helm-чарте Zabbix. В namespace `zabbix-monitoring` разворачиваются:

- **Zabbix Agent2** — `DaemonSet`, по одному поду на ноду. Собирает метрики хоста (`hostNetwork: true`, монтирование rootfs). Образ `zabbix/zabbix-agent2`.
- **Zabbix Proxy** — `Deployment`, active-режим, буфер на SQLite. Принимает данные агентов кластера и пересылает их на внешний Zabbix server. Образ `zabbix/zabbix-proxy-sqlite3`.
- **kube-state-metrics** — отдаёт состояние объектов Kubernetes (Deployments, Pods и т.д.) в виде метрик.

Поток данных: `agent2 (ноды) + kube-state-metrics` → `zabbix-proxy (active)` → `Zabbix server 10.10.1.249`.

| Компонент | Тип | Порт | Сервис |
|---|---|---|---|
| zabbix-agent2 | DaemonSet | 10050 | ClusterIP |
| zabbix-proxy | Deployment | 10051 | ClusterIP |
| kube-state-metrics | Deployment | 8080 | ClusterIP |

---

## 2. Предварительные условия

- Рабочий kubectl до prod-кластера (kubeconfig `prod-config.conf`).
- Установлен Helm.
- Сетевая доступность от кластера до Zabbix server `10.10.1.249:10051`.
- Namespace `zabbix-monitoring`.

---

## 3. Helm repository

```bash
helm repo add zabbix-community https://zabbix-community.github.io/helm-zabbix
helm repo update zabbix-community
helm search repo zabbix-community -l | head
```

> Релиз в prod установлен из старой линейки чарта — `zabbix-helm-chart 1.6.8` (release name `zabbix`). В текущем репозитории чарт переименован в `zabbix-community/zabbix` и переразбит на версии (актуальная ветка `7.0.x`). Для нового развёртывания используйте `zabbix-community/zabbix`; параметры values совместимы по смыслу.

---

## 4. values.yaml (фактическая конфигурация prod)

```yaml
rbac:
  create: true
serviceAccount:
  create: true

kubeStateMetrics:
  enabled: true

zabbixJavaGateway:
  enabled: false

zabbixAgent:
  enabled: true
  image:
    repository: zabbix/zabbix-agent2
    tag: alpine-7.0.27        # держим в уровень версии сервера
    pullPolicy: IfNotPresent
  hostNetwork: true
  hostPID: false
  hostRootFsMount: true
  nodeSelector:
    kubernetes.io/os: linux
  rbac:
    create: true
    pspEnabled: false
  env:
    - name: ZBX_SERVER_HOST
      value: 10.10.1.0/24,10.233.0.0/16
    - name: ZBX_PASSIVE_ALLOW
      value: "true"
    - name: ZBX_ACTIVE_ALLOW
      value: "false"
    - name: ZBX_DEBUGLEVEL
      value: "2"
    - name: ZBX_TIMEOUT
      value: "10"
  service:
    type: ClusterIP
    port: 10050
    targetPort: 10050
    portName: zabbix-agent
    listenOnAllInterfaces: true
    annotations:
      agent.zabbix/monitor: "true"
  tolerations:
    - key: node-role.kubernetes.io/control-plane
      effect: NoSchedule

zabbixProxy:
  enabled: true
  image:
    repository: zabbix/zabbix-proxy-sqlite3
    tag: alpine-7.0.27        # см. раздел 6 — правка
    pullPolicy: IfNotPresent
  env:
    - name: ZBX_PROXYMODE
      value: "0"               # active proxy
    - name: ZBX_HOSTNAME
      value: k8s-prod-proxy    # должно совпадать с именем proxy на сервере
    - name: ZBX_SERVER_HOST
      value: 10.10.1.249       # внешний Zabbix server
    - name: ZBX_DEBUGLEVEL
      value: "2"
    - name: ZBX_CACHESIZE
      value: 256M
    - name: ZBX_STARTJAVAPOLLERS
      value: "0"
    - name: ZBX_PROXYCONFIGFREQUENCY
      value: "120"
  service:
    type: ClusterIP
    port: 10051
    targetPort: 10051
  tolerations:
    - key: node-role.kubernetes.io/control-plane
      effect: NoSchedule
```

Ключевые моменты:

- `ZBX_PROXYMODE=0` — **active** proxy: сам инициирует соединение к серверу и забирает конфигурацию (`ZBX_PROXYCONFIGFREQUENCY=120` сек).
- `ZBX_HOSTNAME=k8s-prod-proxy` обязан совпадать с именем Proxy, заведённого на Zabbix server.
- Агент работает в `hostNetwork` и монтирует rootfs ноды (`hostRootFsMount: true`), поэтому видит метрики хоста.
- `tolerations` на `control-plane` — чтобы агенты и proxy ехали в т.ч. на master-ноды.

---

## 5. Установка / обновление

```bash
helm upgrade --install zabbix zabbix-community/zabbix \
  --namespace zabbix-monitoring --create-namespace \
  -f values.yaml
```

Проверка:

```bash
kubectl -n zabbix-monitoring get pods -o wide
kubectl -n zabbix-monitoring get ds,deploy,svc
```

Ожидаем: DaemonSet агента `N/N Ready` (по числу нод), Deployment `zabbix-proxy` `1/1`, kube-state-metrics `1/1`.

---

## 6. Наши правки

### 6.1. Версия образов = версии сервера

Proxy **не должен быть новее** Zabbix server, иначе сервер отвергает его данные. Сервер — `7.0.27`, поэтому образы зафиксированы на той же патч-версии той же LTS-ветки:

```bash
kubectl -n zabbix-monitoring set image deploy/zabbix-zabbix-helm-chart-proxy \
  zabbix-zabbix-helm-chart-proxy=zabbix/zabbix-proxy-sqlite3:alpine-7.0.27
```

Проверить актуальную ветку 7.0.x можно по appVersion чарта (`helm search repo zabbix-community -l`) и тегам Docker Hub.

### 6.2. Liveness-проба proxy: tcpSocket → exec

**Проблема дефолта.** Чарт ставит на proxy `livenessProbe: tcpSocket:10051`. При зависании proxy (краш воркера по SIGSEGV + дедлок остальных воркеров на общем мьютексе) главный процесс остаётся жив и **продолжает держать TCP-порт открытым на уровне ядра**, поэтому tcpSocket-проба проходит, Kubernetes под не перезапускает, а данные при этом не уходят → на сервере массовый `nodata`.

**Надёжный детектор.** Команда runtime-control требует ответа от рабочего процесса:

- здоровый proxy → `Runtime control command was forwarded successfully`, exit `0`;
- зависший proxy → `Timeout while waiting for response`, exit `1`.

Заменяем пробу на `exec`:

```bash
kubectl -n zabbix-monitoring patch deploy zabbix-zabbix-helm-chart-proxy --type strategic -p '{
  "spec":{"template":{"spec":{"containers":[{
    "name":"zabbix-zabbix-helm-chart-proxy",
    "livenessProbe":{
      "exec":{"command":["zabbix_proxy","-R","config_cache_reload"]},
      "tcpSocket":null,
      "initialDelaySeconds":90,
      "periodSeconds":60,
      "timeoutSeconds":12,
      "failureThreshold":3,
      "successThreshold":1
    }
  }]}}}}'
```

Эквивалент в values (для чарта `zabbix-community/zabbix`, где override проб поддержан):

```yaml
zabbixProxy:
  livenessProbe:
    exec:
      command: ["zabbix_proxy", "-R", "config_cache_reload"]
    initialDelaySeconds: 90
    periodSeconds: 60
    timeoutSeconds: 12
    failureThreshold: 3
    successThreshold: 1
```

Параметры подобраны так, чтобы не флапать: проба запускается через 90 с после старта, раз в минуту, перезапуск только после 3 подряд неудач (~3 мин). `config_cache_reload` раз в минуту для одного proxy — пренебрежимая нагрузка.

---

## 7. Диагностика и полезные команды

```bash
# Статус стека
kubectl -n zabbix-monitoring get pods -o wide

# Логи proxy
kubectl -n zabbix-monitoring logs deploy/zabbix-zabbix-helm-chart-proxy --tail=100

# Версия proxy и подключение к серверу (в логах при старте)
#   Starting Zabbix Proxy (active) [k8s-prod-proxy]. Zabbix 7.0.27 ...

# Ручная проверка "жив ли proxy" (то же, что делает liveness)
POD=$(kubectl -n zabbix-monitoring get pods --no-headers | awk '/proxy/&&/Running/{print $1}' | head -1)
kubectl -n zabbix-monitoring exec $POD -- zabbix_proxy -R config_cache_reload; echo "EXIT=$?"
```

**Признаки зависания proxy (дедлок):**

- в логах последняя строка — аварийный дамп `Got signal [signal:11(SIGSEGV) ...]`, после него тишина;
- внутри контейнера часть воркеров `zabbix_proxy` в состоянии `S` и не пишут логов;
- `zabbix_proxy -R config_cache_reload` отвечает `Timeout while waiting for response`;
- на сервере — триггер «More than 100 items having missing data for more than 10 minutes».

**Лечение вручную (если проба ещё не настроена):**

```bash
kubectl -n zabbix-monitoring rollout restart deploy/zabbix-zabbix-helm-chart-proxy
```

---

## 8. Важные замечания

- **Правки 6.1 и 6.2 применены на уровне Deployment через `kubectl`.** GitOps (ArgoCD/Flux) для этого релиза нет, поэтому правки переживают рестарты подов. Но при любом `helm upgrade` они будут **сброшены** к дефолтам чарта. Чтобы закрепить — перенести их в `values.yaml` (image.tag + livenessProbe) и хранить values в git.
- Чарт, которым установлен prod (`zabbix-helm-chart 1.6.8`), в текущем репозитории отсутствует — он переименован в `zabbix-community/zabbix`. Старый чарт **не поддерживает** override `livenessProbe` через values, поэтому проба правится только на уровне Deployment (либо после миграции на новый чарт).
- Agent (DaemonSet) также поднят до `alpine-7.0.27` (`kubectl set image ds/zabbix-zabbix-helm-chart-agent zabbix-agent=zabbix/zabbix-agent2:alpine-7.0.27`) — rolling-restart по одному поду (`maxUnavailable=1`). Весь стек в lockstep с сервером 7.0.27.
- При смене версии Zabbix server сначала обновляется server, затем proxy/agent до той же патч-версии (proxy не новее server).
