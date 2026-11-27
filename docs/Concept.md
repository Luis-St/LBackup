# LBackup Concept

This document defines the features and the architecture of LBackup,
a self-hosted backup system that moves data from Linux hosts into encrypted repositories on a central server.
It is the starting point for all further specification documents and records the decisions on the open points at the end.

## Goals

- G1 Back up files, configurations and databases of any number of Linux hosts into repositories on one central server.
  The server is not bound to a type of machine,
  because a NAS, a home server, a rented server and a VM in a company network all fill the same role.
- G2 The only network requirement is that an agent can reach the server.
  The server never connects to an agent, because hosts behind NAT or firewalls must work without inbound ports.
- G3 Everything is configured on the server and pushed to the agents at runtime.
  A configuration change must never require logging into an agent host, because that is the main maintenance cost of per-host tools.
- G4 A compromised agent host must not be able to destroy or read the backups of other hosts,
  and by default not its own backup history either.
- G5 The system is safe for production use.
  Every feature ships with an automated test in a local VM or container setup before it is considered usable.
- G6 Backups remain restorable without LBackup.
  Repositories use the restic format, so plain `rustic` or `restic` can open them if the server is lost.
- G7 Backups follow the 3-2-1 rule without a second tool.
  The server copies repositories to an off-site backend, because a single storage location is a single point of failure for fire, theft, hardware failure and ransomware.

## Use cases

LBackup is one general system, and the following deployments are examples of it and not separate modes.
All of them use the same server, agents and repository format,
because a special case in the design would have to be tested and maintained separately.

| Use case | Server | Agents | Off-site backend |
| --- | --- | --- | --- |
| Rented servers to a home NAS | NAS behind a home router, connected as WireGuard client into the VPN of the VPS hosts | VPS hosts that connect to the WireGuard address and port of the NAS | S3-compatible storage |
| Homelab | Home server with a disk array | Other machines and VMs in the same LAN and the server host itself | A second machine at another location over SFTP |
| Small company | VM or dedicated server in the company network | Application and database servers in the same network or in a cloud | S3-compatible storage with object lock |
| Public backup server | Rented server with a public address | Hosts anywhere on the internet that connect directly | Storage of a second provider |
| Single machine | The host itself | One agent on the same host that connects to `localhost` | S3-compatible storage |

A single machine fulfills G7 only together with an off-site backend,
because its local repository shares the hardware with the data it protects.
An agent on the server host itself is an ordinary agent with its own repository,
because a special local path would bypass the mode enforcement of the repository API.

Agents run on Linux only.
Windows and macOS are not supported,
because the confinement through capabilities, read-only mounts and systemd sandboxing has no equivalent there and would need a separate security design.

## Architecture overview

| Component | Runs on | Responsibility |
| --- | --- | --- |
| Server | Any Linux host, Docker | Repository storage, configuration, scheduling, web UI, retention, restore orchestration |
| Worker agent | Linux host, Docker or native systemd service | Reads host data or creates dumps for its assigned targets, deduplicates, encrypts and uploads snapshots, and includes the management role unless it opts out |
| Management agent | Linux host, Docker, optional, at most one per host | Holds the server connection and identity, receives configuration, schedules jobs and relays them to the workers of a host with more than one worker |
| Restore agent | Linux host, Docker or native binary, one-shot | Writes a selected snapshot back to the host, started only on demand |
| Test agent | Dedicated Linux host, Docker or native binary, permanent | Runs scheduled restore tests in sandboxes it creates locally with Docker or libvirt |
| Web UI | Embedded in the server binary | Administration, agent enrollment wizard, browsing and restore, built only on the REST API |
| CLI | Admin machines and CI, static binary | `lbackupctl`, automation of every server function through the REST API |

The server is a single Rust binary based on `tokio` and `axum`.
The web UI is written in SvelteKit with `adapter-static` and embedded into the binary with `rust-embed`,
because a single artifact without a separate web server keeps deployment and versioning simple.
The server keeps its own state in SQLite via `sqlx`, because one installation runs exactly one server instance and an external database would add an operational dependency without benefit.

The server exposes every function through a REST API described with OpenAPI, and the web UI uses only this API,
because every action of the UI is then also available for automation and the API cannot fall behind the UI.
The CLI `lbackupctl` is a separate static binary for `x86_64` and `aarch64`,
because admin machines and CI runners need a small client without the server and its embedded UI.
It authenticates with an API token, which the section on users, roles and permissions defines.

The agent is a separate Rust binary that embeds `rustic_core` as a library.
It runs as worker with embedded management, as worker only or as dedicated management agent,
which the section on management and worker agents defines.
Deduplication, compression and encryption happen on the agent,
because only changed and already encrypted chunks then cross the network and the server never handles plaintext during a backup.

Both binaries live in one Cargo workspace with a shared `protocol` crate,
so that server and agent cannot drift apart in their message definitions.

Server and agent are built for `x86_64` and `aarch64`, as static `musl` binaries and as multi-arch images for `linux/amd64` and `linux/arm64`.
Both architectures are required, because ARM VPS offers, ARM based NAS devices and single-board computers are common and a missing architecture would exclude them entirely.

The server only requires a generic Docker Engine 24 or newer with Compose v2, independent of the machine it runs on.
Targeting a NAS platform such as Synology, TrueNAS or Unraid specifically was rejected,
because vendor features such as ZFS snapshots are not needed by the design and would bind the project to one platform.
Version 24 is the minimum because it is the oldest version still shipped by the current NAS platforms,
which lag behind general Linux distributions and are therefore the most restrictive environment the server runs in.
The agent hosts need Docker 25, which the container hardening section explains.

## Repository layout

Each agent gets its own rustic repository in the primary storage of the server, for example `/data/repos/<agent-id>/`.
A shared repository would deduplicate across hosts, but a single repository key would then give every agent access to every host's data, which contradicts G4.
Targets of one agent are separated inside the repository by snapshot tags and paths.

The primary storage is chosen per installation and is either a local directory or a remote backend.

| Primary storage | Examples | Trade-off |
| --- | --- | --- |
| Local directory | A local disk, the disk array of a NAS, an NFS or SMB share mounted on the host | Fast prune, check and browsing, but the storage must be attached to the server host |
| Remote backend | S3-compatible storage, SFTP or any backend of `rclone`, through the backends of `rustic_core` | No local disk needed, but prune, check and the snapshot browser read every pack over the network |

Both are offered, because a server on a NAS has the disks at hand while a server on a small VM often has none,
and `rustic_core` provides the remote backends without additional code.
A local directory on a network share must support atomic renames,
because the repository API writes every file under a temporary name first, so an interrupted upload never leaves a partial file under its final name.
With a remote backend the server still terminates every repository request and forwards it to the backend,
because the agent must never hold the backend credentials and the server must keep enforcing the agent modes and quotas.
The server caches index and snapshot files of remote repositories in its state directory,
so that browsing and deduplication lookups do not download them on every request.
The off-site copy works the same for both kinds of primary storage,
and a remote primary backend should use a different provider than the off-site backend to fulfill G7.

Repository passwords are generated by the server and stored encrypted in the server database.
The server needs them for retention, verification, browsing and restores.
The agent receives the password of its own repository over the authenticated channel and keeps it in memory only.

## Agent connection and authentication

### Transport

