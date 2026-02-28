# Chrony — NTP синхронизация в кластере

## 1. Что это

chrony — NTP-клиент, заменяющий systemd-timesyncd на нодах кластера. Преимущества перед timesyncd:
- `makestep` — мгновенная коррекция при большом drift (timesyncd делает только медленный slew)
- Лучше работает при нестабильной связи с NTP-серверами
- Подробная диагностика через `chronyc`

## 2. Ansible playbook

`k8s-ansible/03_configure_chrony.yml`:

```yaml
#    Установка и настройка chrony на всех нодах кластера
#    Что делает плейбук:
#      - устанавливает пакет chrony
#      - отключает systemd-timesyncd (если присутствует, apt удаляет автоматически)
#      - деплоит конфиг chrony из шаблона
#      - включает и запускает сервис chrony
#
#    Примеры команд:
#      - ansible-playbook 03_configure_chrony.yml
#      - ansible-playbook 03_configure_chrony.yml -l master-02
#      - ansible-playbook 03_configure_chrony.yml --tags=config
---

- hosts: all
  remote_user: ansible
  become: yes

  tasks:
  - name: Install chrony
    apt:
      name: chrony
      state: present
      update_cache: yes
    tags:
      - install

  - name: Stop and disable systemd-timesyncd
    systemd:
      name: systemd-timesyncd
      state: stopped
      enabled: no
    ignore_errors: yes
    tags:
      - install

  - name: Deploy chrony config
    template:
      src: templates/chrony.conf.j2
      dest: /etc/chrony/chrony.conf
      owner: root
      group: root
      mode: '0644'
    notify: Force time sync
    tags:
      - config

  - name: Enable and start chrony
    systemd:
      name: chrony
      enabled: yes
      state: started
    tags:
      - install

  handlers:
    - name: Force time sync
      command: chronyc makestep
```

---

## 3. Конфиг chrony

`k8s-ansible/templates/chrony.conf.j2`:

```
# NTP servers
pool 0.by.pool.ntp.org iburst
pool 1.by.pool.ntp.org iburst
pool 2.by.pool.ntp.org iburst
pool 3.by.pool.ntp.org iburst

# Record the rate at which the system clock gains/losses time
driftfile /var/lib/chrony/chrony.drift

# Allow the system clock to be stepped in the first three updates
# if its offset is larger than 1 second
makestep 1.0 3

# Enable kernel synchronisation of the real-time clock (RTC)
rtcsync

# Log files
logdir /var/log/chrony
```

Ключевые параметры:
- `pool X.by.pool.ntp.org iburst` — белорусские NTP-серверы, `iburst` для быстрой начальной синхронизации
- `makestep 1.0 3` — если offset > 1 секунды, сделать мгновенный step (первые 3 обновления)
- `rtcsync` — синхронизировать аппаратные часы (RTC)

Конфиг деплоится в `/etc/chrony/chrony.conf` на каждой ноде.

---

## 4. Запуск

```bash
cd ~/projects/K8S/k8s-kubespray-install/k8s-ansible
source ../kubespray/venv/bin/activate
```

### 4.1. Полная установка на все ноды

```bash
ansible-playbook 03_configure_chrony.yml
```

### 4.2. На конкретную ноду

```bash
ansible-playbook 03_configure_chrony.yml -l master-02
```

### 4.3. Только обновить конфиг (без переустановки)

```bash
ansible-playbook 03_configure_chrony.yml --tags=config
```

После обновления конфига перезапустить chrony, чтобы переключился на новые серверы:

```bash
ssh root@<ip> "systemctl restart chrony"
```

---

## 5. Диагностика

### 5.1. Статус синхронизации

```bash
chronyc tracking
```

Ключевые поля:
- `System time` — текущее отклонение от NTP (должно быть < 1ms)
- `Leap status` — должен быть `Normal`
- `Reference ID` — текущий NTP сервер

### 5.2. Список источников

```bash
chronyc sources
```

Обозначения:
- `^*` — текущий основной источник
- `^+` — резервный (хороший)
- `^-` — не используется (допустимый)
- `^x` — отброшен (ненадёжный)

### 5.3. Проверка времени на всех мастерах

```bash
for h in 10.10.1.171 10.10.1.172 10.10.1.173; do
  echo -n "$h: "; ssh root@$h "date -u '+%Y-%m-%d %H:%M:%S.%N'"
done
```

### 5.4. Принудительная синхронизация

```bash
chronyc makestep
```

---

## 6. Откат на systemd-timesyncd

```bash
systemctl stop chrony
systemctl disable chrony
apt install systemd-timesyncd
systemctl enable --now systemd-timesyncd
```
