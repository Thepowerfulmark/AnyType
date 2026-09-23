# Original base

This repository is a snapshot of [anyproto/any-sync-dockercompose](https://github.com/anyproto/any-sync-dockercompose) plus one local guard in `docker-generateconfig/processing.sh`.

The base is published by Any Association under the MIT license ([LICENSE.md](../LICENSE.md)). Node images are `ghcr.io/anyproto/any-sync-*`. Version channels `prod` and `stage1` are resolved from `https://puppetdoc.anytype.io` in `docker-generateconfig/env.py`.

## What comes from the base

- `docker-compose.yml` and the per-service compose fragments
- `Dockerfile`, `Dockerfile-generateconfig-*`
- `Makefile` targets (`start`, `stop`, `down`, `update`, `clean`, `cleanEtcStorage`)
- `docker-generateconfig/` (anyconf, env merge, listen-address rewrite, node config templates)
- `.env.default` and the placeholder MinIO credentials in it
- `.github/workflows/cla.yml` and `release.yml` (upstream automation; the CLA job calls `anyproto/open`)
- [CHANGELOG.md](../CHANGELOG.md) through v5.0.0 (2024-06-24)

Upstream has moved on since this snapshot. New files there (S3 migration helpers, `update-versions.sh`, Garage config) are not part of this tree. Do not treat this repository as a mirror of current `main`.

## What was added here

`docker-generateconfig/processing.sh` returns immediately when `etc/client.yml` is already on disk. The base generates a new network config whenever that step runs. After a backup restore, that would replace peer ids and the client would see a different network.

## What was left out on purpose

These exist on a running host and must not be published:

- `storage/` — Mongo, Redis, MinIO and node data
- `etc/` — generated `client.yml`, peer ids, signing keys
- `.env` and `.env.override` — the host address and any changed credentials

`.gitignore` excludes them. `.env.override.example` is the only override file in git.
