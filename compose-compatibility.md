---
date: 2026-08-09T07:22:10.000Z
description: Which parts of the Compose Specification incus-compose supports, what it does differently, and the x-incus extensions for Incus-only options.
title: Compose Compatibility
---

# Compose Compatibility

incus-compose implements a subset of the Compose Specification. This doc lists what works and what doesn't.

## Supported Features

### Incus Override File

If a `compose.incus.yaml` file exists next to the selected `compose.yaml`, incus-compose loads it automatically as an additional Compose file. Use it for Incus-specific overrides while keeping the upstream Docker Compose file unchanged.

```text
compose.yaml
compose.incus.yaml
```

Example `compose.incus.yaml`:

```yaml
services:
  web:
    ports: !reset []
    x-incus:
      limits.memory: 512MiB

networks:
  default:
    x-incus:
      ipv4.address: 10.100.0.2/24
      ipv4.gateway: 10.100.0.1
```

Running with the base file also applies the Incus override when present:

```bash
incus-compose -f compose.yaml up
```

The override file follows normal Compose merge rules. For example, `!reset []` clears a list from the base file.

### Services

- `image` - OCI images from any registry
- `command` - Override container command (appends, see below)
- `entrypoint` - Override the container entrypoint (see below)
- `working_dir` - Set working directory
- `user` - Run the container process as a specific UID/GID (numeric only, see below)
- `dns` / `dns_search` / `domainname` - DNS resolver configuration (see below)
- `environment` - Environment variables
- `labels` - Metadata (stored as `user.label.*` config, see below)
- `depends_on` - Service dependency order
- `networks` - Multiple networks per service
- `ports` - Port publishing
- `volumes` - Named volumes and bind mounts
- `deploy.replicas` - Service scaling (instances named `{service}-{index}`)
- `restart` - Restart policies (`no`, `always`, `on-failure`, `unless-stopped`)
- `x-incus` extension - pass any Incus project, network and instance option directly (see below)
- Top-level `x-incus-compose.healthd` - configure the ic-healthd sidecar's network and Incus endpoint (see below)

#### Labels

Compose `labels` are stored on the instance as `user.label.<key>` config keys.
Both the map and list forms work:

```yaml
services:
  app:
    image: docker.io/nginx:alpine
    labels:
      caddy: whoami.example.com
      caddy.reverse_proxy: "{{upstreams 80}}"
  api:
    image: docker.io/nginx:alpine
    labels:
      - "traefik.http.routers.api.rule=Host(`api.example.com`)"
```

becomes:

```yaml
config:
  user.label.caddy: whoami.example.com
  user.label.caddy.reverse_proxy: "{{upstreams 80}}"
  user.label.traefik.http.routers.api.rule: "Host(`api.example.com`)"
```

Two labels are always added:

| Key                                 | Value                     |
| ------------------------------------ | ------------------------- |
| `user.label.incus-compose.project` | the compose project name |
| `user.label.incus-compose.service` | the compose service name |

Read them back with the `incus` passthrough:

```bash
incus-compose incus config get app-1 user.label.caddy
```

**Service discovery** - the `user.label.` prefix keeps compose labels out of the
`user.*` namespace incus-compose uses for its own keys, and mirrors the label
conventions of reverse proxies and DNS managers:

- [Traefik](https://doc.traefik.io/traefik/) - `traefik.enable`, `traefik.http.routers.<name>.rule`, ...
- [caddy-docker-proxy](https://github.com/lucaslorentz/caddy-docker-proxy) - `caddy`, `caddy.reverse_proxy`
- [dnsweaver](https://maxfield-allison.github.io/dnsweaver/) - reads the Traefik router labels above

> None of these tools support incus-compose yet: they discover services over the
> Docker socket, not the Incus API. incus-compose only exposes the labels as
> `user.label.*` instance config; consuming them needs an Incus-aware discovery
> integration.

_Changed in 1.0.0-rc.2_: labels moved from `user.<key>` to `user.label.<key>`, and the
`incus-compose.project` / `incus-compose.service` labels were added.

#### User

The `user` attribute overrides the user the container process runs as, mapping to
the image's `oci.uid` / `oci.gid`:

```yaml
services:
  web:
    image: docker.io/nginx:alpine
    user: "1000:1001" # UID:GID; the GID is optional
```

incus-compose accepts only **numeric** values in `UID` or `UID:GID` form. Usernames
and group names (e.g. `nginx` or `nginx:www-data`) are not resolved and will fail.

> The [Compose Specification](https://github.com/compose-spec/compose-spec/blob/main/05-services.md#user)
> only says `user` "overrides the user used to run the container process" and does
> not document a value format. The `UID:GID` form is Docker's convention; we follow
> it but restrict it to numeric IDs because there is no image passwd/group lookup at
> translation time.

_Since: 1.0.0-beta.22_

#### Entrypoint and Command

`entrypoint:` behaves as the compose spec describes: it replaces the image's
entrypoint, and the image's default command is discarded, so the container runs
exactly `entrypoint:` followed by `command:`.

```yaml
services:
  web:
    image: docker.io/library/busybox:glibc
    entrypoint: ["httpd", "-f", "-v", "-p", "8080", "-h", "/www"]
```

| `entrypoint:` | `command:` | The container runs           |
| ------------- | ---------- | ----------------------------- |
| set           | unset      | `entrypoint`                  |
| set           | set        | `entrypoint` + `command`      |
| set           | `[]`       | `entrypoint`                  |
| `[]`          | set        | `command`                     |
| `[]`          | unset      | rejected - nothing to run     |
| unset         | set        | image entrypoint + `command`  |

**`command:` on its own is appended, not substituted.** That last row is the one
place incus-compose deviates from Docker, and it is a limitation of the Incus
API rather than a choice: Incus derives an OCI container's entrypoint from the
runtime bundle's resolved arguments, which already have the image's `ENTRYPOINT`
and `CMD` concatenated, and never exposes the two separately. Without that split
there is no way to replace `CMD` while keeping `ENTRYPOINT`.

So an image with `ENTRYPOINT ["caddy"]` and `CMD ["run"]` plus
`command: ["version"]` runs `caddy run version`, where Docker would run
`caddy version`. **Set `entrypoint:` when you need the command to be exactly
what you wrote** - it takes the image out of the equation entirely.

[lxc/incus#3765](https://github.com/lxc/incus/pull/3765) is the upstream
proposal to expose the split. If it lands, `command:` on its own will substitute
like Docker does, which will be a breaking change for anyone relying on today's
append.

_Since: v1.2.0_

#### DNS

`dns`, `dns_search`, and `domainname` map to Incus's `oci.dns.*` instance
config keys, which seed the container's initial `/etc/resolv.conf`:

```yaml
services:
  web:
    image: docker.io/nginx:alpine
    dns:
      - 8.8.8.8
      - 1.1.1.1
    dns_search:
      - example.com
    domainname: example.com
```

becomes:

```yaml
config:
  oci.dns.nameservers: 8.8.8.8,1.1.1.1
  oci.dns.search: example.com
  oci.dns.domain: example.com
```

Each key is only set when the corresponding compose field is non-empty. `dns_opt`
has no Incus equivalent and is not mapped.

_Since: v1.1.0_

#### x-incus Instance Extensions

Any Incus instance config key can be set via the `x-incus` extension block on a service definition. Keys are passed verbatim to the Incus instance config on creation.

```yaml
services:
  web:
    image: docker.io/nginx:alpine
    x-incus:
      limits.memory: 512MiB
      limits.cpu: "2"
      security.privileged: "true"
```

Any [Incus instance option](https://linuxcontainers.org/incus/docs/main/reference/instance_options/) is accepted.

#### x-incus-compose Devices

Attach raw Incus devices to a service's instances with the `x-incus-compose.devices`
block. Each named entry is passed to Incus verbatim; the `type` key selects the
device type and is required.

```yaml
services:
  web:
    image: docker.io/nginx:alpine
    x-incus-compose:
      devices:
        gpu0:
          type: gpu
          gputype: physical
          pci: "0000:01:00.0"
        extra-disk:
          type: disk
          source: /dev/sdb
          path: /mnt/data
```

This is an escape hatch for device types incus-compose does not model natively
(`gpu`, `unix-char`, `usb`, ...). Compose-managed devices (`ports`, `volumes`,
`networks`) should use their native keys. Any
[Incus device](https://linuxcontainers.org/incus/docs/main/reference/devices/) is
accepted; keys collide by device name, so a raw device sharing a name with a
compose-managed one overrides it.

_Since 1.0.0-beta.22_

### Projects

```yaml
x-incus:
  limits.cpu: "4"
  limits.memory: 2049MiB # +1 MiB
  limits.virtual-machines: 0

services:
  web:
    image: docker.io/nginx:alpine
    deploy:
      replicas: 4
    x-incus:
      limits.cpu: "1"
      limits.memory: 512MiB
```

Any [Project option](https://linuxcontainers.org/incus/docs/main/reference/projects/) is accepted.

#### x-incus-compose Healthd

Configure the ic-healthd sidecar with the top-level `x-incus-compose.healthd`
extension:

```yaml
x-incus-compose:
  healthd:
    scope: global
    incus: https://:8443
    network: :default
    workers: 128
    restart-workers: 32
    x-incus:
      limits.cpu: 2
      limits.memory: 256MiB

services:
  web:
    image: docker.io/nginx:alpine
```

| Key               | Description                                                                                                                                                                        |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `scope`           | `global` (one shared daemon in the Incus `incus-compose` project, the default) or `project` (a sidecar of this project's own). Loses to a scope the Incus project already carries. |
| `incus`           | The Incus API URL healthd connects to. Defaults to the bridge gateway and the connection's port.                                                                                    |
| `network`         | `<project>:<network>` for a managed network, or a plain bridge name. Defaults to the bridge of the project the daemon runs in.                                                       |
| `workers`         | Health checks the daemon runs at once, over every project it watches. Default 128.                                                                                                    |
| `restart-workers` | Restarts it runs at once, over every project it watches. Default 32.                                                                                                                   |
| `x-incus`         | Raw Incus instance config for the sidecar, e.g. `limits.*`.                                                                                                                            |
| `external`        | Use a healthd you run yourself; incus-compose neither creates nor looks one up.                                                                                                        |

`scope`, `incus` and `network` are also `--healthd-scope`, `--healthd-incus` and
`--healthd-network` on the CLI, which override the compose file. See
[Health Checking - Scope](/healthd#scope-one-daemon-or-one-per-project).

With `scope: global` the daemon is shared, so the first project to bring it up
supplies `incus`, `workers`, `restart-workers` and `x-incus`; a later project
asking for something different is warned and ignored.

When this option is set, incus-compose does not create compose-managed Incus network resources for service network attachments. Instances use the network devices provided by the copied profile instead. Service-level static IP assignments (`ipv4_address` / `ipv6_address`) are not supported in this mode because incus-compose does not create explicit NIC devices.

### Networks

- Bridge networks (Incus default)
- Network isolation between services
- DNS resolution by service name and by instance name
- Extra DNS names per service via `aliases` (see below)
- External networks (pre-existing Incus networks)
- `x-incus` extension - pass any Incus network config key directly (see below)
- Automatic DHCP range configuration on creation (see below)
- Static IP assignment per service via `ipv4_address` / `ipv6_address` (see below)

Not supported:

- Custom network drivers

#### x-incus Network Extensions

Any Incus network config key can be set via the `x-incus` extension block on a network definition. Keys are passed verbatim to the Incus network config on creation.

```yaml
networks:
  backend:
    x-incus:
      ipv4.address: 10.100.0.1/24
      ipv6.address: fd42:abc::1/64
      ipv4.dhcp.ranges: 10.100.0.100-10.100.0.200
```

Any [Incus bridge network option](https://linuxcontainers.org/incus/docs/main/reference/network_bridge/) is accepted.

#### External Networks

Mark a network as `external: true` to attach services to a pre-existing Incus network.
incus-compose will never create or delete an external network.

```yaml
networks:
  shared:
    external: true
```

Set `name:` when the Incus network is not called what the compose file calls it.
A bare value is an Incus network name, taken literally, use it for a bridge you
manage yourself:

```yaml
networks:
  shared:
    external: true
    name: alpha:dns # the "dns" network of the "alpha" compose project
```

The reference goes through the same naming rules the owning
project used, so it keeps resolving after a rename to a hash - `alpha:dns`
becomes `alpha-dns`, and a pair long enough to exceed the interface limit
becomes the same `ic-` hash on both sides. Only the project that declares the
network creates it; everyone else is `external: true`.

**Name resolution** - incus-compose probes the following candidates in order and uses
the first one that exists in Incus:

1. `name:` value - literal, only when it names no project
2. `name:` value - resolved (`{project}-{network}`, or its hash)
3. Compose network name - raw
4. Compose network name - sanitized

If none of the candidates match an existing network, `up` fails with a not-found error.

_Since: v1.2.0_

#### Automatic DHCP Ranges

When a managed bridge network is created, incus-compose automatically configures DHCP ranges if they are not already set:

**IPv4** - The first quarter of the address block is reserved for static assignment. The DHCP range starts at that boundary:

| Subnet | Static range   | DHCP range        |
| ------ | -------------- | ------------------ |
| /24    | `.1-.63`       | `.64-.254`         |
| /16    | `.0.0-.63.255` | `.64.0-.255.254`   |
| /28    | `.1-.3`        | `.4-.14`           |

**IPv6** - The first 256 addresses (`::0-::ff`) are reserved for static; DHCP runs from `::100` to `::ffff`. Stateful DHCPv6 (`ipv6.dhcp.stateful`) is enabled automatically.

Setting `ipv4.dhcp.ranges` or `ipv6.dhcp.ranges` in `x-incus` disables auto-calculation for that protocol. Existing networks (already present in Incus when `up` runs) are never modified.

#### Static IP Assignment

A service can be assigned a fixed IP on a specific network using the standard Compose
`ipv4_address` / `ipv6_address` fields on the per-service network attachment:

> An address without a netmask (e.g. `10.100.0.2` instead of `10.100.0.2/24`) is invalid and
> fails silently.

```yaml
services:
  db:
    image: docker.io/postgres:16-alpine
    networks:
      backend:

  web:
    image: docker.io/nginx:alpine
    depends_on:
      db: service_healthy
    networks:
      backend:
      frontend:
        ipv4_address: 10.100.0.2/24
        ipv6_address: fd42:abc::2/64

networks:
  frontend:
    x-incus:
      ipv4.address: "10.0.0.1/24"
      ipv6.address: "fd42:abc::1/64"

  backend:
    internal: true
```

The address is set as `ipv4.address` / `ipv6.address` on the Incus NIC device. The bridge's
built-in DHCP server reserves it so the instance always receives that address on the network.

The address must fall within the static zone (first quarter of the block) to avoid conflicts
with DHCP-assigned addresses.

Setting `internal: true` on a network disables its gateway by setting `ipv4.gateway` and
`ipv6.gateway` to `none`. This requires Incus 7.3 or later (or the 7.0.2 LTS point release).
Override this per-service with `x-incus-compose.internal: false`.

_`internal: true` since: v1.1.0_

#### Network Aliases

The standard Compose `aliases` field on a service's network attachment registers
extra DNS names for that instance:

```yaml
services:
  db:
    image: docker.io/postgres:16-alpine
    container_name: my-db
    networks:
      default:
        aliases:
          - db.mydomain.lan
```

Each alias becomes a `cname=<alias>,<instance>` record in the network's
`raw.dnsmasq`, resolving straight to the instance, with no DHCP lease to wait for,
unlike the IP-based service-name records described in DNS Resolution below. Aliases on networks shared by multiple
projects (`external: true` / `name:`) coexist without
clobbering each other's records, the same way service-name records do.

> Because a CNAME alias can only point at one target, `aliases` is for
> single-instance services. Declaring it on a service with more than one
> replica registers the same alias against every replica's instance name,
> which dnsmasq does not support (an alias must be unique) and produces
> undefined DNS behavior. Use the service name, which does round-robin, for
> scaled services instead.

_Since: v1.1.0_

### Volumes

- Named volumes (Incus custom storage volumes)
- Bind mounts - pass-through when incusd runs on your machine, or copied in with
  `x-incus-compose.seed` against any server (see below)
- Read-only volumes
- Automatic UID/GID shifting
- tmpfs mounts (with optional size limit)
- `x-incus` extension - pass any Incus volume config key directly (see below)
- `x-incus-compose.pool` - select the storage pool for a named volume (see below)
- `x-incus-compose.seed` - copy a bind mount's source into the instance (see below)

Not supported:

- Volume driver options

#### x-incus Volume Extensions

Any Incus storage volume config key can be set via the `x-incus` extension block on a volume definition. Keys are passed verbatim to the Incus volume config on creation.

```yaml
volumes:
  data:
    x-incus:
      size: 10GiB
      block.filesystem: ext4
```

Any Incus storage volume option is accepted.

The `x-incus` block also works inline on a volume entry, which is the only way to
set options on a **bind mount** (a bind's source is a path, not a named volume):

```yaml
services:
  web:
    volumes:
      - type: bind
        source: ./html
        target: /usr/share/nginx/html
        x-incus:
          security.shifted: "false"
```

An inline `x-incus` block takes precedence over the matching named volume
definition. See Volume Permissions below for `security.shifted`.

#### x-incus-compose Volume Pool

Set `x-incus-compose.pool` on a named volume to place it in a specific Incus storage pool. Without this the client's default storage pool is used.

```yaml
volumes:
  data:
    x-incus-compose:
      pool: fast-ssd

services:
  app:
    image: docker.io/myapp:latest
    volumes:
      - data:/var/lib/app
```

To move an existing volume to a different pool, stop the project, then use `incus storage volume move` via the `incus-compose incus` passthrough:

```bash
incus-compose stop
incus-compose incus storage volume move default/vol-library ext/vol-library
incus-compose start
```

Then update `x-incus-compose.pool` in your compose file and run `incus-compose up --recreate` to reattach.

Volumes are stored with a `vol-` prefix. Long names are hashed, so `my-very-long-volume-name` may become `vol-a1b2c3d4...`. Use `incus storage volume list` to find the actual name before moving.

#### x-incus-compose Volume Seeding

A bind mount is normally passed through: Incus attaches a disk device and
**incusd** resolves the source path on its own filesystem, so the files have to
be on the server. Set `x-incus-compose.seed: true` inline on the volume entry to
**copy** the source instead, which is how a bind mount works against a server
that is not your machine:

```yaml
services:
  web:
    image: docker.io/library/busybox:glibc
    volumes:
      - type: bind
        source: ./html
        target: /www
        read_only: true
        x-incus-compose:
          seed: true
```

The source is read by incus-compose, on the machine you run it from, and must
exist there. What happens next depends on what it is:

- **A directory** becomes a custom storage volume, filled from the directory
  when the volume is **created**. Later changes on your machine do not
  propagate: the volume goes its own way from there, and `up` will not re-seed
  it. Delete the volume to start over.
- **A single file** is pushed into the instance on **every start**, while it is
  still stopped, overwriting what is there. Handy for a config or a key you want
  refreshed each time.

Seeding is a copy, in one direction. Nothing written inside the container comes
back out, so it does not replace a named volume for data you mean to keep.

Seeding is off by default: bind mounts are plain pass-through unless you ask.

_Since: v1.0.0_

### Environment

- `.env` file loading
- `env_file` directive
- Variable interpolation
- Default values: `${VAR:-default}`
- Required variables: `${VAR?error message}`

### Project

- `name` - Project name
- Project isolation (Incus projects)
- Profiles - Compose profiles

### Build

See [Builds](/builds) for supported options, builder selection, and platform handling.

### Health Checks

Supported via the `ic-healthd` sidecar. See [Health Checking](/healthd) for full details,
including config keys, defaults, security model, and `healthd` management commands.

The healthcheck status (`starting`, `healthy`, `unhealthy`) is reported in the `Status` column of
`incus-compose list` and `incus-compose ps` when healthchecks are configured.

### Resource Limits

`deploy.resources` is not mapped. Use `x-incus` to set Incus instance limits directly:

```yaml
services:
  app:
    x-incus:
      limits.cpu: "1"
      limits.memory: 512MiB
```

Any Incus instance config key is accepted.

### Restart Policies

Restart policies map to Incus boot configuration:

| Compose `restart` | Incus Config                                    |
| ------------------ | ------------------------------------------------ |
| `no` (default)     | `boot.autostart=false`                            |
| `always`           | `boot.autostart=true`                             |
| `on-failure`       | `boot.autostart=true`, `boot.autorestart=true`    |
| `unless-stopped`   | Uses last-state behavior (Incus default)          |

```yaml
services:
  app:
    image: docker.io/nginx:alpine
    restart: always
```

Restart enforcement is handled by the ic-healthd sidecar, including
`restart` without a healthcheck - see [Health Checking](/healthd#restart-without-a-test).

### Secrets

- `secrets` - File-based secrets pushed into container at `/run/secrets/{name}`
- `secrets[].file` - Read secret from file
- `secrets[].environment` - Read secret from environment variable
- Service `secrets[].target` - Custom target path
- Service `secrets[].uid` / `secrets[].gid` - File ownership
- Service `secrets[].mode` - File permissions (default: 0400)

### Configs

- `configs` - Config files pushed into the container at `/{name}` by default
- `configs[].file` - Read config from a file
- `configs[].content` - Inline content in the compose file
- `configs[].environment` - Read config from an environment variable
- Service `configs[].target` - Custom target path
- Service `configs[].uid` / `configs[].gid` - File ownership
- Service `configs[].mode` - File permissions (default: `0444`); the writable
  bit is always ignored, per the compose-spec, even if an explicit mode with
  a write bit is set

```yaml
configs:
  app_config:
    file: ./app_config.txt

services:
  app:
    configs:
      - app_config
      - source: app_config
        target: /etc/app/config.txt
        uid: "1000"
        gid: "1000"
        mode: 0o440
```

#### Overwriting Image Files

Configs and secrets are written into the instance before it first starts, and
they replace a file the image already ships at that target. This is how you
override an application's own default config.

Docker achieves the same by mounting over the path, so the image file is only
hidden for the container's lifetime. incus-compose writes into the instance's
root filesystem instead, so the replacement is permanent for that instance -
the original is gone until the instance is recreated.

_Changed in 1.2.0_: a target that already existed in the image was previously
left untouched, which silently ignored the config or secret.

## Not Supported (Yet)

### External Secrets and Configs

`secrets[].external` and `configs[].external` are not supported.

In Docker Swarm, `external: true` means "this secret/config already exists:
don't create it, just reference it by name." incus-compose has no equivalent
standalone "secret" or "config" resource in Incus to reference: it only knows
how to read a `file`, inline `content`, or an `environment` variable and push
the result into a container as a file. Use `file`, `content` (configs only),
or `environment` instead.

### Dockerfile HEALTHCHECK

The `HEALTHCHECK` instruction embedded in Docker images is not read, so declare
`healthcheck.test` explicitly in the compose file.
See [Health Checking](/healthd#dockerfile-healthcheck-not-supported) for the background.

### Extended Features

Not supported:

- `extends` - Service extension
- `deploy` - Most deployment options (except `replicas`)
- `links` - Legacy linking (use networks)
- `external_links` - Cross-project links

## Local vs Remote Incus

> **The Incus server must have `core.https_address` set in all cases**, even for
> a local Unix-socket client. Image caching copies images between Incus projects
> using pull mode, which requires the server to be reachable over the network.
> Without it, `up` fails with `The source server isn't listening on the network`.
> See [Getting Started](/getting-started#incus-must-listen-on-the-network-required).

With that in place, a few behaviors still depend on whether incus-compose talks
to a local Incus over the Unix socket or to a remote daemon over HTTPS:

| Feature       | Local (Unix socket)                                                      | Remote (HTTPS)                                                     |
| -------------- | -------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| Bind mounts   | Supported                                                                | Pass-through only when incusd is the same machine; otherwise seed  |
| Health checks | Auto when `core.https_address` names a host, else set `--healthd-incus` | Auto                                                                |

The line for bind mounts is not the transport, it is which machine holds the
files. A pass-through bind is a disk device whose source **incusd** opens on its
own filesystem, so the path has to be on the server. Over HTTPS to the machine
you are sitting at (a `local-https` remote, say), that is still true and bind
mounts work normally.

Talking to a server somewhere else, incus-compose refuses a pass-through bind
with `not on the same host` rather than handing incusd a path it will not find.
The check compares the remote's address against your own interfaces, so it also
refuses a different machine that happens to have the same directory layout, even
though incusd could have resolved it.

Copy the files across with `x-incus-compose.seed` and none of this
applies: that is what the option is for.

For health checks, ic-healthd reaches Incus over HTTPS. When
`core.https_address` names a host (`10.0.0.5:8443`) that address is used, however
you connected. Only a bare `:8443` falls back to the bridge IP plus the port
incus-compose connected on, which a Unix socket does not have, so there the
endpoint must be set explicitly. See [Network Configuration](/healthd#network-configuration).

## Behavioral Differences

### Images

**Registries:**

Image names work just like Docker: a bare `nginx:alpine` resolves to
`docker.io/library/nginx:alpine`, and an explicit registry prefix
(`ghcr.io/...`) is honored as-is.

`docker.io`, `ghcr.io`, `quay.io`, `mcr.microsoft.com`, `registry.gitlab.com`
and `codeberg.org` need no setup. Any other registry has to be an Incus remote,
and adding one of the six above overrides its built-in address, which is how
you point at a pull-through cache:

```bash
incus remote add --protocol oci registry.example.com https://registry.example.com
incus remote add --protocol oci docker.io https://docker-mirror.example.com
```

```yaml
# Both work, identical to Docker Compose
image: nginx:alpine # resolves to docker.io/library/nginx:alpine
image: ghcr.io/myorg/app:v1 # explicit registry
```

**Global cache:**

Like Docker, images are cached globally. An image pulled for one project is available to all projects. This avoids duplicate downloads.

**Platform selection:**

Docker allows `--platform linux/amd64`. incus-compose uses the host architecture automatically. Multi-arch images select the correct variant.

### Port Publishing

**Docker Compose:**

```yaml
ports:
  - "8080:80" # iptables NAT rule
```

**incus-compose:**

```yaml
ports:
  - "8080:80" # Incus proxy device
```

By default incus-compose uses userspace proxy devices (a Go
process per forwarded connection). For high-throughput services you can opt in to kernel-mode NAT
via a service extension, which installs nftables DNAT rules instead:

```yaml
services:
  web:
    image: docker.io/nginx:alpine
    ports:
      - published: "8081"
        target: "80"
        x-incus-compose:
          nat: true
    networks:
      - frontend
```

`nat: true` requires Incus 7.2 or later (or the 7.0.1 LTS point release) for ARP/NDP-based
instance IP detection. Combining `nat: true` with a static instance IP additionally requires
Incus 7.3 or later (or the 7.0.2 LTS point release).

> **Warning:** with `nat: true`, published ports are not reachable via `localhost`/`127.0.0.1` on
> the host running incus-compose. The nftables DNAT rules only masquerade traffic for the
> hairpin case (an instance reaching itself via its own forwarded address); host-loopback traffic
> keeps its `127.0.0.1` source address, which is dropped or fails to route back. Use the host's
> real (LAN/bridge) address to reach the port, or stick with the default userspace proxy if you
> need `localhost` access to work.

_Since: v1.1.0_

### Network Naming

**Docker Compose:**

```
{project}_{network}  # e.g., myapp_frontend
```

**incus-compose:**

```
{project}-{network}  # e.g., myapp-frontend (if ≤13 chars)
ic-{hash}            # e.g., ic-a1b2c3d4e5 (if >13 chars)
```

Network names are limited to 13 chars for dhclient compatibility.

### Volume Permissions

**Docker Compose:**

- Volumes owned by root by default
- Manual chown often needed

**incus-compose:**

- Volumes automatically shifted to match container's UID/GID
- Reads `oci.uid` and `oci.gid` from image
- Files appear with correct ownership inside container

**Disabling shifting (`security.shifted: "false"`):**

Shifting maps host files to the container's UID/GID so they appear correctly
owned. Set `security.shifted: "false"` via `x-incus` to turn it off, e.g. for a
read-only bind mount you don't want re-owned. Without shifting, the host file
keeps its raw host UID/GID inside the container, which for an unprivileged
container outside the idmap range shows up as `nobody` (65534):

```yaml
services:
  web:
    volumes:
      - type: bind
        source: ./html
        target: /usr/share/nginx/html
        read_only: true
        x-incus:
          security.shifted: "false"
```

For a bind mount this must be set inline on the volume entry (see x-incus Volume Extensions above).

### External Volumes

**Docker Compose:** an external volume must already exist. Compose will
never create it, and never removes it (not even with `down --volumes` /
equivalent), since it doesn't own the volume's lifecycle.

**incus-compose:** every named volume, external or not, goes through the same
get-or-create path: reuse the Incus storage volume if it already exists,
create it if it doesn't. There's no tracking of "this one was pre-existing."
Concretely, that means:

- A typo'd or renamed volume that would fail fast under Docker (volume not
  found) instead silently creates a new, empty volume here.
- `incus-compose down --volumes` deletes every storage volume tracked for the
  project, including ones marked `external: true`, and there's no protection
  against removing a volume you intended to be pre-existing and shared with
  something else.

If you need to reference a real pre-existing Incus storage volume without
risking it being deleted, avoid `down --volumes` for that project, or manage
the volume directly with `incus storage volume` outside of compose.

### Instance Naming

Instances are named `{service}-{index}` where index starts at 1:

```yaml
services:
  web:
    image: docker.io/nginx:alpine
    deploy:
      replicas: 3
```

Creates instances: `web-1`, `web-2`, `web-3`

You can also override replicas via CLI:

```bash
incus-compose up --scale web=5
```

`--scale` applies only to that invocation. Like `docker compose up`, a plain `up`
reconciles each service back to `deploy.replicas` in both directions: it recreates
instances removed by an earlier `--scale` and tears down extras added by one. Use
`--scale` (or edit `deploy.replicas`) to change the persistent count.

### DNS Resolution

After `up`, both the **service name** and the **instance name** resolve inside containers:

```
database    → round-robins across all database instances (A/AAAA records)
database-1  → specific instance (registered by Incus dnsmasq)
```

This matches Docker Compose behavior. No configuration is required: records are
written automatically to the project bridge network's `raw.dnsmasq` and updated
whenever the scale changes.

A service can also register extra DNS names for itself via `aliases`; see Network Aliases above.

**Note:** Setting `raw.dnsmasq` on the bridge disables AppArmor for the dnsmasq
process (not for containers). dnsmasq still runs as an unprivileged user.

### Environment Variables

**Docker Compose:**

```bash
export MY_VAR=value
docker-compose up  # MY_VAR available
```

**incus-compose:**

```bash
export MY_VAR=value
incus-compose up  # MY_VAR NOT available (security)
```

Use `.env` files or `--os-env` flag for docker-compose compatibility. See [Environment Variables](/environment-variables) for the full picture.

### Config Output

`config --format=yaml` is byte-identical to `docker compose config`.
`config --format=json` deliberately is not.

Docker renders JSON straight from the compose model, and compose-go tags every
extension field `json:"-"` - so `docker compose config --format json` silently
drops every `x-` block. incus-compose renders JSON through the YAML
representation instead, which keeps them.

Two consequences of rendering through YAML:

- Fields Docker emits as explicit nulls - `command`, `entrypoint`, and a
  network's empty `ipam` - are omitted here rather than written as `null`/`{}`.
- Object keys are sorted alphabetically rather than following the compose-spec
  field order. JSON objects are unordered, so this only matters if you diff the
  raw text.

Parse the JSON rather than diffing it against `docker compose` output.

_Since: v1.2.0_

## Testing Compatibility

To test if your compose file works:

```bash
# Validate syntax
incus-compose config --quiet

# Show what will be created
incus-compose config

# Try starting
incus-compose up --no-start

# Check what was created
incus-compose list
```

## Reporting Compatibility Issues

If you find a compose feature that should work but doesn't, please report it with:

1. Minimal `compose.yaml` that reproduces the issue
2. Expected behavior (what docker-compose does)
3. Actual behavior (what incus-compose does)
4. Incus version: `incus version`
