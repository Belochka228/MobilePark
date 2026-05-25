# MobilePark — TestTask

Репозиторий содержит решение трёх задач: улучшение Dockerfile, деплой стека в Docker Swarm и автоматизация настройки серверов через Ansible.

---

## Структура репозитория

```
MobilePark/
├── docker/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── app.py
│   └── requirements.txt
├── swarm/
│   └── docker-compose.yml
└── ansible/
    ├── ansible.cfg
    ├── site.yml
    ├── inventory/
    │   ├── hosts.yml
    │   └── group_vars/
    │       └── all.yml
    └── roles/
        ├── common/
        ├── users/
        ├── docker/
        └── swarm/
```

---

## Задача 1 — Dockerfile

### Что было не так в исходном файле

**Исходный Dockerfile:**
```dockerfile
FROM python
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 5000
CMD ["python", "app.py"]
```

**Проблемы:**

1. `FROM python` — нет фиксации версии. При каждой сборке может подтянуться новая версия образа и сломать приложение. 

2. `COPY . .` перед установкой зависимостей — при каждом изменении любого файла проекта Docker будет заново устанавливать все зависимости, не используя кэш слоёв. Это замедляет сборку.

3. Запуск от пользователя `root` — Это нарушение базовых принципов безопасности.

4. Нет `.dockerignore` — в образ попадают лишние файлы: `.git`, `.venv`, `__pycache__`, что увеличивает размер образа и время сборки.

5. Нет флага `--no-cache-dir` при установке pip — pip сохраняет кэш внутри образа, увеличивая его размер.

### Исправленный Dockerfile

```dockerfile
FROM python:3.12-slim

RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN chown -R appuser:appuser /app

USER appuser

EXPOSE 5000

CMD ["python", "app.py"]
```

**Что исправлено:**

- `python:3.12-slim` — фиксированная версия + slim уменьшает образ.
- Сначала копируем `requirements.txt`, устанавливаем зависимости — потом копируем код.
- Создаём non-root пользователя `appuser` и запускаем приложение от него
- `--no-cache-dir` убирает кэш pip из образа
- `.dockerignore` исключает лишние файлы

### Как собрать и запустить

```bash
cd docker
docker build -t mobilepark-app .
docker run -p 5000:5000 mobilepark-app
```

Открыть в браузере: `http://localhost:5000`

---

## Задача 2 — Docker Swarm Compose

### Что реализовано

Стек включает три сервиса:
- `app` — ubuntu:22.04, 2 реплики, запускается на worker нодах
- `postgres` — PostgreSQL 15, запускается на manager
- `redis` — Redis 7, запускается на manager

### Требования и как они реализованы

| Требование | Реализация |
|---|---|
| Не менее 2 реплик | `deploy.replicas: 2` |
| Постоянная работа | `command: sleep infinity` |
| Rolling update | `update_config: parallelism: 1, order: start-first` |
| Логи на хост сервере | `volumes: /var/log/app:/var/log/app` |
| Лимит CPU и RAM | `resources.limits: cpus: "1", memory: 500M` |
| Лимит логов | `logging: max-size: 5m, max-file: 1` |
| Label SERVERTYPE=worker | `placement.constraints: node.labels.SERVERTYPE == worker` |
| HOSTNAME ноды | `environment: HOSTNAME={{.Node.Hostname}}` |
| Сети postgres и redis | `networks: db-postgres-net, ds-redis-net` (overlay) |

### Как задеплоить стек

**Требования:** Docker Swarm кластер с минимум одной manager и двумя worker нодами с label `SERVERTYPE=worker`.

Создать папку для логов на всех worker нодах:
```bash
sudo mkdir -p /var/log/app
```

Задеплоить стек:
```bash
docker stack deploy -c docker-compose.yml mobilepark
```

Проверить что всё запустилось:
```bash
docker service ls
```

Ожидаемый вывод:
```
mobilepark_app        replicated  2/2  ubuntu:22.04
mobilepark_postgres   replicated  1/1  postgres:15
mobilepark_redis      replicated  1/1  redis:7
```

### Подключение к контейнеру и проверка

Найти на каком worker запущен контейнер:
```bash
docker service ps mobilepark_app
```

Зайти на нужный worker и подключиться к контейнеру:
```bash
ssh user@worker-ip
docker exec -it CONTAINER_ID bash
```

Внутри контейнера установить клиенты:
```bash
apt-get update && apt-get install -y postgresql-client redis-tools
```

Проверить подключение к PostgreSQL:
```bash
psql -h postgres -U admin -d testdb
# пароль: admin123
```

Проверить подключение к Redis:
```bash
redis-cli -h redis ping
# ответ: PONG
```

---

## Задача 3 — Ansible

### Что реализовано

Четыре роли:

- **common** — установка пакетов (mc, ncdu, cifs-utils, nfs-common), отключение SSH по паролю, запрет входа под root
- **users** — создание пользователей admin-1/2/3 (sudo), ansible (sudo), autodeploy (docker), копирование SSH публичных ключей
- **docker** — установка Docker фиксированной версии из официального репозитория, добавление пользователей в группу docker
- **swarm** — инициализация Swarm на manager, подключение workers, установка label `SERVERTYPE=worker` на worker ноды

### Требования к серверам

- Ubuntu 24.04 LTS
- Минимум 3 сервера: 1 manager + 2 worker
- Пользователь с sudo доступом на всех серверах

### Настройка перед запуском

**1. Заполнить inventory** — отредактировать `ansible/inventory/hosts.yml`:
```yaml
all:
  children:
    managers:
      hosts:
        ubuntu-manager:
          ansible_host: YOUR_MANAGER_IP
          ansible_user: YOUR_USER
    workers:
      hosts:
        ubuntu-worker-1:
          ansible_host: YOUR_WORKER1_IP
          ansible_user: YOUR_USER
        ubuntu-worker-2:
          ansible_host: YOUR_WORKER2_IP
          ansible_user: YOUR_USER
```

**2. Добавить SSH ключи пользователей** — положить публичные ключи в `ansible/roles/users/files/`:
```
admin-1.pub
admin-2.pub
admin-3.pub
ansible.pub
autodeploy.pub
```

Сгенерировать можно так:
```bash
ssh-keygen -t rsa -b 4096 -f admin-1 -N "" -C "admin-1"
```

**3. Настроить sudo без пароля** для пользователя который запускает Ansible:
```bash
echo 'YOUR_USER ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/YOUR_USER
sudo chmod 440 /etc/sudoers.d/YOUR_USER
```

**4. Скопировать SSH ключ** на все серверы:
```bash
ssh-copy-id user@server-ip
```

### Запуск

```bash
cd ansible
ansible-playbook -i inventory/hosts.yml site.yml -b
```

### Проверка результата

После выполнения плейбука на manager проверить:

```bash
# Кластер Swarm
docker node ls

# Labels на workers
docker node inspect ubuntu-worker-1 --pretty | grep Labels -A2

# Пользователи
cat /etc/passwd | grep -E "admin|ansible|autodeploy"

# Пакеты
which mc && which ncdu

# SSH настройки
grep PasswordAuthentication /etc/ssh/sshd_config
grep PermitRootLogin /etc/ssh/sshd_config
```

### Версия Docker

Плейбук устанавливает Docker фиксированной версии. Версия указана в `ansible/inventory/group_vars/all.yml`:

```yaml
docker_version: "5:27.5.1-1~ubuntu.24.04~noble"
```

---

## Важные замечания

- Приватные SSH ключи пользователей не хранятся в репозитории — только публичные `.pub` файлы
- Тестировалось на Ubuntu 24.04 LTS с Docker 27.5.1