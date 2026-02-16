# Active-Active MTProxy + HAProxy + Ansible

Ansible-проект для развёртывания отказоустойчивого кластера Telegram MTProxy с балансировкой нагрузки через HAProxy (TCP mode, опциональный TPROXY).

---

## Тестовый стенд (развёрнуто и проверено)

Проект развёрнут на тестовом VPS и доступен для проверки:

| Ресурс | Ссылка / адрес |
|---|---|
| **Подключение через Telegram** | `tg://proxy?server=206.245.131.154&port=8443&secret=ddb3b34e89c9471207ad107a03b565e9a3` |
| **То же, через браузер** | [https://t.me/proxy?server=206.245.131.154&port=8443&secret=ddb3b34e89c9471207ad107a03b565e9a3](https://t.me/proxy?server=206.245.131.154&port=8443&secret=ddb3b34e89c9471207ad107a03b565e9a3) |
| **HAProxy Stats Dashboard** | [http://206.245.131.154:8404/stats](http://206.245.131.154:8404/stats) (логин: `admin`, пароль: `Ch4ng3M3!`) |

**Конфигурация тестового стенда:**

- **VPS**: `206.245.131.154`, Debian 11 (bullseye), 2 vCPU / 2 GB RAM
- **mtproxy_1**: порт `4431` — UP
- **mtproxy_2**: порт `4432` — UP
- **HAProxy**: порт `8443` (вход), `8404` (stats), алгоритм `leastconn`
- **Failover проверен**: `docker stop mtproxy_2` — HAProxy определяет DOWN за ~6 сек, трафик автоматически идёт на `mtproxy_1`. После `docker start mtproxy_2` — возврат в UP за ~4 сек.

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
- **Failover**: если один из бэкендов недоступен (health check: TCP connect каждые 3 сек, 2 неудачи = DOWN), трафик автоматически переключается на оставшийся.

## Transparent Mode (TPROXY) — анализ и реализация

### Рассмотренные варианты

| Вариант | Подходит? | Почему |
|---|---|---|
| **PROXY Protocol** | Нет | `telegrammessenger/proxy` не поддерживает PROXY Protocol |
| **DSR (Direct Server Return)** | Нет | HAProxy — полноценный TCP-прокси, ответ должен идти обратно через него |
| **TPROXY (`IP_TRANSPARENT`)** | **Условно** | Спуфинг source IP на уровне ядра Linux; работает с любым TCP-приложением, но **требует разные сетевые сегменты** |

### Реализация

Проект поддерживает **TPROXY** (`source 0.0.0.0 usesrc clientip`) — спуфинг source IP на уровне ядра Linux, что позволяет MTProxy видеть реальный IP клиента. Функция управляется переменной `haproxy_tproxy`.

**Ограничение single-host:** на одном хосте HAProxy и MTProxy общаются через loopback (`127.0.0.1`). Ядро Linux отбрасывает пакеты с нелокальным source IP на lo-интерфейсе — SYN с клиентским IP до MTProxy не доходит. Поэтому для single-host развёртывания TPROXY отключён (`haproxy_tproxy: false`).

**Для multi-host** (HAProxy и MTProxy на разных серверах или в разных network namespace): установить `haproxy_tproxy: true`. В этом случае пакеты идут через физический интерфейс, и TPROXY работает корректно.

Инфраструктура TPROXY (sysctl, ip rule/route, iptables mangle, capabilities) деплоится всегда и готова к включению.

Все контейнеры работают с `--network host` для единого сетевого пространства, исключающего конфликты Docker NAT.

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

Через переменные окружения (рекомендуется):

```bash
export MTPROXY_HOST=203.0.113.10
export MTPROXY_USER=root
```

Или отредактировать `inventory/hosts.yml` напрямую — указать IP и пользователя целевой ВМ.

> **Безопасность**: Не храните пароли в открытом виде в inventory. Используйте SSH-ключи или `--ask-pass` / `ansible-vault`.

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

Для подключения по паролю:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --ask-pass
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

> Ниже — инструкция на примере тестового стенда (`206.245.131.154`). Для своего сервера замените IP.

### Шаг 1. Убедиться, что оба бэкенда работают

Открыть HAProxy Stats Dashboard в браузере:

```
http://206.245.131.154:8404/stats
```

> Логин: `admin`, пароль: `Ch4ng3M3!`

Оба сервера `mtproxy_1` и `mtproxy_2` должны быть в статусе **UP** (зелёные).

Или через curl:

```bash
curl -s -u admin:Ch4ng3M3! http://206.245.131.154:8404/stats | grep -oP 'mtproxy_\d.*?(UP|DOWN)'
```

### Шаг 2. Подключиться через Telegram

Добавить прокси по ссылке (для тестового стенда):

```
tg://proxy?server=206.245.131.154&port=8443&secret=ddb3b34e89c9471207ad107a03b565e9a3
```

> Обратите внимание на `dd`-префикс в секрете — он обязателен для fake-TLS режима.

Убедиться, что подключение работает (статус: "Connected").

### Шаг 3. Остановить один инстанс

```bash
ssh root@206.245.131.154
docker stop mtproxy_2
```

### Шаг 4. Проверить failover

1. **Stats Dashboard** (`http://206.245.131.154:8404/stats`):
   - `mtproxy_1` — **UP** (зелёный)
   - `mtproxy_2` — **DOWN** (красный, определяется за ~6 сек)

2. **Telegram**: подключение **продолжает работать** — трафик автоматически идёт через `mtproxy_1`.

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

## Персистентность сетевых правил

Правила TPROXY сохраняются между перезагрузками:

- **sysctl** — записываются в `/etc/sysctl.d/`, применяются автоматически при загрузке.
- **ip rule / ip route** — `tproxy-routing.service` (systemd oneshot) восстанавливает правила маршрутизации после ребута.
- **iptables** — правила сохраняются через `iptables-persistent` (`/etc/iptables/rules.v4`).

## Структура проекта

```
testovoe_zadanie/
├── ansible.cfg                    # конфигурация Ansible
├── requirements.yml               # зависимости (community.docker, ansible.posix)
├── inventory/
│   ├── hosts.yml                  # целевые хосты (через env vars)
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
│       ├── tasks/main.yml         # TPROXY-настройка, персистентность, деплой
│       ├── templates/
│       │   └── haproxy.cfg.j2     # Jinja2-шаблон с динамическим backend
│       └── handlers/main.yml      # валидация + graceful reload (SIGUSR2)
└── README.md
```

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
| `haproxy_stats_user` | `admin` | Логин для Stats Dashboard |
| `haproxy_stats_password` | `Ch4ng3M3!` | Пароль для Stats Dashboard |
| `haproxy_balance` | `leastconn` | Алгоритм балансировки (leastconn / roundrobin) |
| `haproxy_tproxy` | `false` | Включить TPROXY (usesrc clientip) — только для multi-host |
