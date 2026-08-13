---
date: 2026-08-09T08:12:41.000Z
description: Health checks and restart policies on Incus, which has neither natively - how the ic-healthd sidecar watches your services and restarts what fails.
title: Health Checking (ic-healthd)
---

# Health Checking (ic-healthd)

incus-compose implements health checks via a sidecar container called `ic-healthd`.
Incus has no native healthcheck support, so ic-healthd fills that role.

> **ic-healthd is a core component.** Every `healthcheck`, every restart policy
> (`restart: always | on-failure | unless-stopped`), and every
> `depends_on: { condition: service_healthy }` is enforced by this sidecar, not by
> Incus. If healthd is misconfigured, stopped, or crashing:
>
> - instances are not restarted, and
> - **the project may fail to come up at all**: `incus-compose up` waits for
>   `service_healthy` dependencies to be reported healthy by healthd. If that
>   status never arrives, `up` blocks until `--dependency-timeout` (default 5m;
>   `0` waits forever) and then fails.
>
> Opt out of healthd entirely with `incus-compose up --no-healthd` (this also
> drops the dependency wait); `--no-deps` skips the wait too.

## How It Works

`incus-compose up` makes sure a healthd is watching the project when any service
declares a `healthcheck`, has a restart policy other than `no`, or is depended on
with `condition: service_healthy`. By default that is one daemon shared by the
whole server; see Scope below. It:

1. Marks the Incus project `user.healthcheck.scope`, which is how the daemon finds it.
2. Creates the `ic-healthd` container if it is not already there, with an Incus trust token.
3. ic-healthd authenticates once (token consumed) and persists the resulting cert.
4. ic-healthd discovers which instances to watch by reading the Incus API, then opens an Incus lifecycle event listener and reacts to
   project and instance create/update/delete/start/stop events from then on - no
   polling, no reload needed for config or instance-set changes to take effect.
5. ic-healthd runs the health loop per watched instance and writes the result to
   `user.healthcheck.status`.

The daemon is running before the regular services start, so `service_healthy`
dependencies can be evaluated. A project-scoped sidecar is removed by
`incus-compose down`; the shared daemon is never touched by it.

## Config Storage

Health check config and runtime state live in the instance's `user.*` config keys.
There is no separate config file. ic-healthd reacts to `incus config set`/instance
create/delete changes as they happen via the Incus event stream; `incus-compose
healthd reload` remains available to force a full manual resync.

```
user.incus-compose.managed       true
user.healthcheck.enabled         true
user.healthcheck.test            '["CMD","wget","-q","--spider","http://localhost"]'
user.healthcheck.start_period    10s
user.healthcheck.start_interval  2s
user.healthcheck.interval        10s
user.healthcheck.timeout         5s
user.healthcheck.retries         3
user.healthcheck.status          unknown | stopped | starting | healthy | unhealthy
user.healthcheck.restart         always | on-failure | unless-stopped
user.healthcheck.ignore          true
```

These keys are visible in `incus config show <instance>`.

`user.healthcheck.status` is the only key ic-healthd writes, and **nothing else
writes it**. All the others are set by incus-compose at instance creation time
and treated as read-only by the daemon.

## Health Checking Is Opt-In

ic-healthd watches an instance only when it carries
`user.healthcheck.enabled: "true"`. A `healthcheck:` block or a restart policy
alone is no longer enough - the instance has to say it wants watching.

incus-compose writes the key automatically for every service that declares a
`healthcheck:` or a restart policy other than `no`, so you do not normally set
it by hand.

### Opting a service out

Set `user.healthcheck.enabled: "false"` via `x-incus`. The service keeps its
`healthcheck:` block - it simply is not watched:

```yaml
services:
  sidecar-tool:
    image: docker.io/example/tool:latest
    x-incus:
      user.healthcheck.enabled: "false"
```

## Scope: One Daemon Or One Per Project

One ic-healthd watches any number of projects from a single Incus event
listener, so by default there is exactly one on the server:

| Scope              | Where it runs                                                | Watches                                                |
| -------------------- | --------------------------------------------------------------- | --------------------------------------------------------- |
| `global` (default) | instance `ic-healthd` in the Incus `incus-compose` project    | every project marked `user.healthcheck.scope=global`   |
| `project`          | instance `{project}-ic-healthd` in the project                 | that one project                                         |

### Choosing project scope

```yaml
x-incus-compose:
  healthd:
    scope: project
```

or `incus-compose up --healthd-scope project` the first time. Reasons to:

- **Least privilege.** A project-scoped sidecar gets an Incus certificate
  restricted to its own project. The shared one cannot be restricted - it has to
  reach projects that do not exist yet - so it is registered unrestricted.
- **Isolation.** A wedged or stopped daemon takes down health checking for its
  own project only.

## Defaults

When keys are missing, ic-healthd falls back to:

| Key            | Default        |
| ---------------- | ---------------- |
| start_period   | 0s (disabled)  |
| start_interval | 5s             |
| interval       | 30s            |
| timeout        | 30s            |
| retries        | 3              |

`retries` must be greater than 0.

After `retries` consecutive failures the instance is restarted. The first
restart waits `interval * retries`; the delay doubles on every further restart,
capped at 5 minutes.

