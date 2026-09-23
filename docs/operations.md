# Operations

## Configure

```bash
cp .env.override.example .env.override
```

Set `EXTERNAL_LISTEN_HOSTS` to the address clients use. Several addresses are space-separated. `0.0.0.0` inside the container is not a client address.

Version keys:

- `prod` or `stage1` — resolved on each `make start` from the Anytype version feed
- an explicit image tag — pinned, no lookup

Other keys in `.env.default` (ports, Mongo, Redis, MinIO) can be overridden the same way.

## Start, stop, update

```bash
make start
make logs
make stop
make down
make update
```

`make update` pulls images, recreates containers and keeps `storage/` and `etc/`.

Check that clients can open TCP and UDP for ports 1001–1006 and 1011–1016. If those ports are closed, the client imports `client.yml` and still cannot sync.

## Backup

Stop the stack so files are consistent:

```bash
make stop
tar -C /opt/anytype-anysync -czf anytype-backup.tgz storage etc .env.override
make start
```

Back up all three. Mongo alone is not enough: files live in MinIO, and the network identity lives in `etc/`.

`.env` can be omitted. `make start` rebuilds it.

## Restore onto a new host

1. Install Docker and Compose v2.
2. Clone this repository.
3. Unpack `storage/`, `etc/` and `.env.override` into the repository root.
4. If the address changed, edit `EXTERNAL_LISTEN_HOSTS` only.
5. `make start`.

Expected result: containers come up, `etc/client.yml` is left untouched, existing clients keep their spaces.

If `etc/client.yml` was not restored, generation creates a new network. Do not point old clients at it and expect old data.

## Change the listen address

1. Edit `.env.override`.
2. `make start`.

The guard in `docker-generateconfig/processing.sh` skips regeneration, so peer ids stay the same. Addresses inside an already generated `etc/client.yml` are not rewritten. After an address change, either update the addresses in the existing `etc/*.yml` or generate a fresh `etc/` only when a new network is acceptable. For a restored network, edit the listen addresses in `etc/` to the new host and keep the peer ids and signing keys.

## Destructive commands

| Command | Effect |
|---|---|
| `make down` | Removes containers. Data remains. |
| `make cleanEtcStorage` | Deletes `etc/` and `storage/`. Next start is a new network. |
| `make clean` | `docker system prune --all --volumes` for the whole Docker host. |
| `make upgrade` | `down`, then `clean`, then `start`. Same host-wide prune. |

## Local checks

`tests/main.sh` drives `make` with overrides under `tests/run.d/`. It writes a `backup/` directory (gitignored). Run it only on a host where this compose project is the one you are willing to stop.
