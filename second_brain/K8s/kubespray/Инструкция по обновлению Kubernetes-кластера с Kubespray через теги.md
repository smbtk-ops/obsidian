## 1. Проверить текущую версию

Сначала узнай, какая версия Kubernetes сейчас используется в кластере:
```bash
kubectl version
```

Также можно проверить установленный Kubespray:
```bash
cd ~/kubespray
git describe --tags
```

---

## 2. Перейти на нужный релиз Kubespray

Kubespray выпускает версии строго привязанные к поддерживаемым версиям Kubernetes. Поэтому обновление делают через **git-tag**.

Пример:
```bash
cd ~/kubespray
git fetch --tags
git checkout tags/v2.27.1
```
> Здесь `v2.24.1` — тег Kubespray. Нужно выбрать такой тег, который поддерживает нужную целевую версию Kubernetes (см. https://github.com/kubernetes-sigs/kubespray/blob/master/docs/operations/upgrades.md).

---

## 3. Обновить зависимости

После переключения на новый тег:
```bash
pip install -r requirements.txt
```
(лучше через venv, чтобы не ломать системный Python).

---

## 4. Задать новую версию Kubernetes

В файле `inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml` обнови параметр:
```yaml
kube_version: v1.32.8
```
или ту версию, которая поддерживается выбранным Kubespray-тегом.
либо можно передать параметром через -e в команде запуска ansible playbook

---

## 5. Запуск playbook для обновления

Kubespray умеет обновлять кластер **пошагово**. Запускается так:
```bash
sudo env "PATH=$PATH" ansible-playbook -i inventory/k8s/inventory.yaml upgrade-cluster.yml -u root -b \
  --limit=master1,master2,master3
```
- `--limit` можно указать для пошагового обновления (сначала мастер, потом остальные).
- если хочешь обновить всё сразу — можно без `--limit`.

После мастеров — аналогично обновляешь ноды:
```bash
sudo env "PATH=$PATH" ansible-playbook -i inventory/k8s/inventory.yaml upgrade-cluster.yml -u root -b \
  --limit=worker1,worker2
```
---

## 6. Проверка после обновления

- Проверить версию API-сервера:
```bash
kubectl get nodes -o wide
kubectl version
```
- Убедиться, что все поды в `kube-system` в состоянии `Running`:
```bash
kubectl get pods -n kube-system
```

---

## 7. Важные замечания

1. **Только теги**: никогда не обновляй с `master`  веток в продакшене — могут быть несовместимости.
2. **Совместимость**: один тег Kubespray поддерживает фиксированный набор версий Kubernetes (обычно 3 минорных релиза подряд).
3. **Резервная копия**: перед обновлением обязательно сделай бэкап etcd:
```bash
kubectl -n kube-system exec -it etcd-master1 -- etcdctl snapshot save /var/lib/etcd/backup.db
```
---