The agent opens a persistent connection to the server over HTTPS and upgrades it to a WebSocket control channel.
Configuration updates, job triggers, status reports and log streams travel over this channel.
A WebSocket was chosen over gRPC because it passes through reverse proxies and the same `axum` stack serves it without a second protocol.
Bulk data uses separate HTTPS requests, so that a large upload never blocks control messages.

The server exposes two listeners.
The agent listener is the only port that needs to be reachable from the agent hosts.
The UI listener can stay restricted to the LAN or a VPN,
because exposing the administration interface to the internet is not required by any agent function.

### Data path

The agent implements the backend traits of `rustic_core` with its own backend that talks to the server's repository API.
The API follows the restic REST server protocol, so the semantics are well understood and a stock `rest-server` can serve as a reference in tests.
A custom backend was chosen over the stock rustic REST backend because it reuses the agent's mutual TLS identity and lets the server enforce per-agent modes on every request.

### Enrollment

1. The administrator starts the agent wizard in the UI and enters a name, the target selection and the agent mode.
2. The server creates a one-time enrollment token with a configurable lifetime, 30 minutes by default.
3. The UI shows a ready-to-run `docker run` command and an equivalent `compose.yaml`,
   both containing the server URL, the token and the fingerprint of the server certificate.
4. The agent generates an Ed25519 key pair on first start and sends a certificate signing request together with the token.
5. The server signs a client certificate from its internal CA and invalidates the token.
6. The agent stores key and certificate in a dedicated writable state volume and connects with mutual TLS from then on.
7. The wizard shows the agent as connected and offers a first test backup.

The certificate fingerprint in the command pins the server identity,
so the first connection cannot be intercepted even when the server uses its self-signed CA.
A server certificate from a public CA is supported as an alternative.
A reverse proxy in front of the agent listener must pass TLS through on layer 4, for example by SNI routing,
because terminating TLS at the proxy would strip the client certificate that authenticates the agent.
Client certificates are short-lived and renewed automatically over the control channel,
because a stolen certificate then expires without manual action.
Agents can be revoked in the UI, which closes the connection and rejects the certificate immediately.

### Management and worker agents

Each target is assigned to a worker whose image covers it,
so a file backup runs in the plain agent image while a PostgreSQL dump runs in the `-postgres` variant.
Separate workers were chosen over one container with every client,
because each worker then gets only the mounts and networks its targets need.
The file worker holds the read-only host root mount but joins no database network,
and a database worker joins the database network but never sees the host filesystem.

Management is the role that enrolls, holds the client certificate and the cached configuration in the state volume,
connects to the server and schedules the jobs of the host.
Every worker includes the management role and enables it by default, and it is disabled with `--no-management`.
A host has exactly one active management role, which lives in one of two places.

| Topology | Containers | Management |
| --- | --- | --- |
| Single worker | One worker | Embedded in the worker |
| Single worker with dedicated management | One worker with `--no-management` and one management agent | Dedicated management agent |
| Several workers | Every worker with `--no-management` and one management agent | Dedicated management agent |

A host with one worker therefore runs a single container,
because a separate management container would add a service without any benefit for the common case.
The dedicated management agent is only required once a host has more than one worker,
because one identity must then serve several containers and none of the workers should become the hub for the others.
An embedded management role serves only its own worker and never accepts other workers,
so the privileged file worker never relays the traffic of database workers.

With a dedicated management agent, workers connect to it over a Unix socket in a shared volume,
and the management agent relays control messages and bulk data to the server.
This keeps one identity, one repository and one configuration view per host,
so the UI shows a host as one agent regardless of how many workers it runs.
Enrolling every worker separately was rejected, because it would split one host into several agents with separate repositories and configurations.
Starting workers on demand from the management agent was rejected, because it requires the Docker socket.

The management agent sends each worker only the configuration and credentials of its own targets,
so a compromised database worker learns nothing about the file targets or other databases.
It accepts only the worker names listed in its configuration and reports a missing or unknown worker to the UI.
Every worker of a host writes into the same repository with its key,
so the limitation on the repository key held by the agent applies per host and not per worker.

The native agent always runs with embedded management and all targets in one process,
because without containers there are no separate images and mounts to split.

### Topology migration

The identity of a host is the state volume and not the container that mounts it.
A change of topology therefore moves the state volume from one container to another,
so the host keeps its certificate, repository and history and never enrolls again.

When a target needs a worker that does not exist yet, the UI starts a migration guide instead of only showing a new command.

1. The guide shows the new `compose.yaml` and a diff against the current one.
2. The existing worker gets `--no-management` and loses its state volume.
3. A dedicated management agent is added and mounts the same state volume the worker used for management, together with a new socket volume.
4. The new worker is added and connects to the socket volume.
5. The administrator runs `docker compose up -d` on the host.
6. The guide waits until the management agent connects with the existing certificate and every expected worker reports in,
   and shows each step as done only after the server has observed it.

The reverse migration is offered when a host drops to one worker.
The guide asks whether to keep the dedicated management agent or to merge management back into the remaining worker.
Keeping it is offered, because it avoids touching a working setup and makes a later second worker a one-line change.
Merging moves the state volume back to the remaining worker and removes `--no-management` and the socket volume.

The state volume must never be active in two management roles at once,
because two connections with one certificate would issue conflicting jobs.
The management role holds a lock file in the state volume, and the server accepts only one connection per certificate,
so a half-applied migration fails with a clear error in the UI instead of running two managers.

### Dynamic configuration

The server holds the full agent configuration and pushes a versioned document whenever it changes.
The agent acknowledges each version, so the UI can show whether a change has been applied.
The agent caches the last configuration in its state volume, so it can still report and retry after a restart while the server is unreachable.
Settings that are fixed at container creation, such as additional bind mounts, Docker networks or the set of workers,
cannot be changed at runtime and the UI shows a regenerated command when one of them changes.

## Agent modes

The mode is set per agent on the server and enforced by the server on every repository request.
Enforcement on the server side is required, because a compromised agent can ignore anything it is told locally.

| Mode | Write new data | Delete or overwrite | Read snapshots back | Typical use |
| --- | --- | --- | --- | --- |
| `append-only` | yes | only own lock files | metadata needed for deduplication | Default for every agent |
| `backup-restore` | yes | only own lock files | yes, file content | Hosts where self-service restore is wanted |
| `read-only` | no | no | yes | Verification or migration hosts |
| `restore` | no | no | yes, limited to one snapshot | One-shot restore agent |
| `restore-test` | no | no | yes, snapshots of the agents assigned to its tests | Test agent |

`append-only` is the default, because a compromised agent host then cannot delete or rewrite its backup history.
Deletion of data only happens on the server through retention and prune, which run locally on the server.
No mode grants an agent the right to forget snapshots or to prune,
because those operations are the main lever of ransomware against backups.

A limitation remains that an `append-only` agent holds its repository key and could therefore decrypt its own history.
This is accepted, because the data originates from that host and an attacker on it can read the current state directly.
The open point on the repository key held by the agent records the rejected alternatives and the mitigation for data the host has since deleted.

## Container hardening of the agent

The agent containers run with the following settings, which the wizard always generates.
A host with a single worker runs one container with embedded management.

