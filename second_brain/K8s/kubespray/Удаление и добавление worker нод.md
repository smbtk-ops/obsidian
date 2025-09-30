## Удаление worker-ноды

### 1. Перевод ноды в состояние cordon/drain

**Если нода доступна:**
```bash
kubectl cordon worker1
kubectl drain worker1 --ignore-daemonsets
```

> Примечание: если узел недоступен, ключ `--ignore-daemonsets` обязателен.

---
### 2. Удаление ноды из кластера

**Если нода доступна:**
```bash
sudo env "PATH=$PATH" ansible-playbook -i inventory/k8s/inventory.yaml \
  -e node=worker1 remove-node.yml -u root -b
```

**Если нода недоступна:**
```bash
kubectl delete node worker1
```

Уберите ноду из `inventory/k8s/inventory.yaml`:

```
sudo env "PATH=$PATH" ansible-playbook -i inventory/k8s/inventory.yaml scale.yml -u root -b
```

**надо учитывать тот факт что если нода снова оживет то она автоматически переподключится обратно**

---
### 3. Чистка инвентаря

После удаления вручную отредактировать файл `inventory/k8s/inventory.yaml` и удалить `worker1`.
Проверить корректность:
```bash
ansible-inventory -i inventory/k8s/inventory.yaml --graph
```

---

## Добавление новой worker-ноды

### 1. Правка инвентаря

В файл `inventory/k8s/inventory.yaml` добавить новую ноду по аналогии с уже существующими worker-нодами.

---

### 2. Запуск плейбука для добавления ноды

Указать в `--limit` имя новой ноды (как прописано в инвентаре):
```bash
sudo env "PATH=$PATH" ansible-playbook -i inventory/k8s/inventory.yaml \
  --limit=worker1 cluster.yml
```

---

- Для **удаления**: cordon/drain → remove-node.yml (или `kubectl delete node` + upgrade-cluster.yml) → правка инвентаря.
- Для **добавления**: правка инвентаря → cluster.yml с `--limit`.



