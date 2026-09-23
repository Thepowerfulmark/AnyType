# Architecture

One host runs a private Any Sync network. Anytype clients talk to it instead of the public Anytype network. Spaces, objects and files stay on this host.

## Components

```text
Anytype client
    |  etc/client.yml (peer ids + EXTERNAL_LISTEN_HOSTS)
    v
node-1..3          coordinator           filenode            consensus
TCP/UDP            TCP/UDP               TCP/UDP             TCP/UDP
    |                  |                     |                  |
    |                  v                     v                  |
    |               MongoDB               Redis + MinIO         |
    +---------------- network identity in etc/ -----------------+
                         data in storage/
```

- **Sync nodes** store and replicate objects. Three nodes is the layout of this compose file.
- **Coordinator** is the entry the client uses to join the network. Its database is a single-node Mongo replica set (`rs0`).
- **Filenode** stores file blobs in MinIO and metadata in Redis.
- **Consensus node** agrees on changes together with the coordinator. Both receive the same network signing key at generation time (`docker-generateconfig/anyconf.sh`).

Images are pulled from `ghcr.io/anyproto/any-sync-*`. Tags come from `.env`, which `make start` builds from `.env.default` and `.env.override`. Values `prod` and `stage1` are replaced with the compatible versions published by Anytype. A concrete tag is left unchanged.

## Network identity

`etc/client.yml` is the file the client imports. It lists peer ids and the addresses from `EXTERNAL_LISTEN_HOSTS`.

`docker-generateconfig/processing.sh` exits before generation when that file is already present. Restart, image update and a changed listen address do not mint a new network. A new network appears only when `etc/` is absent.

Generated files also hold node signing keys. They are secrets of this deployment. They are created on first start and are not part of the git tree.

## Ports

Defaults live in `.env.default`.

| Name | TCP | UDP | Also published |
|---|---|---|---|
| node-1 | 1001 | 1011 | API `0.0.0.0:8081`, metrics `0.0.0.0:8001` |
| node-2 | 1002 | 1012 | API `0.0.0.0:8082`, metrics `0.0.0.0:8002` |
| node-3 | 1003 | 1013 | API `0.0.0.0:8083`, metrics `0.0.0.0:8003` |
| coordinator | 1004 | 1014 | metrics `0.0.0.0:8004` |
| filenode | 1005 | 1015 | metrics `0.0.0.0:8005` |
| consensus | 1006 | 1016 | metrics `0.0.0.0:8006` |
| mongo | 127.0.0.1:27001 | | |
| redis | 127.0.0.1:6379 | | |
| minio console | 127.0.0.1:9001 | | API 9000 stays on the compose network |

Sync ports are bound on all host interfaces. Database ports are bound to localhost.

## Disk

`STORAGE_DIR` defaults to `./storage`.

| Path | Contents |
|---|---|
| `storage/mongo-1` | Coordinator data |
| `storage/redis` | Filenode Redis |
| `storage/minio` | File blobs |
| `storage/any-sync-node-*`, `storage/anyStorage`, `storage/networkStore` | Node state |
| `etc/` | Generated configs, `client.yml`, AWS credential file for MinIO |

MinIO credentials in `.env.default` are local placeholders (`minio_access_key` / `minio_secret_key`). Change them in `.env.override` before the first start if the host is reachable beyond a lab.