```yaml
services:
  lbackup-agent:
    image: ghcr.io/<owner>/lbackup-agent:<version>
    command: ["worker", "--name", "files"]
    restart: unless-stopped
    read_only: true
    user: "0:0"
    cap_drop:
      - ALL
    cap_add:
      - DAC_READ_SEARCH
    security_opt:
      - no-new-privileges:true
    pids_limit: 256
    mem_limit: 1g
    tmpfs:
      - /tmp:size=256m,noexec,nosuid,nodev
    volumes:
      - type: bind
        source: /
        target: /host
        read_only: true
        bind:
          propagation: rprivate
          recursive: disabled
      - lbackup-agent-state:/state
      - lbackup-agent-cache:/cache
    environment:
      LBACKUP_SERVER: https://10.8.0.1:8443
      LBACKUP_SERVER_FINGERPRINT: sha256:0000000000000000000000000000000000000000000000000000000000000000
      LBACKUP_ENROLL_TOKEN: replace-with-token-from-wizard

volumes:
  lbackup-agent-state:
  lbackup-agent-cache:
```

After the migration to a second worker, the same host runs a dedicated management agent,
the file worker without management and a PostgreSQL worker.
The state volume `lbackup-agent-state` moves from the worker to the management agent unchanged.

```yaml
x-lbackup-hardening: &hardening
  restart: unless-stopped
  read_only: true
  cap_drop:
    - ALL
  security_opt:
    - no-new-privileges:true
  pids_limit: 256
  tmpfs:
    - /tmp:size=256m,noexec,nosuid,nodev

services:
  lbackup-manager:
    <<: *hardening
    image: ghcr.io/<owner>/lbackup-agent:<version>
    command: ["manager"]
    user: "0:0"
    mem_limit: 256m
    volumes:
      - lbackup-agent-state:/state
      - lbackup-agent-socket:/socket
    environment:
      LBACKUP_SERVER: https://10.8.0.1:8443
      LBACKUP_SERVER_FINGERPRINT: sha256:0000000000000000000000000000000000000000000000000000000000000000
      LBACKUP_ENROLL_TOKEN: replace-with-token-from-wizard

  lbackup-worker-files:
    <<: *hardening
    image: ghcr.io/<owner>/lbackup-agent:<version>
    command: ["worker", "--name", "files", "--no-management"]
    user: "0:0"
    cap_add:
      - DAC_READ_SEARCH
    mem_limit: 1g
    volumes:
      - type: bind
        source: /
        target: /host
        read_only: true
        bind:
          propagation: rprivate
          recursive: disabled
      - lbackup-agent-socket:/socket:ro
      - lbackup-agent-cache:/cache

  lbackup-worker-postgres:
    <<: *hardening
    image: ghcr.io/<owner>/lbackup-agent:<version>-postgres
    command: ["worker", "--name", "postgres", "--no-management"]
    user: "0:0"
    mem_limit: 1g
    networks:
      - default
      - app-db
    volumes:
      - lbackup-agent-socket:/socket:ro
      - lbackup-worker-postgres-cache:/cache

networks:
  app-db:
    external: true

volumes:
  lbackup-agent-state:
  lbackup-agent-socket:
  lbackup-agent-cache:
  lbackup-worker-postgres-cache:
```

The dedicated management agent and the database workers run without any capability and without a host mount,
because they neither read host files nor need privileges to talk to the server or a database.
Every agent container runs as UID 0, because the state volume moves between containers during a topology migration,
and an unprivileged user would not be able to read key files written by the previous owner without `DAC_OVERRIDE` or `CHOWN`.
UID 0 without capabilities on a read-only root filesystem grants no access beyond ordinary file permissions inside the container.
It remains host root after a container escape, which is accepted because the file worker already runs as UID 0 for the same reason.
The socket volume is mounted read-only into the workers,
because connecting to a Unix socket works on a read-only mount and the workers never need to write there.

Every worker mounts its own cache volume at `/cache`, which holds the rustic cache of index, snapshot and tree files.
Without it every run would download the index and the trees of the parent snapshot again,
which costs time and traffic on metered links and grows with the size of the repository.
The cache holds the repository files exactly as stored on the server, so it contains no plaintext and needs no protection beyond the repository encryption.
It is a separate volume and not part of the state volume,
because it can be deleted at any time without losing the identity of the host and it never moves during a topology migration.
The file worker keeps the volume `lbackup-agent-cache` across the migration, so its first run afterwards finds a warm cache.
A worker that starts with an empty cache only pays one full download of the metadata.
The agent caps the cache at a configurable size, 2 GB by default, and evicts the least recently used files,
because a Docker named volume has no size limit of its own and must not fill the disk of the host.
The management agent has no cache volume, because it never opens the repository.
Placing the cache in the `/tmp` `tmpfs` was rejected, because it is lost on every restart and its 256 MB are too small for large repositories.

`DAC_READ_SEARCH` is the only capability of the file worker, because it lets the agent read every file regardless of permissions and nothing else.
`DAC_OVERRIDE` is not added, because it also grants write access, which a backup agent never needs.
The file worker runs as UID 0 inside the container, because file capabilities on a non-root binary are blocked by `no-new-privileges`.
User namespace remapping is not used for the agent, because capabilities inside a remapped namespace do not apply to host-owned files.

The root filesystem is mounted non-recursively with `recursive: disabled`, which Docker supports since version 25.
Submounts such as `/proc`, `/sys`, `/dev`, `/run`, container overlay mounts and the Docker socket therefore never appear in the container.
Masking them with empty `tmpfs` mounts was rejected, because a missed submount would silently stay visible.
Data that lives on separate filesystems, such as a dedicated `/home` or `/var/lib/postgresql` partition,
is added as explicit additional read-only bind mounts.
The wizard offers a `findmnt` command whose output can be pasted back, so it can propose these mounts instead of the user listing them by hand.
If a mount must be recursive, the wizard uses `recursive: readonly`,
because a plain `read_only` flag on a recursive bind does not make the submounts read-only.

Paths that are visible but must never be backed up are excluded by a built-in default list,
including `/var/lib/docker/overlay2`, `/var/lib/containerd`, `/swapfile`, `/tmp`, `/var/tmp` and `/var/cache`.
Reading them would waste space and produce inconsistent copies of runtime state.

Database dumps need no additional capability, because the agent connects to the database over the network like any client.
Unix sockets of databases live under `/run` and are therefore hidden.
When a database only listens on a socket, the wizard adds a read-only bind mount of exactly that socket directory to the database worker.
Connecting to a socket on a read-only mount works, because the read-only flag does not apply to socket inodes.

The Docker socket is never mounted into the management agent or the file and database workers.
It would grant full root on the host, which contradicts the purpose of the minimal capability set.
Only a custom worker can receive it as an explicit opt-in, as defined in the section on custom exports.

## Native agent

The agent also runs as a native binary under systemd, for hosts without Docker such as small VPS instances or LXC containers.
Configuration, protocol, modes and enrollment are identical to the container agent,
because the native agent is the same binary and only the confinement differs.

### Installation

The wizard offers an install command as an alternative to the `docker run` command.

```bash
curl -fsSk --pinnedpubkey 'sha256//AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=' \
  https://10.8.0.1:8443/install.sh | sudo sh -s -- --token replace-with-token-from-wizard
```

The script is served by the agent listener, so a host that can enroll can also install without reaching GitHub.
`--pinnedpubkey` pins the server key, so the script cannot be replaced in transit even with the self-signed CA,
and `-k` only skips the CA check that the pin replaces.
The script detects `x86_64` or `aarch64`, downloads the matching binary over the same pinned connection,
verifies its SHA-256 checksum against the value served by the server and installs the systemd unit.
Running the same command again updates the agent in place and keeps its state,
so the update command shown in the UI is this command without a token.

