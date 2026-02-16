# Active-Active MTProxy + HAProxy (TPROXY) + Ansible

Ansible-проект для развёртывания отказоустойчивого кластера Telegram MTProxy с балансировкой нагрузки через HAProxy в transparent-режиме (TPROXY).

---

## Тестовый стенд (развёрнуто и проверено)

Проект развёрнут на тестовом VPS и доступен для проверки:

| Ресурс | Ссылка / адрес |
|---|---|
| **Подключение через Telegram** | `tg://proxy?server=206.245.131.154&port=8443&secret=b3b34e89c9471207ad107a03b565e9a3` |
| **То же, через браузер** | [https://t.me/proxy?server=206.245.131.154&port=8443&secret=b3b34e89c9471207ad107a03b565e9a3](https://t.me/proxy?server=206.245.131.154&port=8443&secret=b3b34e89c9471207ad107a03b565e9a3) |
| **HAProxy Stats Dashboard** | [http://206.245.131.154:8404/stats](http://206.245.131.154:8404/stats) |

**Конфигурация тестового стенда:**

- **VPS**: `206.245.131.154`, Debian 11 (bullseye), 2 vCPU / 2 GB RAM
- **mtproxy_1**: порт `4431` -- UP
- **mtproxy_2**: порт `4432` -- UP
- **HAProxy**: порт `8443` (вход), `8404` (stats), алгоритм `leastconn`, TPROXY
- **Failover проверен**: `docker stop mtproxy_2` -- HAProxy определяет DOWN за ~6 сек, трафик автоматически идёт на `mtproxy_1`. После `docker start mtproxy_2` -- возврат в UP за ~4 сек.

---

## Архитектура

```
                     ┌──────────────────────────────────────────────┐
                     │              Единая ВМ (--network host)      │
                     │                                              │
 Telegram-клиент ──► │  HAProxy :8443                               │
 (реальный IP        │    ├─► mtproxy_1 :4431 ──► Telegram Servers  │
  сохраняется)       │    └─► mtproxy_2 :4432 ──► Telegram Servers  │
                     │                                              │
                     │  Stats dashboard :8404/stats                 │
                     └──────────────────────────────────────────────┘
```

- **HAProxy** принимает TCP-подключения на порту 8443 и балансирует их между MTProxy-инстансами по алгоритму `leastconn`.
- **TPROXY** (`source 0.0.0.0 usesrc clientip`) сохраняет реальный IP-адрес клиента в пакетах — MTProxy видит настоящий IP подключающегося.
- **Failover**: если один из бэкендов недоступен (health check: TCP connect каждые 3 сек, 2 неудачи = DOWN), трафик автоматически переключается на оставшийся.

## Выбор реализации Transparent Mode

| Вариант | Подходит? | Почему |
|---|---|---|
| **PROXY Protocol** | Нет | `telegrammessenger/proxy` не поддерживает PROXY Protocol |
| **DSR (Direct Server Return)** | Нет | HAProxy — полноценный TCP-прокси, ответ должен идти обратно через него |
| **TPROXY (`IP_TRANSPARENT`)** | **Да** | Спуфинг source IP на уровне ядра Linux; работает с любым TCP-приложением без модификации бэкенда |

**Выбран TPROXY** как единственный способ сохранить IP клиента для приложения, не поддерживающего PROXY Protocol. Все контейнеры работают с `--network host` для единого сетевого пространства, исключающего конфликты Docker NAT с transparent-соединениями.

## Требования

- **ВМ**: 1 vCPU / 1 GB RAM (минимум), 2 vCPU / 2 GB RAM (рекомендовано)
- **ОС**: Debian 11+ / Ubuntu 22.04+ (протестировано на Debian 11 bullseye)
- **Ansible**: >= 2.14 на управляющей машине
- **SSH-доступ** к целевому хосту с правами `sudo`

## Быстрый старт

### 1. Установить зависимости Ansible

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Настроить inventory

Отредактировать `inventory/hosts.yml` — указать IP, пользователя и SSH-ключ целевой ВМ:

```yaml
all:
  children:
    mtproxy_servers:
      hosts:
        mtproxy-host:
          ansible_host: 203.0.113.10       # IP вашей ВМ
          ansible_user: root
          ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

Или через переменные окружения:

```bash
export MTPROXY_HOST=203.0.113.10
export MTPROXY_USER=root
export MTPROXY_SSH_KEY=~/.ssh/id_rsa
```

### 3. Сгенерировать секрет MTProxy

```bash
# Генерация 32 hex символов (16 байт)
openssl rand -hex 16
```

Вставить полученный секрет в `inventory/group_vars/all.yml`:

```yaml
mtproxy_secret: "<32_hex_символа>"
```

> Образ `telegrammessenger/proxy` сам добавляет `dd`-префикс (fake-TLS) при генерации ссылок.

### 4. Запустить playbook

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

### 5. Получить ссылку для Telegram

```
tg://proxy?server=<IP_ВМ>&port=8443&secret=<ваш_секрет>
```

Или в формате HTTPS:

```
https://t.me/proxy?server=<IP_ВМ>&port=8443&secret=<ваш_секрет>
```

## Проверка Failover

> Ниже -- инструкция на примере тестового стенда (`206.245.131.154`). Для своего сервера замените IP.

### Шаг 1. Убедиться, что оба бэкенда работают

Открыть HAProxy Stats Dashboard в браузере:

```
http://206.245.131.154:8404/stats
```

Оба сервера `mtproxy_1` и `mtproxy_2` должны быть в статусе **UP** (зелёные).

Или через curl:

```bash
curl -s http://206.245.131.154:8404/stats | grep -oP 'mtproxy_\d.*?(UP|DOWN)'
```

### Шаг 2. Подключиться через Telegram

Добавить прокси по ссылке (для тестового стенда):

```
tg://proxy?server=206.245.131.154&port=8443&secret=b3b34e89c9471207ad107a03b565e9a3
```

Убедиться, что подключение работает (статус: "Connected").

### Шаг 3. Остановить один инстанс

```bash
ssh root@206.245.131.154
docker stop mtproxy_2
```

### Шаг 4. Проверить failover

1. **Stats Dashboard** (`http://206.245.131.154:8404/stats`):
   - `mtproxy_1` -- **UP** (зелёный)
   - `mtproxy_2` -- **DOWN** (красный, определяется за ~6 сек)

2. **Telegram**: подключение **продолжает работать** -- трафик автоматически идёт через `mtproxy_1`.

### Шаг 5. Вернуть инстанс в строй

```bash
docker start mtproxy_2
```

Через ~6 секунд (2 успешных health check, `inter 3s rise 2`) `mtproxy_2` вернётся в статус **UP** и начнёт принимать новые подключения.

## Масштабирование

Изменить количество реплик в `inventory/group_vars/all.yml`:

```yaml
mtproxy_replicas: 4
```

Перезапустить playbook:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

Ansible автоматически:
- Создаст новые контейнеры `mtproxy_3`, `mtproxy_4` на портах 4433, 4434.
- Обновит конфиг HAProxy (добавит новые бэкенды в шаблон).
- Выполнит graceful reload HAProxy без разрыва существующих соединений.

При уменьшении `mtproxy_replicas` лишние контейнеры будут остановлены и удалены.

## Ротация конфигов HAProxy

При изменении любых переменных в `group_vars/all.yml` (алгоритм балансировки, порты, количество реплик):

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

Порядок действий (автоматически):
1. Шаблон `haproxy.cfg.j2` рендерится с новыми значениями.
2. Если конфиг изменился — срабатывает handler `Reload HAProxy`.
3. Handler валидирует конфиг внутри контейнера (`haproxy -c -f ...`).
4. При успешной валидации — отправляет `SIGUSR2` (graceful reload в master-worker mode).
5. Новые воркеры стартуют с обновлённым конфигом, старые дослуживают соединения.

## Структура проекта

```
testovoe_zadanie/
├── ansible.cfg                    # конфигурация Ansible
├── requirements.yml               # зависимости (community.docker, ansible.posix)
├── inventory/
│   ├── hosts.yml                  # целевые хосты
│   └── group_vars/
│       └── all.yml                # все переменные (реплики, секрет, порты)
├── playbooks/
│   └── site.yml                   # главный playbook
├── roles/
│   ├── docker/
│   │   └── tasks/main.yml         # установка Docker CE + Python SDK
│   ├── mtproxy/
│   │   ├── defaults/main.yml      # переменные по умолчанию
│   │   ├── tasks/main.yml         # создание N контейнеров + scale-down
│   │   ├── templates/
│   │   │   └── run.sh.j2          # кастомный run.sh (поддержка PORT env)
│   │   └── handlers/main.yml      # рестарт контейнеров
│   └── haproxy/
│       ├── defaults/main.yml      # переменные HAProxy
│       ├── tasks/main.yml         # TPROXY-настройка, деплой, запуск контейнера
│       ├── templates/
│       │   └── haproxy.cfg.j2     # Jinja2-шаблон с динамическим backend
│       └── handlers/main.yml      # валидация + graceful reload (SIGUSR2)
└── README.md
```

## Подготовка хоста (выполняется Ansible автоматически)

Для работы TPROXY Ansible настраивает:

- **sysctl**: `net.ipv4.ip_nonlocal_bind=1` (биндинг на нелокальные IP), `net.ipv4.ip_forward=1`
- **ip rule**: `fwmark 1 lookup 100` — маршрутизация маркированных пакетов
- **ip route**: `local 0.0.0.0/0 dev lo table 100` — доставка обратных пакетов на loopback
- **iptables mangle**: маркировка обратного трафика от MTProxy-портов (fwmark 1)
- **Docker capabilities**: `NET_ADMIN`, `NET_RAW` для HAProxy-контейнера

## Переменные

| Переменная | Умолчание | Описание |
|---|---|---|
| `mtproxy_replicas` | `2` | Количество MTProxy инстансов |
| `mtproxy_secret` | `000...` | Секрет прокси (32 hex символа) |
| `mtproxy_image` | `telegrammessenger/proxy:latest` | Docker-образ MTProxy |
| `mtproxy_base_port` | `4431` | Базовый порт (инстансы: 4431, 4432, ...) |
| `haproxy_image` | `haproxy:2.9-alpine` | Docker-образ HAProxy |
| `haproxy_frontend_port` | `8443` | Порт входа для клиентов |
| `haproxy_stats_port` | `8404` | Порт HAProxy Stats Dashboard |
| `haproxy_balance` | `leastconn` | Алгоритм балансировки (leastconn / roundrobin) |
