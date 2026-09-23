# Self-hosted Anytype sync network

Self-hosted [Anytype](https://anytype.io) sync network on one Linux host: Docker Compose, persistent data, and a network identity that survives restart and migration.

The original base is [anyproto/any-sync-dockercompose](https://github.com/anyproto/any-sync-dockercompose) (MIT, see [LICENSE.md](LICENSE.md) and [docs/original-base.md](docs/original-base.md)). Compose, image build, config generation and the upstream changelog are that project. This tree adds one operational rule: if `etc/client.yml` already exists, config generation does not overwrite `etc/`. That keeps the same network after a restore.

## What runs

| Service | Role | Published by default |
|---|---|---|
| any-sync-node-1..3 | Object sync | TCP 1001–1003, UDP 1011–1013 |
| any-sync-coordinator | Network coordination | TCP 1004, UDP 1014 |
| any-sync-filenode | File storage (MinIO) | TCP 1005, UDP 1015 |
| any-sync-consensusnode | Consensus | TCP 1006, UDP 1016 |
| mongo-1 | Coordinator database | `127.0.0.1:27001` only |
| redis | Filenode index | `127.0.0.1:6379` only |
| minio | Filenode blobs | console `127.0.0.1:9001` |

Mongo, Redis and the MinIO console stay on localhost. Sync ports are published on the host and must be reachable by clients.

Details: [docs/architecture.md](docs/architecture.md). Day-2 operations: [docs/operations.md](docs/operations.md).

## Requirements

- Linux with Docker Engine and Compose v2 (`docker compose`)
- Outbound HTTPS, so image tags for `prod` / `stage1` can be resolved
- A public IP or DNS name that clients can reach
- Firewall openings for the TCP and UDP ports above

## First start

```bash
cp .env.override.example .env.override
# set EXTERNAL_LISTEN_HOSTS to this host's address
make start
```

`make start`:

1. Builds a small image and writes `.env` from `.env.default` plus `.env.override`.
2. Generates `etc/` and `etc/client.yml` on the first run.
3. Starts the stack.

Upload `etc/client.yml` into the Anytype client as the self-hosted network config: [Anytype self-hosting](https://doc.anytype.io/anytype-docs/data-and-security/self-hosting#switching-between-networks).

Do not edit `.env` by hand. The next `make start` regenerates it. Put changes in `.env.override`.

## Same network after a move

Clients stay on the same spaces only if both of these are restored:

- `storage/` — Mongo, Redis, MinIO and node data
- `etc/` — node configs and `etc/client.yml`

If `etc/client.yml` is missing, the next start creates a new network. Old clients will not see the old spaces.

When the host address changes, do not delete `etc/`. `make start` will not rewrite an existing `etc/client.yml`. Update the listen addresses in `etc/` and keep the peer ids and signing keys. See [docs/operations.md](docs/operations.md).

## Commands

```bash
make start     # generate .env if needed, then up -d
make stop      # stop containers, keep data
make down      # remove containers, keep storage/ and etc/
make logs
make restart   # down + start
make update    # pull images, down, start
```

`make clean` runs `docker system prune --all --volumes` on the whole Docker host. Do not use it on a machine that has other containers. `make cleanEtcStorage` deletes `etc/` and `storage/` and creates a new network.

## What this repository does not contain

Generated network keys (`etc/`), service data (`storage/`) and the local `.env` / `.env.override` are gitignored. Commit only the templates: `.env.default` and `.env.override.example`.