### Confinement

The installer generates the following unit:

```ini
[Unit]
Description=LBackup agent
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/lbackup-agent
User=lbackup
Group=lbackup
StateDirectory=lbackup-agent
CacheDirectory=lbackup-agent
AmbientCapabilities=CAP_DAC_READ_SEARCH
CapabilityBoundingSet=CAP_DAC_READ_SEARCH
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
SystemCallFilter=@system-service
MemoryMax=1G
TasksMax=256
Nice=10
IOSchedulingClass=idle
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

The agent runs as the dedicated user `lbackup` with `CAP_DAC_READ_SEARCH` as an ambient capability,
which gives the same read access as the container agent without running as root.
Ambient capabilities survive `NoNewPrivileges`, because they do not depend on file capabilities or setuid.
`ProtectSystem=strict` makes the whole filesystem read-only for the service except its `StateDirectory`,
which replaces the read-only bind mount of the container variant.
`CacheDirectory` creates `/var/cache/lbackup-agent` as the writable cache location, with the same role and size cap as the cache volume of the container agent.
It is kept apart from the `StateDirectory`, because the cache is disposable and the state is not.
A plain root service was rejected, because a compromised agent would then own the host.

The native agent sees `/proc`, `/sys`, `/dev`, `/run` and every other mount, because no mount namespace hides them.
It therefore skips pseudo filesystems by filesystem type, such as `proc`, `sysfs`, `devtmpfs`, `tmpfs`, `cgroup2` and `overlay`,
in addition to the default exclude list.
Matching by type was chosen over a path list, because a path list misses submounts in the same way the rejected `tmpfs` masking does.

Database dumps use the client tools installed on the host,
because the native agent has no image that could ship them.
The agent reports the client versions to the server,
so the UI can warn when a `pg_dump` is older than the PostgreSQL server it targets.

A native restore runs the same binary as a one-shot process with a restore token, started by the administrator with `sudo`.
It receives the write capabilities listed in the restore section only for that process,
so the permanent service never holds them.

## Backup targets

Every target has a name, a schedule, include and exclude rules, optional pre and post steps and a retention policy.

### Files and directories

- Paths relative to the host root, for example `/etc`, `/srv`, `/home`
- Glob excludes, `.gitignore`-style exclude files and exclusion of caches via `CACHEDIR.TAG`
- Option to stay on one filesystem
- Preservation of ownership, permissions, timestamps, symlinks, hard links and readable extended attributes

### Configuration presets

Presets are predefined file targets for common configuration, so that the frequent case needs one click instead of a hand-written path list.

- System: `/etc`, crontabs, systemd units, SSH host keys
- Docker: compose projects with their `.env` files, found by scanning configured roots for `compose.yaml` and `docker-compose.yml`
- Web servers: nginx, Caddy, Traefik, certificate stores
- Package state: list of installed packages, written as a generated file into the snapshot

### Docker volumes

Named volumes are backed up from `/var/lib/docker/volumes/<name>/_data`, which is reachable through the read-only root mount.
Volumes of a database are not backed up this way by default,
because a file copy of a running database is not consistent and the database target exists for that case.

### Databases

Dumps are streamed directly into `rustic_core` as a stdin-style source, so no temporary dump file is written.
This matters because the agent filesystem is read-only and a dump can be larger than the available `tmpfs`.

The client tools ship in image variants per database family.

| Image tag | Contents |
| --- | --- |
| `lbackup-agent:<version>` | Agent only, enough for files, Docker volumes and SQLite |
| `lbackup-agent:<version>-postgres` | `pg_dump` and `pg_dumpall` of every supported PostgreSQL major version |
| `lbackup-agent:<version>-mariadb` | `mariadb-dump`, also used for MySQL |
| `lbackup-agent:<version>-mongodb` | `mongodump` |
| `lbackup-agent:<version>-redis` | `redis-cli` |
| `lbackup-agent:<version>-full` | All clients |
| `lbackup-agent:<version>-custom` | Shell, `curl` and the `docker` CLI for custom exports |

The wizard assigns each target to a worker with the smallest variant that covers it,
and regenerates the compose file when a new target needs a worker that does not exist yet.
Variants were chosen over one large image, because most hosts run one database family and would otherwise pull several hundred MB of unused clients.
The `-full` variant exists for hosts where one worker for all databases is preferred over several containers.
Downloading clients at runtime was rejected, because it needs a writable cache and network access beyond the server.

A database target connects in one of three ways, chosen per target in the wizard.

| Connection | How the agent reaches the database | Trade-off |
| --- | --- | --- |
| Docker network | The database worker joins the internal network of the database | Isolated, but adding a network recreates the worker container |
| Host network | The database worker uses `network_mode: host` and the database publishes a port on `127.0.0.1` | Changes need no new worker container, but every database compose file publishes a port |
| Direct address | The agent connects to any `host:port` reachable from its network, such as a published host port, another machine or a managed database | Needs no change on the agent, but the port must be reachable from the agent network |

All three are offered, because the right choice depends on how the existing services are deployed and none fits every host.
For a direct address on the same host in bridge mode the wizard adds `extra_hosts: host.docker.internal:host-gateway`,
because ports published only on `127.0.0.1` are not reachable from a bridge network and the wizard warns about that case.
The native agent always uses the host network, so only the host network and direct address connections apply to it.

| Database | Method | Notes |
| --- | --- | --- |
| PostgreSQL | `pg_dump` per database in custom format, `pg_dumpall --globals-only` for roles | Client version must be at least the server version, so the image ships several major versions |
| MariaDB and MySQL | `mariadb-dump --single-transaction` | Consistent for InnoDB without locking |
| MongoDB | `mongodump --archive` | Replica set oplog option for point-in-time consistency |
| Redis and Valkey | Fetch the RDB via `redis-cli --rdb` | No access to the Redis data directory needed |
| SQLite | Copy of database, `-wal` and `-shm` files, then integrity check | Works on the read-only mount, see below |

Database credentials are stored encrypted on the server and sent to the agent only for the duration of a job.
Alternatively a target can reference a credentials file on the host, such as a `.pgpass` or an `.env` file, which the agent reads through the read-only mount.
The UI recommends a dedicated read-only database user and shows the SQL to create one, because the agent never needs write access to a database.

SQLite databases are copied together with their `-wal` and `-shm` files in one pass,
because the online backup API needs to create these files and fails on the read-only mount.
The agent checks that all three files have the same modification state before and after the copy,
opens the copy in memory with `PRAGMA integrity_check` and repeats the copy up to three times when the check fails.
`immutable=1` was rejected, because it silently produces an inconsistent copy during writes,
and an export through a custom command remains available for applications that provide one.

Each database target can verify its dump after the upload,
for example by running `pg_restore --list` against the stream, so that a truncated dump is detected immediately and not on the day of a restore.

### Custom commands

A target can run a command whose standard output is backed up, for example an application-specific export.
Commands are defined only on the server and the agent executes nothing that is not part of its signed configuration,
because an arbitrary command channel would turn the backup system into a remote shell.
Commands run inside the worker the target is assigned to and with its restrictions, so they cannot write to the host.

### Custom exports

A custom export integrates an application that brings its own backup tool, such as mailcow or Artifactory.
The target runs a command defined on the server, waits until it exits successfully and then backs up a configurable export directory.
It complements the custom command target,
because tools like the mailcow backup script write several files into a directory instead of one stream to standard output.

Custom exports run in a dedicated custom worker, whose permissions are chosen in the wizard when the worker is created.

| Permission | Example |
| --- | --- |
| Writable bind mount of the export directory | `/srv/backup-export/mailcow` |
| Additional read-only bind mounts | The mailcow installation directory `/opt/mailcow-dockerized` |
| Docker networks or the host network | The network of Artifactory, to call its export API |
| Capabilities | Only what the command needs, none by default |
| Docker socket | Required by the mailcow backup script, which starts its own containers |

The permissions are fixed at container creation like every other mount and network,
because a permission that could be pushed at runtime would let a compromised server widen the access of an agent without anyone touching the host.
The custom worker is separate from the file and database workers,
because only it holds write access and possibly the Docker socket, and the standard workers stay read-only.
The Docker socket is an explicit opt-in in the wizard, and the UI shows a permanent warning that this worker is equivalent to root on its host.
It is allowed despite the rule for the other workers,
because applications like mailcow offer no consistent backup without starting containers,
and a host cron job outside LBackup would bypass scheduling, logging and alerts.

The custom worker uses the image variant `-custom`, which adds a shell, `curl` and the `docker` CLI,
because export commands are usually shell scripts or API calls.
An own image based on this variant is supported for commands that need further tools.

What happens to the export directory after a successful upload is set per target.

| Cleanup | Behavior |
| --- | --- |
| `keep` | The directory stays untouched, and the application manages its own cleanup |
| `delete` | The content is removed after the snapshot was uploaded successfully |
| `keep-latest` | Only the export of the last successful run stays |

The cleanup never runs after a failed upload, because the export would then exist nowhere.

## Scheduling and execution

- Cron expressions per target with a time zone, plus manual triggers from the UI
- Random start jitter per agent, so that many hosts do not hit the server at the same minute
- Concurrency limits per agent and globally on the server
- Retry with backoff when the server is unreachable, and a missed-run policy that runs once after an outage instead of catching up every slot
- Bandwidth limit per agent and time window, because the traffic of rented servers is often metered
- CPU and IO niceness of the agent process, which needs no capability because it only lowers priority
- Live progress and logs streamed to the UI

## Anomaly detection

The server compares each new snapshot with the previous runs of the same target,
because ransomware shows up as an unusual change in a backup before anyone notices it on the host.

| Signal | Source | Detection |
| --- | --- | --- |
| Changed, new and deleted files | Snapshot summary | The count exceeds the rolling average of a configurable number of runs, 10 by default, by a configurable factor |
| Upload size and deduplication | Server, from the written pack files | The uploaded size jumps or the deduplication and compression ratio drops, because encrypted files neither deduplicate nor compress |
| Entropy and extensions | Agent, computed while chunking | Many changed files with high entropy or with extensions known from ransomware |
| Missing paths | Snapshot trees | A configurable share of the previous tree is missing, which indicates mass deletion |

Thresholds are configurable per target with defaults, because a log directory and a configuration directory have very different normal change rates.
The entropy signal is computed by the agent, because only the agent sees file content in plaintext and it reads every changed file anyway.
A compromised agent can falsify that signal,
so the size and deduplication signal is computed by the server from the data actually written.

The reactions are configurable per target.

| Reaction | Effect |
| --- | --- |
| Alert | Raises an alert of the kind anomaly |
| Legal hold | Places a legal hold on the last snapshot before the anomaly, so that the last known good state cannot be removed |
| Pause retention | Stops retention and prune for the agent |
| Pause off-site copy | Stops the off-site copy of the agent, so that the suspicious snapshot does not replace good data off-site |

Only the alert is active by default,
because legitimate large changes such as upgrades or migrations would otherwise create holds and stop maintenance.
Paused actions continue once an administrator acknowledges the anomaly in the UI, and the acknowledgment is written to the audit log.

## Server side repository maintenance

Retention, prune and verification run on the server directly on the repository files, because only the server is allowed to delete data.

- Retention policies with `keep-last`, `keep-hourly`, `keep-daily`, `keep-weekly`, `keep-monthly`, `keep-yearly` and `keep-within` per target
- Scheduled prune with a configurable maximum of unused data
- Scheduled `check`, with a configurable percentage of pack data read per run, so that the whole repository is verified over a rotation period
- Cleanup of stale locks left by crashed agents
- Storage quota per agent, because an `append-only` agent could otherwise fill the server storage
- Repository statistics such as size, deduplication ratio and growth over time

### Legal hold

A legal hold protects snapshots from retention and prune, for example during an investigation or for a retention duty.
A hold covers single snapshots or a rule of agent, target and time range, which also covers snapshots created later in that range.
Each hold has a reason and an optional expiry date, and without one it stays until it is lifted.
Setting and lifting a hold is a separate permission, because it overrides retention and is not part of routine administration.
Held snapshots are also kept in the off-site copy.
With the protection level `object-lock`, the server sets the S3 legal hold flag on the packs they reference,
because a legal hold in object lock has no expiry and survives a compromised server in the same way as the retention period.
Removing single files from old snapshots, for example for a GDPR request, is not offered,
because it conflicts with legal holds and with the off-site protection levels, which deliberately prevent deleting data before the protection period ends.

### Key rotation

Repository keys are rotated on two levels.

| Level | Procedure | Use |
| --- | --- | --- |
| Password rotation | The server adds a new password for the repository key, removes the old one and sends the new password to the agent | Routine, on a configurable interval or manually |
| Re-key | The server creates a new repository with a new master key, copies all snapshots with `rustic copy`, switches the agent to it and deletes the old repository after a configurable period | After a compromise of the host or the server |

Password rotation alone is not enough after a compromise,
because the password only protects the master key, which stays the same and which a compromised agent has held in memory.
The re-key runs on the server and costs the decryption and encryption of the whole repository once.
The old repository is kept for the configurable period, because it stays the reference if the copy turns out to be incomplete.
Legal holds carry over to the snapshots in the new repository.
A mirror off-site copy shares the key of the primary repository, so a re-key also starts a new mirror,
and the old mirror is deleted once its protection period allows it.
A re-encrypted off-site copy has its own key and is not affected by a re-key of the primary repository.
After a compromise of a host, the re-key is combined with revoking its client certificate and enrolling the host again,
because a new key does not help while the attacker still holds a valid identity.
After a compromise of the server, every repository is re-keyed, because the server held every password.

### Off-site copy

The server copies repositories to off-site backends on a schedule, which fulfills G7.
The copy runs on the server and not on the agents, because agents in `append-only` mode must not hold credentials for a second location.

Supported backends are S3-compatible storage, SFTP and every backend of `rclone`,
which `rustic_core` already provides, so no storage backend code has to be written.
A second restic REST server was not included, because `rclone` covers it with its `restic` remote type.

Each off-site target chooses one of two modes.

| Mode | Behavior | Trade-off |
| --- | --- | --- |
| Mirror | Copies the encrypted pack files 1:1 after each prune, same key as the primary repository | Cheap and fast, but the off-site copy has the same retention and the same key |
| Re-encrypted copy | `rustic copy` into a separate repository with its own password and its own retention policy | A leaked key of the primary repository does not open the off-site copy, but every snapshot is decrypted and encrypted again on the server |

Both modes are offered, because the choice weighs server CPU and provider traffic against key separation and differs per installation.
Off-site credentials and the passwords of re-encrypted repositories are stored encrypted like repository passwords and are part of the recovery kit.
Off-site repositories stay in the restic format, so G6 also holds for them.

### Protection against a compromised server

The server holds the off-site credentials, so a compromised server could delete the off-site copy as well.
The protection against that case is enforced by the backend,
because any rule the server enforces itself disappears together with the server.
Each off-site target declares one of the following protection levels.

| Protection | Backend | Mechanism |
| --- | --- | --- |
| `object-lock` | S3-compatible storage with object lock | Every uploaded object gets a retention period in compliance mode, so neither the server nor the account owner can delete it earlier |
| `versioning` | S3-compatible storage with versioning and credentials without the right to delete object versions | A deletion only hides the current version, and a lifecycle rule of the bucket removes old versions after the protection period |
| `append-only-remote` | A restic REST server in `--append-only` mode, reached through the `restic` remote of `rclone` | The remote side rejects every deletion, and pruning happens there by its own administrator |
| `peer-delayed-delete` | A peer LBackup server | The peer moves deleted files into a trash and removes them only after the protection period |
| `none` | Any other backend | No protection, shown as a permanent warning on the dashboard |

The wizard proposes `object-lock` whenever the backend supports it, because it is the only level that also survives a leaked account of the storage provider.
The server verifies a declared level when the target is saved and again before every copy,
by reading the bucket configuration and by trying to delete a probe object, which must fail.
A declared level that fails verification stops the copy and raises a notification,
because a silently unprotected copy is worse than a visible failure.
Snapshots on a remote filesystem, such as ZFS on an SFTP host, count as `none`,
because the server cannot verify them, and the administrator can acknowledge the warning per target.

The protection period is configurable per target and defaults to 30 days.
With `object-lock`, the server extends the retention of every object that is still referenced once less than half of the period remains,
because an expired lock on a pack that a current snapshot still needs would make that snapshot deletable again.
Pruned packs are deleted off-site only after their lock has expired,
because an earlier deletion fails anyway and retrying it every run would only produce errors.
In the re-encrypted mode, prune of the off-site repository keeps every pack younger than the protection period,
using the option of `rustic_core` to keep recent packs, for the same reason.
The storage overhead is the data pruned within one protection period, which is the price of the delay an attacker has to wait out.

A compromised server can still stop copying without deleting anything,
and its own notifications cannot be relied on in that case.
The server therefore calls a configurable heartbeat URL after every successful off-site copy,
so that a monitor outside the server reports when copies stop.

### Peer servers

A second LBackup server can serve as an off-site backend for a primary server.
The peer only stores the encrypted repositories and never receives their passwords,
so it can neither browse nor restore them, and a compromised peer reveals no data.
Every server can be a peer for other servers in addition to serving its own agents,
because two homelabs that back each other up are a common case and a dedicated storage mode would need a second installation.
A primary is enrolled at the peer like an agent, with a one-time token created on the peer and a client certificate,
and reaches the peer over its agent listener.
The copy uses the mirror or the re-encrypted mode like every other off-site backend.

The peer enforces the protection level `peer-delayed-delete`.
The primary may request deletions after a prune,
but the peer moves the files into a trash and removes them only after the protection period, 30 days by default.
Strict append-only was rejected, because the peer holds no password and could never prune, so the copy would grow without end.
The delay is enforced by the peer and not by the primary, so a compromised primary cannot wipe the copy.
The primary verifies the level with the same probe deletion, after which the probe object must stay readable on the peer.
The peer applies a storage quota per primary server, because a primary could otherwise fill the storage of the peer.

### Cold disks

Rotating USB or other disks connected to the server host serve as offline off-site backends,
because a disk stored at another location is out of reach of an attacker and of the provider of a cloud backend.
Disks are registered in the UI by the UUID of their LUKS header and form a set per off-site target.
When a registered disk is connected, the server unlocks and mounts it, runs the copy and unmounts and locks it again,
so that the disk can be unplugged right afterwards.
LUKS is required on every cold disk in addition to the restic encryption,
because a lost disk would otherwise reveal repository structure, sizes and snapshot times.
The server holds the LUKS keys encrypted in its database and in the recovery kit, like every other secret.
Unlocking and mounting needs device access and `CAP_SYS_ADMIN` in the server container,
which the server receives only when cold disks are enabled.
A helper on the host was rejected, because it would add a second component to install and update outside the server container.
The copy mode is mirror or re-encrypted copy like for every other off-site target.
A cold disk has no protection level, because it is protected by being disconnected,
and the dashboard shows the time since the last copy for each disk instead.

## Restore and rollback

Restores are started from the UI and never by the permanently running agent in `append-only` mode.

- Browse snapshots and their file tree in the UI, compare two snapshots
- Download single files, directories or database dumps as `tar.gz` or `zip` directly from the server, which needs no agent at all
- Restore onto the host through a one-shot restore agent

The one-shot restore works like the enrollment wizard.
The UI generates a `docker run` command with a restore token that is bound to one snapshot, one target path and a short lifetime.
The restore container gets a writable bind mount of only the target path and the additional capabilities that writing files with original ownership needs,
which are `CHOWN`, `FOWNER`, `DAC_OVERRIDE` and `SETFCAP`.
Running this as a separate container keeps the permanent agent free of write access and write capabilities,
while the administrator still decides on the host itself that a restore happens.
Restoring into a staging directory next to the original path is the default,
because overwriting live data in place is a decision the administrator confirms explicitly in the wizard.

Database rollbacks restore a dump into a database chosen in the wizard, by default a new database with a suffix,
so that the current state stays available for comparison until the administrator switches over.

## Restore tests

Restore tests prove on a schedule that snapshots can actually be restored,
because `check` only verifies the repository structure and not whether a database or application starts from the restored data.

### Test agent

Restore tests run on a test agent, which is enrolled like any agent with the test flag `--restore-test` and stays connected permanently.
The server assigns it the `restore-test` mode, which lets it read the snapshots of the agents assigned to its tests and never write to a repository.
The test agent runs on a dedicated machine and never on the server host or a production host,
because it creates sandboxes with Docker or libvirt and therefore needs the Docker socket or the libvirt socket, which are root on that machine.
This keeps the server and the backup agents free of these privileges.
The test agent receives the repository password of a tested agent only for the duration of a test and keeps it in memory,
but it can read the data of every agent assigned to it, so its machine must be trusted to the same degree as the server.

| Sandbox | Creation | Use |
| --- | --- | --- |
| Docker | Containers on the test machine, for example a PostgreSQL server of the same major version | Files, databases and containerized applications |
| VM | A libvirt or QEMU VM from a base image configured per test | Restores that need a full system, such as a host with systemd services or separate partitions |

Sandboxes have no network access except to the test agent by default,
because a restored application such as a mail server must never contact production systems or send mail.
Every sandbox is removed after the test, also after a failure, so that restored data does not stay on the test machine.

### Scheduled tests

Each scheduled test chooses an agent, its targets, the snapshot, by default the latest one, the sandbox and the checks that run.

| Check | Verification |
| --- | --- |
| Checksums | Restored file content is compared against the content hashes in the snapshot metadata |
| Database restore and query | The dump is restored into a fresh database server and a configurable query, such as a row count, must return the expected result |
| Custom script | A command defined on the server runs in the sandbox after the restore, such as an HTTP health check of a restarted application |

The checks are chosen per test, because a database target needs a restore and a query while a file target only needs checksums.
Results with duration and failed checks are stored per test, shown on the dashboard and raise alerts.

## Server administration and UI

- Dashboard with last run, next run, status and size per agent and target
- Agent wizard, agent list with version, connection state and applied configuration version
- Target and schedule editor with a dry run that lists what would be backed up
- Users with roles per scope and API tokens, as defined below, passwords hashed with Argon2id
- TOTP as second factor and optional OIDC login for an existing identity provider
- Audit log of every configuration change, restore, download, revocation, legal hold and key rotation,
  where each entry names the acting user, API token, agent or server together with its role or mode at that time
- Alert rules and summary reports, as defined below
- Prometheus metrics endpoint for external monitoring
- Export and import of the full configuration as TOML, so that the setup can be versioned in Git

### Users, roles and permissions

Permissions are granted as a role per scope, where the scope is global, an agent group or a single agent,
because in a company or on a public backup server a team should manage its own hosts without seeing the others.

| Role | Permissions within the scope |
| --- | --- |
| `admin` | Everything except the separate permissions, including users, agents, targets and off-site targets |
| `operator` | Run backups, start restores and restore tests, download files and acknowledge anomalies |
| `viewer` | Dashboard, logs, snapshot lists and reports, without browsing file content or downloads |

| Separate permission | Reason |
| --- | --- |
| `legal-hold` | Setting or lifting a hold overrides retention |
| `key-rotation` | A re-key deletes the old repository and replaces the mirror off-site copy |

The separate permissions are not part of any role, not even `admin`, and are granted per scope,
because both act against the normal life cycle of the data and should be held by few people.
Fixed roles were chosen over freely defined ones, because every combination of permissions needs a test and a fixed set keeps that manageable.
The snapshot browser, downloads, alerts and reports respect the scope, so a user only sees the agents of the scopes assigned.
OIDC groups can be mapped to role assignments, so that membership is maintained in the identity provider.

API tokens carry a role and a scope like a user, have an optional expiry and can be revoked in the UI,
and every use of a token is written to the audit log,
because a token used by a pipeline acts without a person present and must remain traceable.

### Alerts

Alerts are configured as rules, because different people are responsible for different hosts and kinds of events.
Each rule selects the event kinds, a scope of agents, a minimum severity and the recipients.

| Event kind | Examples |
| --- | --- |
| Backup | Failed run, missed run, quota warning |
| Restore test | Failed check, test not run |
| Anomaly | A signal crossed its threshold |
| Off-site | Failed copy, failed protection verification |
| Security | Revoked agent, failed certificate renewal, repeated login failures, finished key rotation |

Alerts are delivered by email, webhook, ntfy and Gotify.
Mail is sent through an SMTP server configured on the server.

### Summary reports

Summary reports are sent by mail daily, weekly or monthly, with recipients and a scope of agents per report.
A report contains successful, failed and missed runs, the latest restore test results, anomalies,
storage growth, quota usage and the status of the off-site copies and cold disks.
It is sent as HTML mail with a PDF attachment, because a PDF can be archived as evidence for audits.
A report can be limited to periods in which something needs attention, because a daily report without findings is quickly ignored.

## Security of the server

- TLS on both listeners, with the internal CA or a provided certificate
- Secrets in the server database encrypted with a master key supplied as a Docker secret,
  so that a copy of the database file alone does not reveal repository passwords
- A recovery kit export containing the repository passwords and the master key, protected by a passphrase,
  because losing the server database must not make the repositories unreadable
- Rate limiting on login and enrollment endpoints
- The server container also runs read-only with all capabilities dropped and only the repository and state directories writable,
  except for device access and `CAP_SYS_ADMIN` when cold disks are enabled

## Testing

Every feature needs an automated test before it counts as done, as required by G5.
The [LBackup Test Concept](TestConcept.md) defines the test cases per milestone and the rules that make them reproducible.

- Unit tests for configuration parsing, mode enforcement, retention calculation and protocol messages
- Integration tests with Docker Compose that start the server, several agents and sample services,
  namely PostgreSQL, MariaDB, MongoDB, Redis and a file tree with special cases like hard links, sparse files, long paths and unreadable permissions
- Round-trip tests that back up, restore into a fresh container and compare checksums of files and the logical content of databases
- Security tests that assert an `append-only` agent receives a rejection on `DELETE` and on overwriting an existing file,
  that the agent cannot write to `/host`, and that `/proc`, `/sys`, `/dev` and the Docker socket are absent inside the container
- Fault injection for network loss during upload, server restart, agent kill, full disk on the server storage and expired certificates
- Compatibility tests that open repositories with the upstream `restic` and `rustic` CLIs, which verifies G6
- VM tests with QEMU and a real distribution image, because a host-root bind mount and separate partitions cannot be reproduced faithfully inside nested containers
- Isolation tests that assert a database worker sees no host files, a file worker joins no database network,
  and the management agent rejects worker names missing from its configuration
- Migration tests from one to two workers and back, with and without keeping the management agent,
  that assert the host keeps its certificate and repository and that two management roles on one state volume are rejected
- VM tests of the native agent that assert it cannot write outside its state and cache directories and skips pseudo filesystems
- Cache tests that assert a run with a deleted or corrupt cache succeeds, the size cap holds,
  and a run with a warm cache downloads no index or tree files that are already cached
- Primary storage tests with a local directory, an NFS share and MinIO as remote backend,
  that assert mode enforcement and quotas behave the same on all three
- Off-site tests against MinIO, an SFTP container and an `rclone` local remote, including a restore from the off-site copy alone
- Off-site protection tests against MinIO with object lock and with versioning and against a `rest-server` in `--append-only` mode,
  that assert the probe deletion fails, a misconfigured bucket stops the copy,
  locks of referenced packs are extended and prune never deletes a locked pack
- Custom export tests with a sample application that asserts the export directory is backed up,
  each cleanup mode behaves as configured and a failed upload never triggers the cleanup
- Restore test tests that run every check in a Docker and a VM sandbox,
  including a deliberately corrupted snapshot that must fail, and assert that the sandbox has no network and is removed afterwards
- Anomaly tests that simulate ransomware by encrypting and renaming a file tree and assert that every signal and each configured reaction triggers
- Alert and report tests against a local SMTP test server that assert rule scopes and the content of the PDF report
- Key rotation tests that assert the old password is rejected after a password rotation,
  and that a re-key keeps every snapshot, carries over legal holds and leaves a repository readable with the upstream `restic` CLI
- Legal hold tests that assert retention and prune never remove a held snapshot and that the S3 legal hold flag is set off-site
- Peer tests with two servers that assert the peer cannot read any data, deletions are delayed by the protection period and the quota holds
- Cold disk tests in a VM with a LUKS loop device that assert the disk is unlocked, copied and locked again on connect
- Permission tests that assert every role and separate permission per scope through the REST API and through `lbackupctl`,
  and that each audit log entry names the actor and its role
- Every integration and VM test runs on `x86_64` and `aarch64`

## License

LBackup is published under the AGPL-3.0,
because the server is a network service and the AGPL keeps modified hosted versions open where the GPL would not.
The license is compatible with `rustic_core`, which is dual licensed under Apache-2.0 and MIT.

## Milestones

1. Workspace, protocol crate, server with SQLite, REST API with OpenAPI, UI skeleton on that API,
   enrollment with mutual TLS, file targets in `append-only` mode
2. Management and worker agents, scheduling, retention, prune, check, alert rules, anomaly detection, PostgreSQL and MariaDB targets
3. Snapshot browser, download, one-shot restore agent, database rollback, test agent with scheduled restore tests
4. MongoDB, Redis, SQLite, configuration presets, Docker volumes, custom commands, custom worker with custom exports
5. Roles with scopes, API tokens, `lbackupctl`, TOTP, OIDC, audit log, metrics, configuration export, recovery kit,
   legal hold, key rotation, summary reports
6. Off-site copy with backend protection and heartbeat, peer servers, cold disks, remote primary storage,
   native agent with install script and systemd unit
7. Hardening review, VM test suite on `x86_64` and `aarch64`, documentation, first release

The order puts the safety-critical path of enrollment and `append-only` writes first,
because every later feature builds on its guarantees.

## Open points

### Reachability of the server

#### Problem

G2 requires every agent host to reach the agent listener of the server,
which can sit behind a home router, inside a company network or on a public address.

#### Options

- Direct exposure of the agent listener, on a public address or through port forwarding
- A VPN such as WireGuard or Tailscale between the agent hosts and the server
- A relay on a public host that the server connects out to

#### Approach

Direct exposure, a reverse proxy and a VPN are all supported, and the deployment decides between them.
The agent always connects to one configured address of the form `https://<host>:<port>`,
where the host is a domain name or an IP address and the port is the externally reachable one.
The server separates this advertised address from its internal listen address and port,
because a port mapping, a forwarded port or a reverse proxy usually makes the two differ.
The wizard puts the advertised address into every generated command, so the server design stays independent of the network path.
The agent listener is designed to be exposed to the internet,
because every endpoint except enrollment and the install script requires a client certificate,
enrollment needs a one-time token and is rate limited, and the install script serves only public data.
A server with a public address, a forwarded port or a reverse proxy with TLS passthrough therefore needs no VPN,
and its domain name and external port form the advertised address.
A VPN is recommended when the server sits behind a router without a stable public address, such as a NAS at home,
because it avoids a dynamic DNS name and keeps the listener off the internet.
The server can join the VPN as a client, for example a NAS that connects with WireGuard into the VPN network of the VPS hosts,
because the server then needs no inbound port on the home router either.
The agent listener then binds to the VPN interface, and the VPN address and port of the server form the advertised address.
The container agent reaches the VPN through the routing of the host,
so the documentation recommends the VPN IP address over names like Tailscale MagicDNS, which containers do not resolve by default.
A relay was rejected, because it is one more component to build, secure and operate, and a VPN covers the same case.
The UI listener is never exposed by the wizard, because no agent function needs it.

