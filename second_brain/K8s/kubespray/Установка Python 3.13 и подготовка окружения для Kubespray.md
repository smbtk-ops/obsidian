## 1. Обновление системы и базовых пакетов

```bash
sudo apt update -y
sudo apt install -y git
```
## 2. Установка Python 3.13

Добавляем PPA и ставим Python 3.13:
```bash
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update
sudo apt install -y python3.13 python3.13-venv python3.13-dev
```

Проверяем:
```bash
python3.13 --version
```

## 3. Настройка Python через `update-alternatives`

Чтобы Python 3.13 стал основным интерпретатором по умолчанию:
```bash
sudo update-alternatives --install /usr/bin/python python /usr/bin/python3.13 1
sudo update-alternatives --config python
```

➡ Здесь можно выбрать нужную версию Python, если в системе установлено несколько.

Проверка:
```bash
python --version
```

Ожидаемый результат: `Python 3.13.x`

## 4. Подготовка каталога Kubespray

Клонируем репозиторий:
```bash
cd ~
git clone git@github.com:kubernetes-sigs/kubespray.git
cd kubespray
```

## 5. Создание виртуального окружения (venv)

```bash
python -m venv venv
source venv/bin/activate
```

Обновляем `pip` и утилиты:
```bash
pip install --upgrade pip setuptools wheel
```

## 6. Установка зависимостей Kubespray

```bash
pip install -r requirements.txt
```

## 7. Проверка

```bash
ansible --version
```

Если видите версию Ansible и путь в ваш `venv`, значит всё работает.
