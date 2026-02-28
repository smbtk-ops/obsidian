# Prod кластер — сетевые доступы

## 1. Инфраструктура кластера

| Роль | Хост | IP |
|------|------|----|
| Master 1 | master1 | 10.10.1.171 |
| Master 2 | master2 | 10.10.1.172 |
| Master 3 | master3 | 10.10.1.173 |
| Worker 1 | worker1 | 10.10.1.181 |
| Worker 2 | worker2 | 10.10.1.182 |
| Worker 3 | worker3 | 10.10.1.183 |
| API Server (VIP) | kube-vip | 10.10.1.170:6443 |
| MetalLB пул | — | 10.10.1.176 - 10.10.1.180 |

---

## 2. Сервисы с внешним доступом (LoadBalancer / Ingress)

| Сервис                    | IP          | Порт                               |
| ------------------------- | ----------- | ---------------------------------- |
| Ingress (slot-dev.mxl.by) | 10.10.1.176 | 80                                 |
| slotegrator               | 10.10.1.177 | 8090                               |
| back-mgmnt-microservice   | 10.10.1.178 | 9000                               |
| back-mgmnt-nginx          | 10.10.1.178 | 8081                               |
| logger-microservice       | 10.10.1.178 | 9090                               |
| mariadb-logger            | 10.10.1.178 | 3306                               |
| mongodb-stories           | 10.10.1.178 | 27017                              |
| stories-microservice      | 10.10.1.178 | 9091                               |
| jaeger-logger             | 10.10.1.178 | 16686, 14268, 6831/UDP, 4317, 4318 |

---

## 3. Внутренние сервисы (ClusterIP)

| Namespace | Сервис | Cluster IP | Порт(ы) |
|-----------|--------|------------|---------|
| default | kubernetes | 10.233.0.1 | 443 |
| default | slot-client | 10.233.34.148 | 80 |
| maxline | back-mgmnt-redis | 10.233.52.152 | 6379 |
| kube-system | coredns | 10.233.0.3 | 53/UDP, 53/TCP, 9153 |
| kube-system | metrics-server | 10.233.10.72 | 443 |
| datadog | datadog-agent | 10.233.14.133 | 8125/UDP, 8126 |
| datadog | datadog-agent-cluster-agent | 10.233.12.9 | 5005 |
| datadog | datadog-agent-cluster-agent-admission-controller | 10.233.54.234 | 443 |
| longhorn-system | longhorn-frontend | 10.233.34.66 | 80 |
| longhorn-system | longhorn-backend | 10.233.37.61 | 9500 |
| longhorn-system | longhorn-admission-webhook | 10.233.38.147 | 9502 |
| longhorn-system | longhorn-recovery-backend | 10.233.5.116 | 9503 |
| metallb-system | metallb-webhook-service | 10.233.23.132 | 443 |
| metallb-system | webhook-service | 10.233.30.193 | 443 |
| zabbix-monitoring | zabbix-kube-state-metrics | 10.233.39.155 | 8080 |
| zabbix-monitoring | zabbix-zabbix-helm-chart-agent | 10.233.29.64 | 10050 |
| zabbix-monitoring | zabbix-zabbix-helm-chart-proxy | 10.233.2.59 | 10051 |

---

## 4. Свободные IP из MetalLB пула

| IP | Статус |
|----|--------|
| 10.10.1.179 | свободен |
| 10.10.1.180 | свободен |