### Server platform

#### Problem

The server runs on NAS platforms such as Synology, TrueNAS and Unraid as well as on general Linux distributions and VMs,
and these differ in Docker version and filesystem features.
`recursive: disabled` needs Docker 25 on the agent hosts, but the Docker version of a NAS limits the server container options.

#### Options

- Target a specific NAS platform
- Require only a generic Docker Engine of a minimum version

#### Approach

A generic Docker Engine 24 or newer, as defined in the architecture overview.

### Repository key held by the agent

#### Problem

An `append-only` agent can decrypt its own backup history, because rustic uses a symmetric repository key.

#### Options

- Accept it, since the data originates from the same host
- Asymmetric encryption where agents only hold a public key, which leaves the restic format and breaks G6
- Separate repositories per time window with rotated keys

#### Approach

Accepted, because the data originates from the same host and an attacker who controls the host can read its current state without the repository.
Asymmetric encryption was rejected, because leaving the restic format breaks G6, which weighs more than hiding data from the host it came from.
Separate repositories per time window were rejected,
because every window loses deduplication against the previous ones, each rotation needs a full upload,
and the agent still holds the key of the current window.
The remaining risk is old data that the host has deleted since, such as removed accounts or rotated secrets.
A target that must protect such data from its own host combines a short retention in the primary repository with a long retention in a re-encrypted off-site copy,
because the agent never holds the key of the off-site repository.
After a compromise of the host, a re-key as defined in the section on key rotation replaces the master key the attacker has seen.