## Dockerfile HEALTHCHECK Not Supported

incus-compose does not read or inherit the `HEALTHCHECK` instruction embedded in Docker images.

Incus imports OCI images via umoci, which converts the OCI image config into an
OCI runtime spec. The Docker `HEALTHCHECK` extension is not part of the OCI image
spec and is discarded during that conversion. Fetching it from the registry at
`up` time would require registry access on every run and fails in air-gapped
environments.

**Workaround:** Always declare `healthcheck.test` explicitly in the compose file:

```yaml
services:
  db:
    image: docker.io/postgres:16-alpine
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

## Restart Without a Test

`restart: always`, `on-failure`, or `unless-stopped` without a `healthcheck`
block is also handled. ic-healthd monitors the instance state and restarts it
when stopped, without running an exec-based test command.

With `unless-stopped`, instances stopped intentionally (`user.healthcheck.stopped=true`,
set by `incus-compose stop`) are not restarted.

## Network Configuration

ic-healthd runs in its own container and must reach the Incus HTTPS API from the
inside. Two things are configured:

- **`network`** - the Incus network (or host bridge) healthd attaches its NIC to.
- **`incus`** - the Incus API URL healthd connects to.

```yaml
name: my-project
x-incus-compose:
  healthd:
    incus: https://<ip-of-the-projects-bridge>:8443
    network: :default
```

| Flag                 | Environment variable              | Compose key                          |
| ---------------------- | ------------------------------------ | --------------------------------------- |
| `--healthd-incus`   | `INCUS_COMPOSE_HEALTHD_INCUS`      | `x-incus-compose.healthd.incus`       |
| `--healthd-network` | `INCUS_COMPOSE_HEALTHD_NETWORK`    | `x-incus-compose.healthd.network`     |

## Security

Whichever daemon watches a project can exec into its instances and start, stop
and restart them. What differs is how far that reaches.

**A project-scoped sidecar** gets a restricted certificate: can only exec/manage instances within its own project.

**The shared daemon is registered unrestricted**, deliberately, so it can pick
up projects created after it was registered. Practically it means a
compromised `ic-healthd` container is a compromised Incus server. If that is
not a trade you want to make, use `scope: project` - every project then gets a
daemon bounded to itself, at the cost of one container each.

## Management Commands

The `healthd` command group manages the sidecar directly without touching
services. Each follows the project's scope:

| Subcommand         | Description                                                    |
| --------------------- | ------------------------------------------------------------------ |
| `logs [--follow]`  | Stream the ic-healthd container log                             |
| `reload`           | Send SIGHUP to force a full manual resync (rarely needed)       |
| `restart`          | Restart the ic-healthd container                                |
| `up`               | Create the sidecar, or replace one running an older image       |
| `down [--force]`   | Stop and remove the sidecar                                     |

## Disabling the Sidecar

```bash
incus-compose up --no-healthd
```

## Using Your Own healthd

You can run the daemon yourself instead of letting `up` create a sidecar, and
point incus-compose at it with `up --external-healthd` / `down
--external-healthd`. Set it permanently for a project in the compose file:

```yaml
x-incus-compose:
  healthd:
    external: true
```

## Sidecar Image

Default image: `ghcr.io/lxc/incus-compose/ic-healthd:{version}`

Override with `--healthd-image` flag or `INCUS_COMPOSE_HEALTHD_IMAGE` env var.

Both `incus-compose up` and `incus-compose healthd up` upgrade the daemon for
you: when the image you ask for is a _newer_ release than the one it is running,
the sidecar is stopped, removed and recreated from that image.

## Debugging ic-healthd

Because healthd drives all health and restart behavior, most "container did not
restart" or "stuck `service_healthy`" problems are diagnosed from the sidecar.
Work through these in order.

### 1. Check the reported health status

```bash
incus config get web-1 user.healthcheck.status --project <project>
```

`starting` that never becomes `healthy` means the test never passes within the
start period; `unhealthy` means it failed `retries` times.

### 2. Inspect the config keys healthd reads

```bash
incus config show web-1 --project <project> | grep -E 'user\.(healthcheck|restart)'
```

### 3. Watch the daemon logs

```bash
incus-compose healthd logs --follow
```

### 4. Confirm the sidecar is actually running

```bash
incus-compose list        # the daemon is listed by default (since 1.0.0-rc.1)
incus-compose healthd up  # create it if missing
```

### 5. Reproduce the health test by hand

```bash
incus-compose exec <service> -- sh -c 'wget -q --spider http://localhost; echo exit: $?'
```

### 6. Force a manual resync

```bash
incus-compose healthd reload   # sends SIGHUP
```

### `incus-compose up` hangs or times out on dependencies

If a service uses `depends_on: { condition: service_healthy }`, `up` waits for
healthd to report the dependency `healthy` before starting the dependent service.
A broken or missing healthd means that status never arrives and `up` blocks until
`--dependency-timeout` (default 5m) elapses, then fails.

```bash
incus-compose up --no-healthd   # also stops managing healthchecks/restarts
# or keep healthd but skip the wait:
incus-compose up --no-deps
```

## See Also

- [Compose Compatibility](/compose-compatibility) - healthcheck and restart policy support
