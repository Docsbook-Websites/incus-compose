---
date: 2026-08-08T02:12:01.000Z
description: Run Immich, self-hosted photo and video backup, on Incus - five services following Immich's own Compose layout.
title: Immich
---

# Immich

[Immich](https://immich.app/), a self-hosted photo and video backup solution.

The files for this example are on [GitHub](https://github.com/lxc/incus-compose/tree/main/examples/immich).

## The example

Five services, following [Immich's official Compose layout](https://docs.immich.app/install/docker-compose): `server`, `machine-learning`, `microservices` (background workers, split from `server` via `IMMICH_WORKERS_INCLUDE`/`EXCLUDE`), `redis`, and `database` (a Postgres fork with vector search support). Version, secrets, and storage paths come from `.env`.

`server` and `microservices` share the `library` volume; both depend on `redis` and `database` being healthy, and `server` also depends on `machine-learning`.

## Usage

```bash
incus-compose up
```

Open http://10.131.32.17:2283/

## Notes

- `library`'s storage pool comes from `UPLOAD_POOL` in `.env` via `x-incus-compose.pool`; set it before first run if you need another pool than the default.

This is the demo recorded during the beta and referenced from the [project README](/) - the workflow is unchanged in current releases.
