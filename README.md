# MTProxy + HAProxy TPROXY + WireGuard — Ansible

Ansible-проект для развёртывания Telegram MTProxy с балансировкой через HAProxy и прозрачным проксированием (TPROXY) — клиентский IP сохраняется на бэкенде. Два VPS соединены WireGuard-туннелем.

---

## Тестовый стенд

| Ресурс | Адрес |
|---|---|
| **Proxy (Telegram)** | `tg://proxy?server=5.10.213.161&port=8443&secret=ddb3b34e89c9471207ad107a03b565e9a3` |
| **Proxy (браузер)** | [t.me/proxy?server=5.10.213.161&port=8443&secret=ddb3b34e89c9471207ad107a03b565e9a3](https://t.me/proxy?server=5.10.213.161&port=8443&secret=ddb3b34e89c9471207ad107a03b565e9a3) |
| **HAProxy Stats** | [http://5.10.213.161:8404/stats](http://5.10.213.161:8404/stats) — `admin` / `Ch4ng3M3!` |
| **MTProxy Dashboard** | [http://206.245.131.154:8080](http://206.245.131.154:8080) — `admin` / `Ch4ng3M3!` |

| Хост | Роль | IP | WireGuard |
|---|---|---|---|
| Балансер | HAProxy + TPROXY | `5.10.213.161` | `10.8.0.1` |
| Бэкенд | MTProxy ×2 | `206.245.131.154` | `10.8.0.2` |

---

## Архитектура

```
 Telegram        ┌──────────────────────────────────┐
 клиент          │  Балансер  5.10.213.161           │
   │             │                                   │
   └──► :8443 ──►│  HAProxy (source 0.0.0.0          │
                 │           usesrc clientip)         │
                 │       │                           │
                 │       │  wg0  10.8.0.1            │
                 └───────┼───────────────────────────┘
                         │  WireGuard  UDP :51820
                         ▼
                 ┌───────────────────────────────────┐
                 │  Бэкенд  206.245.131.154          │
                 │                                   │
                 │       wg0  10.8.0.2               │
                 │       │                           │
                 │       ├──► mtproxy_1  :4431       │
                 │       └──► mtproxy_2  :4432       │
                 │                 │                  │
                 │  iptables mark + ip rule fwmark 1 │
                 │  → ответы уходят обратно через wg0│
                 └───────────────────────────────────┘
                                   │
                                   ▼
                            Telegram DC
```

### Поток TPROXY

1. Клиент → HAProxy `5.10.213.161:8443`
2. HAProxy открывает TCP к `10.8.0.2:4431` с **source IP = IP клиента** (`usesrc clientip`)
3. Пакет уходит через WireGuard — внешне от IP балансера, внутри туннеля source IP подменён
4. MTProxy на бэкенде видит реальный IP клиента
5. Ответ от MTProxy помечается `iptables -t mangle -A OUTPUT --sport 4431 -j MARK --set-mark 1`
6. `ip rule fwmark 1 lookup 100` → `default dev wg0 table 100` — ответ идёт обратно через WireGuard
7. На балансере `iptables -t mangle PREROUTING -i wg0 --sport 4431 -j MARK --set-mark 1` → `local 0.0.0.0/0 dev lo table 100` доставляет пакет в TPROXY-сокет HAProxy
8. HAProxy отдаёт ответ клиенту

### Зачем WireGuard?

Без туннеля TPROXY между двумя VPS не работает:

- **BCP38** — провайдер дропает исходящие пакеты с чужим source IP
- **Асимметричный маршрут** — ответ от MTProxy уходит напрямую клиенту, минуя HAProxy, и соединение ломается

WireGuard решает обе проблемы: снаружи пакет идёт с реальным IP сервера, а внутри туннеля source IP может быть любым.

### NAT info (`--nat-info`)

MTProxy требует параметр `--nat-info LOCAL_IP:GLOBAL_IP` для корректной работы Telegram-протокола. В схеме с WireGuard:

- `LOCAL_IP` = `10.8.0.2` — адрес, на котором MTProxy принимает соединения (WireGuard-интерфейс)
- `GLOBAL_IP` = `5.10.213.161` — публичный адрес, по которому подключаются клиенты (IP балансера)

Это настраивается через переменные `mtproxy_nat_internal_ip` и `mtproxy_nat_external_ip`. Кастомный `run.sh` передаёт их в MTProxy вместо автоопределения.

---

## Требования

- **2 VPS** с публичными IP (Debian 11+ / Ubuntu 22.04+)
- **Ansible** >= 2.14 на управляющей машине
- **sshpass** — если используется аутентификация по паролю
- Открытые порты:
  - **TCP 8443** на балансере (вход для клиентов)
  - **UDP 51820** между серверами (WireGuard)

---

## Быстрый старт

### 1. Установить зависимости

```bash
ansible-galaxy collection install -r requirements.yml
sudo apt install -y sshpass  # если используется ansible_password
```

### 2. Настроить inventory

`inventory/hosts.yml`:

```yaml
all:
  vars:
    ansible_user: root
    ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
  children:
    balancers:
      hosts:
        balancer:
          ansible_host: <IP_БАЛАНСЕРА>
          ansible_password: <ПАРОЛЬ>
          wireguard_address: "10.8.0.1/24"
    backends:
      hosts:
        backend:
          ansible_host: <IP_БЭКЕНДА>
          ansible_password: <ПАРОЛЬ>
          wireguard_address: "10.8.0.2/24"
```

> Для продакшена замените `ansible_password` на SSH-ключи или используйте `ansible-vault`.

### 3. Настроить переменные

`inventory/group_vars/all.yml`:

```yaml
mtproxy_secret: "<openssl rand -hex 16>"
mtproxy_nat_internal_ip: "10.8.0.2"        # WireGuard IP бэкенда
mtproxy_nat_external_ip: "<IP_БАЛАНСЕРА>"   # публичный IP, по которому клиенты подключаются
```

### 4. Запустить playbook

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

### 5. Подключиться через Telegram

```
tg://proxy?server=<IP_БАЛАНСЕРА>&port=8443&secret=dd<ваш_секрет>
```

Префикс `dd` перед секретом включает fake-TLS (рекомендуется).

---

## Проверка работы

### HAProxy Stats Dashboard

```
http://<IP_БАЛАНСЕРА>:8404/stats
```

Оба бэкенда `mtproxy_1` и `mtproxy_2` должны быть **UP** (зелёные).

### WireGuard туннель

```bash
# На любом хосте
wg show
ping -c 3 10.8.0.2   # с балансера
ping -c 3 10.8.0.1   # с бэкенда
```

### MTProxy Dashboard — реальные IP клиентов

Веб-панель на бэкенде показывает активные клиентские соединения в реальном времени:

```
http://<IP_БЭКЕНДА>:8080
```

Таблица соединений содержит:

| Колонка | Описание |
|---------|----------|
| **Клиентский IP (белый)** | Реальный публичный IP клиента Telegram |
| **Клиентский порт** | Исходный порт клиента |
| **IP туннеля (WireGuard)** | IP интерфейса wg0, на котором MTProxy принял соединение |
| **Порт MTProxy** | Порт инстанса (4431, 4432, ...) с именем контейнера |

Также отображаются: статистика (активные соединения, уникальные клиенты), статус Docker-контейнеров и WireGuard.

Страница обновляется автоматически каждые 5 секунд. Доступен JSON API: `GET /api/connections`.

### TPROXY — проверка через CLI

```bash
# На бэкенде
conntrack -L -p tcp --dport 4431 2>/dev/null | head
ss -tn state established '( sport = :4431 or sport = :4432 )'
```

В `src=` должны быть реальные клиентские IP, а не `10.8.0.1`.

### Failover

```bash
ssh root@<IP_БЭКЕНДА>
docker stop mtproxy_2
# Stats Dashboard: mtproxy_2 → DOWN (~6 сек), трафик идёт через mtproxy_1
docker start mtproxy_2
# mtproxy_2 → UP (~6 сек)
```

---

## Масштабирование

Изменить количество реплик в `inventory/group_vars/all.yml`:

```yaml
mtproxy_replicas: 4
```

Перезапустить playbook — Ansible создаст новые контейнеры и обновит конфиг HAProxy. Контейнеры с индексом выше `mtproxy_replicas` удаляются автоматически (scale-down до `mtproxy_max_cleanup_index`).

---

## Персистентность

Все настройки переживают перезагрузку:

| Компонент | Механизм |
|---|---|
| sysctl | `/etc/sysctl.d/` — применяется при загрузке |
| ip rule / ip route | `tproxy-routing.service` (systemd oneshot) |
| iptables | `iptables-persistent` → `/etc/iptables/rules.v4` |
| WireGuard | `wg-quick@wg0.service` (systemd) |
| Docker-контейнеры | `restart_policy: unless-stopped` |
| MTProxy Dashboard | `mtproxy-dashboard.service` (systemd) |

---

## Ротация конфигурации

При изменении переменных:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

HAProxy выполнит graceful reload (`SIGUSR2`) без разрыва существующих соединений.

---

## Структура проекта

```
testovoe_zadanie/
├── ansible.cfg                        # настройки Ansible
├── requirements.yml                   # зависимости (community.docker, ansible.posix)
├── inventory/
│   ├── hosts.yml                      # хосты: балансер + бэкенд
│   └── group_vars/
│       └── all.yml                    # все переменные проекта
├── playbooks/
│   └── site.yml                       # главный playbook (3 play)
└── roles/
    ├── wireguard/
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml             # установка, генерация ключей, обмен
    │   ├── templates/wg0.conf.j2      # конфиг с автообменом ключей между хостами
    │   └── handlers/main.yml
    ├── docker/
    │   └── tasks/main.yml             # установка Docker CE + Python SDK
    ├── mtproxy/
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml             # создание N контейнеров + scale-down
    │   ├── templates/run.sh.j2        # кастомный entrypoint с поддержкой NAT_*_IP
    │   └── handlers/main.yml
    ├── tproxy_backend/
    │   ├── tasks/main.yml             # rp_filter, iptables mark, route via wg0
    │   └── handlers/main.yml
    ├── haproxy/
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml             # sysctl, TPROXY routing, iptables, контейнер
    │   ├── templates/haproxy.cfg.j2   # конфиг с usesrc clientip
    │   └── handlers/main.yml
    └── mtproxy_dashboard/
        ├── defaults/main.yml
        ├── tasks/main.yml             # деплой веб-панели + systemd
        ├── templates/dashboard.py.j2  # Python HTTP-сервер (без зависимостей)
        └── handlers/main.yml
```

---

## Переменные

### MTProxy

| Переменная | Умолчание | Описание |
|---|---|---|
| `mtproxy_replicas` | `2` | Количество инстансов MTProxy |
| `mtproxy_secret` | `000...` | Секрет прокси (32 hex символа) |
| `mtproxy_image` | `telegrammessenger/proxy:latest` | Docker-образ |
| `mtproxy_base_port` | `4431` | Базовый порт (инстансы: 4431, 4432, ...) |
| `mtproxy_max_cleanup_index` | `10` | Верхний индекс для scale-down очистки |
| `mtproxy_nat_internal_ip` | — | IP, на котором MTProxy принимает соединения (WireGuard) |
| `mtproxy_nat_external_ip` | — | Публичный IP для клиентов (IP балансера) |

### HAProxy

| Переменная | Умолчание | Описание |
|---|---|---|
| `haproxy_image` | `haproxy:2.9-alpine` | Docker-образ |
| `haproxy_frontend_port` | `8443` | Порт входа для клиентов |
| `haproxy_stats_port` | `8404` | Порт Stats Dashboard |
| `haproxy_stats_user` | `admin` | Логин Stats Dashboard |
| `haproxy_stats_password` | `Ch4ng3M3!` | Пароль Stats Dashboard |
| `haproxy_balance` | `leastconn` | Алгоритм балансировки |
| `haproxy_tproxy` | `false` | TPROXY (`usesrc clientip`) |
| `mtproxy_backend_ip` | `127.0.0.1` | IP бэкенда для HAProxy (WireGuard IP при multi-host) |

### MTProxy Dashboard

| Переменная | Умолчание | Описание |
|---|---|---|
| `dashboard_port` | `8080` | Порт веб-панели |
| `dashboard_user` | `admin` | Логин Basic Auth |
| `dashboard_password` | `Ch4ng3M3!` | Пароль Basic Auth |

### WireGuard

| Переменная | Умолчание | Описание |
|---|---|---|
| `wireguard_listen_port` | `51820` | UDP-порт WireGuard |
| `wireguard_address` | per-host | WireGuard IP (`10.8.0.x/24`) |

---

## Роли

### `wireguard`

Устанавливает WireGuard, генерирует ключи на каждом хосте, автоматически обменивается публичными ключами через Ansible facts и разворачивает `wg0.conf`. На бэкенде `Table = off` — маршруты управляются вручную через policy routing.

### `docker`

Устанавливает Docker CE и Python Docker SDK (`python3-docker`) — необходим для модуля `community.docker`.

### `mtproxy`

Разворачивает N контейнеров MTProxy в `--network host` режиме. Каждый инстанс получает свой порт (`mtproxy_base_port + i`). Кастомный `run.sh` принимает `NAT_INTERNAL_IP` / `NAT_EXTERNAL_IP` через env-переменные — без них MTProxy автоопределяет IP и ломается в WireGuard-сетапе. Поддерживает scale-down — лишние контейнеры удаляются.

### `tproxy_backend`

Настраивает return-path маршрутизацию на бэкенде: отключает `rp_filter`, создаёт policy route (`fwmark 1 → table 100 → default dev wg0`), помечает ответы MTProxy через `iptables mangle OUTPUT`. Systemd-сервис обеспечивает персистентность.

### `haproxy`

Настраивает TPROXY на стороне балансера: `ip_nonlocal_bind`, policy routing для return-трафика, `iptables mangle PREROUTING -i wg0`. Рендерит `haproxy.cfg` из шаблона и запускает контейнер с `NET_ADMIN` + `NET_RAW` capabilities. Graceful reload через `SIGUSR2`.

### `mtproxy_dashboard`

Разворачивает веб-панель мониторинга активных клиентских соединений на бэкенде. Чистый Python 3 без внешних зависимостей. Показывает таблицу соединений (реальный IP клиента, порт, IP туннеля, порт MTProxy), статус Docker-контейнеров и WireGuard. Авто-обновление каждые 5 секунд, Basic Auth, JSON API (`/api/connections`). Работает как systemd-сервис.