### Database client versions in the agent image

#### Problem

`pg_dump` must be at least as new as the PostgreSQL server, and shipping every client makes the image large.

#### Options

- One image with several PostgreSQL major versions and all other clients
- Per-database sidecar images started on demand, which would require the Docker socket and is therefore unsuitable
- Image variants per database family

#### Approach

Image variants per database family, as defined in the database section.
The native agent uses the clients installed on the host.

### Reaching databases in Docker networks

#### Problem

Databases in internal Docker networks are not reachable from the host network.
The agent must join those networks, which is fixed at container creation and cannot be pushed dynamically.

#### Options

- Wizard lists required networks and regenerates the compose file
- Agent uses `network_mode: host` and databases publish a port on `127.0.0.1`
- Agent connects to a direct `host:port` address

#### Approach

All three, chosen per target, as defined in the database section.

### SQLite with WAL on a read-only mount

#### Problem

The SQLite backup API needs to read the `-wal` and `-shm` files and may try to create them, which fails on a read-only mount.

#### Options

- Open with `immutable=1` and accept a possibly inconsistent copy
- Require an application-side export via a custom command
- Copy database, WAL and SHM files together and check integrity afterwards

#### Approach

Copy all three files together and check integrity afterwards, as defined in the database section.

### Agent updates

