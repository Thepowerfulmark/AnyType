# Self-hosted сеть Anytype

Один Linux-хост поднимает свою сеть синхронизации [Anytype](https://anytype.io): Docker Compose, данные на диске и идентичность сети, которая переживает перезапуск и переезд.

Оригинальная база — [anyproto/any-sync-dockercompose](https://github.com/anyproto/any-sync-dockercompose) (MIT, [LICENSE.md](LICENSE.md), [docs/original-base.md](docs/original-base.md)). Compose, сборка образов генерации, шаблоны конфигов и changelog — оттуда. Здесь добавлено одно правило: если `etc/client.yml` уже есть, генерация не перезаписывает `etc/`. После восстановления из бэкапа сеть остаётся той же.

## Как устроено

Клиент Anytype подключается не к публичной сети Anytype, а к этому хосту. Файл `etc/client.yml` говорит клиенту, какие узлы и по каким адресам доступны.

```text
клиент Anytype
    |  etc/client.yml
    v
node-1..3          coordinator          filenode           consensus
объекты            вход в сеть          файлы              согласование
    |                  |                    |                  |
    |                  v                    v                  |
    |               MongoDB              Redis + MinIO         |
    +---------------- идентичность в etc/ --------------------+
                         данные в storage/
```

- **node-1..3** хранят и синхронизируют объекты.
- **coordinator** регистрирует сеть. Его база — MongoDB, один узел replica set `rs0`.
- **filenode** кладёт файлы в MinIO, служебные данные — в Redis.
- **consensus** вместе с coordinator согласует изменения. Общий ключ сети им выдаёт генерация при первом старте.

Образы — `ghcr.io/anyproto/any-sync-*`. Каналы `prod` и `stage1` при `make start` превращаются в совместимые теги. Конкретный тег остаётся как есть.

Mongo, Redis и консоль MinIO слушают только localhost. Порты синхронизации открыты на хосте, их должны видеть клиенты.

| Сервис | По умолчанию |
|---|---|
| node-1..3 | TCP 1001–1003, UDP 1011–1013 |
| coordinator | TCP 1004, UDP 1014 |
| filenode | TCP 1005, UDP 1015 |
| consensus | TCP 1006, UDP 1016 |
| mongo | `127.0.0.1:27001` |
| redis | `127.0.0.1:6379` |
| консоль MinIO | `127.0.0.1:9001` |

Подробнее: [docs/architecture.md](docs/architecture.md). Эксплуатация: [docs/operations.md](docs/operations.md).

## Что нужно

- Linux, Docker Engine и Compose v2 (`docker compose`)
- Исходящий HTTPS, чтобы подтянуть теги `prod` / `stage1`
- Адрес, который видят клиенты
- Открытые TCP и UDP порты из таблицы выше

## Первый запуск

```bash
cp .env.override.example .env.override
# в EXTERNAL_LISTEN_HOSTS — адрес этого хоста
make start
```

`make start`:

1. Собирает маленький образ и пишет `.env` из `.env.default` и `.env.override`.
2. При первом запуске создаёт `etc/` и `etc/client.yml`.
3. Поднимает стек.

`etc/client.yml` загружается в клиент Anytype как конфиг self-hosted сети: [документация Anytype](https://doc.anytype.io/anytype-docs/data-and-security/self-hosting#switching-between-networks).

`.env` руками не правят: следующий `make start` его перезапишет. Изменения кладут в `.env.override`.

## Та же сеть после переезда

Клиенты останутся в тех же пространствах, только если восстановлены оба каталога:

- `storage/` — Mongo, Redis, MinIO и данные нод
- `etc/` — конфиги и `etc/client.yml`

Нет `etc/client.yml` — следующий старт создаст новую сеть. Старые клиенты старые пространства не увидят.

Смена адреса не переписывает уже созданный `etc/client.yml`. `etc/` не удалять. Адреса в существующих файлах `etc/` правят вручную, peer id и ключи подписи оставляют. Подробности в [docs/operations.md](docs/operations.md).

## Команды

```bash
make start     # .env и up -d
make stop      # остановить, данные на месте
make down      # убрать контейнеры, storage/ и etc/ остаются
make logs
make restart   # down + start
make update    # pull, down, start
```

`make clean` делает `docker system prune --all --volumes` на всём Docker хоста. На машине с другими контейнерами его не запускать. `make cleanEtcStorage` удаляет `etc/` и `storage/` и тем самым создаёт новую сеть.

## Чего в репозитории нет

Сгенерированные ключи (`etc/`), данные сервисов (`storage/`) и локальные `.env` / `.env.override` в gitignore. В репозитории только шаблоны: `.env.default` и `.env.override.example`.