#### Problem

Agents must stay compatible with the server, and updating them requires access to the host.

#### Options

- Show a version warning and an update command in the UI
- Protocol versioning with a supported range of agent versions

#### Approach

Both, with the server rejecting agents outside the supported range.
Native agents update with the install command, which the UI shows without a token.

### Protection of the off-site copy against a compromised server

#### Problem

The server holds the off-site credentials, so a compromised server can delete the off-site copy as well,
which weakens G7 against ransomware.

#### Options

- S3 object lock or bucket versioning with a retention period, so deleted objects stay recoverable
- Credentials without delete permission, with deletion done manually or by a separate process
- Delayed deletion in mirror mode, which removes pruned packs off-site only after a configurable period
- Accept the risk for the first version

#### Approach

Protection enforced by the backend through object lock, versioning or an append-only remote,
as defined in the section on protection against a compromised server.
Credentials without delete permission are part of the `versioning` level, because without versioning they would also block prune.
Delayed deletion is used only to schedule deletions after a lock expires and not as protection on its own,
because the server enforces it and a compromised server ignores it.
Accepting the risk remains possible per target as level `none` with a permanent warning,
because some backends offer no immutability and a visible unprotected copy is still better than none.

### Event-triggered backups

#### Problem

Some backups should run right before an event on the host, such as an `apt upgrade` or a deploy,
with a selection of targets per event.

#### Options

- A local CLI command that asks the running agent over a socket to run the targets of an event defined on the server
- A trigger through the REST API with a token, for example from a CI pipeline
- A trigger that waits until the backup has finished, so that a failed backup can abort the upgrade

#### Approach

Deferred until after the first release,
because scheduled and manual backups cover the regular case and the trigger builds on the REST API and its tokens, which are only complete in milestone 5.
