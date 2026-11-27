# LBackup Test Concept

This document defines how every milestone of the [LBackup Concept](Concept.md) is tested before it counts as done.
It refines the testing section of the concept into test cases per phase and sets the rules that make every result reproducible.
A phase in this document is one milestone of the concept, so phase 1 tests milestone 1.

## Principles

Every phase is tested in two directions.

| Direction | Question | Passes when |
| --- | --- | --- |
| Verification | Does the code do what the concept specifies? | Every specified behavior is observed with the expected result |
| Falsification | Can the code be made to violate the concept? | Every deliberate attempt to break a guarantee fails |

Both directions are required, because tests written only from the specification confirm the cases the author already thought of,
while backup software fails in the cases nobody thought of, such as a crash between two writes or a malicious agent.
A phase whose falsification tests find nothing is only accepted when mutation testing shows that the tests would have found a planted bug,
because a test suite that cannot fail proves nothing.

Every test names the requirement it checks, either a goal such as G4 or a section of the concept,
so that a change of the concept shows which tests must change with it.
Every test also states its oracle, which is the independent source of the expected result.
An oracle never shares code with the component under test,
because a bug in shared code would produce the same wrong result on both sides and the comparison would pass.
The upstream `restic` and `rustic` CLIs, a stock `rest-server`, the database servers themselves and the test kit described below serve as oracles.

## Reproducibility

A test result is reproducible when the same command on the same commit produces the same verdict on any machine.
The following rules enforce that.

### Pinned environment

- The Rust toolchain is pinned in `rust-toolchain.toml` and all crates in the committed `Cargo.lock`,
  because a floating dependency version changes the code under test without a commit.
- Every container image in the test setups is referenced by digest and never by tag,
  because a tag such as `postgres:16` points to a different image after every patch release.
- VM tests use distribution cloud images that are stored with their SHA-256 checksum and verified before use,
  because a re-published image of the same name would change the test host silently.
- All tests run inside a test runner image that is itself pinned by digest,
  so that CI and a developer machine use identical tool versions for `restic`, `rustic`, `qemu`, `cargo-mutants` and the fuzzers.
- Tests have no network access beyond the local test network,
  because an external service can change or disappear and would make a past result impossible to repeat.
  MinIO, an SFTP container, `rest-server`, Mailpit as SMTP server, ntfy, Gotify, a webhook receiver and Dex as OIDC provider run locally instead.

### Controlled time

The server and the agent read the time only through a `Clock` trait, which the tests replace with a simulated clock.
This is a design requirement for the code and not only a test convenience,
because token lifetimes, certificate expiry, cron schedules across daylight saving changes, retention,
legal hold expiry and protection periods cannot be tested deterministically against the wall clock.
Asynchronous tests use the paused time of `tokio`, so that a 30 minute token lifetime is tested in milliseconds.
External services with their own clock, such as MinIO object lock, are tested with protection periods of a few seconds,
because their time cannot be simulated and a short period keeps the test fast without changing the logic.

### Controlled randomness

All randomness that is not cryptographic, such as start jitter, generated file trees, property tests, fault schedules and fuzzing,
draws from one seed per test run.
The seed is printed at the start of every run and in every failure report,
and setting `LBACKUP_TEST_SEED` replays the run with the same inputs.
Cryptographic randomness such as keys and nonces stays real, because seeding it would test a different code path than production.
Tests therefore compare logical content, such as file manifests and database exports, and never the bytes of a repository.

### Controlled faults

Crashes and errors at a specific point are injected through failpoints of the `fail` crate,
which are compiled in only with the feature `failpoints`.
A failpoint stops the process at a named step, such as after a pack upload and before the index upload,
which is reproducible where killing a process after a random delay is not.
Network faults use a scripted proxy that cuts or delays a connection at a given byte offset or protocol message,
because a timing-based fault depends on the speed of the machine.
A release build test asserts that the shipped binaries contain no failpoint symbols,
because a failpoint in production would be a remote kill switch.

### Flaky tests

A test that passes and fails on the same commit and seed is a bug, either in the code or in the test.
Automatic retries are not used, because a retry hides exactly the race conditions that backup software must not have.
A nightly job runs every suite twice with the same seed and compares the verdicts and the generated reports,
so that hidden nondeterminism is found before it shows up as a flaky test.

### Failure bundles

Every failed test writes a bundle with the seed, the commit, the image digests, the generated compose files, all logs and the test kit reports.
`cargo xtask test-replay <bundle>` runs the failed test again from the bundle,
so a failure from CI is reproduced locally with one command instead of a description.
Bundles contain only test data and the canary secrets described below, so they can be attached to issues without review.

## Test kit

The test kit is a separate crate `lbackup-testkit` in the workspace that provides the oracles and attackers for all phases.
It depends neither on `rustic_core` nor on the server or agent crates,
because it must stay an independent oracle for exactly the behavior those crates implement.

| Tool | Purpose |
| --- | --- |
| Tree generator | Generates a file tree from a seed with hard links, sparse files, long and non-UTF-8 paths, symlinks, extended attributes, unreadable files, FIFOs and empty directories |
| Manifest | Records path, type, mode, owner, timestamps, extended attributes, link targets, hard link groups and SHA-256 of every entry, and diffs two manifests |
| Database workloads | Writes seeded transactional load with a checkable invariant, such as transfers between accounts whose total stays constant |
| Logical database export | Exports every table, collection or keyspace in a canonical sorted form, so that two databases compare by content and not by dump bytes |
| Evil agent | A client with a valid client certificate that speaks the repository API and the control channel and sends whatever a test scripts |
| Evil worker | A process on the worker socket that impersonates workers and sends forged control messages |
| Fault proxy | Cuts, delays or corrupts connections at scripted offsets |
| Canary secrets | Repository passwords, database passwords and tokens with a recognizable marker, so that a leak is found by searching for the marker |
| Ransomware simulator | Encrypts and renames a seeded share of a tree, fast or spread over several runs |

## Test levels

| Level | Runs in | Typical duration |
| --- | --- | --- |
| Unit and property | `cargo test` on the host | Seconds |
| Fuzzing | `cargo fuzz` with committed corpora, a fixed number of iterations in CI and long campaigns nightly | Minutes in CI |
| Mutation | `cargo-mutants` on the safety-critical modules listed per phase | Minutes to hours |
| Integration | Docker Compose setups started by `cargo xtask test <phase>` | Minutes |
| VM | QEMU with a real distribution image, for host mounts, partitions, systemd and LUKS | Minutes |
| Soak | A long run with a seeded fault schedule and the invariants below checked after every event | Hours |

Every crash found by fuzzing and every surviving mutant that is fixed becomes a regression test,
because a fuzzing corpus alone does not guarantee that the same input is tried again.

## Global invariants

The following invariants are checked automatically at the end of every integration, VM and soak test, in addition to the assertions of the test itself.
Checking them everywhere was chosen over dedicated tests, because a violation often appears as a side effect of a test about something else.

| ID | Invariant | Oracle |
| --- | --- | --- |
| I1 | Every snapshot reported as successful restores to content identical to the source manifest taken at backup time | Test kit manifest, upstream `restic restore` |
| I2 | No snapshot disappears except through retention, a re-key or a test step that deletes it explicitly | Snapshot list before and after, retention dry run |
| I3 | Every repository passes `restic check --read-data` | Upstream `restic` |
| I4 | No canary secret appears in any log, metric, API response that does not need it, state volume of an agent or failure bundle | Search for the canary marker |
| I5 | Every action of the test appears in the audit log with actor and role | Audit log query, from phase 5 on |
| I6 | No process, container or sandbox started by the test is left behind | `docker ps`, `virsh list`, process list |

## Phase 1

Workspace, protocol crate, server with SQLite, REST API with OpenAPI, UI skeleton, enrollment with mutual TLS, file targets in `append-only` mode.

### Verification

| ID | Claim | Method and oracle |
| --- | --- | --- |
| P1-V1 | Every protocol message round-trips without loss | Property test that serializes and deserializes generated messages |
| P1-V2 | The OpenAPI document matches the implemented routes | The document is generated from the code and compared with the committed one, and every route of the router appears in it |
| P1-V3 | The UI uses only the REST API | The UI client is generated from the OpenAPI document, and a build check rejects any other `fetch` call |
| P1-V4 | Enrollment completes as specified | Integration test of all seven enrollment steps, asserting that the token is invalidated and the agent connects with mutual TLS |
| P1-V5 | The token lifetime defaults to 30 minutes and is configurable | Simulated clock, enrollment at 29:59 succeeds and at 30:00 fails, repeated with a configured lifetime |
| P1-V6 | Client certificates renew before expiry | Simulated clock across several renewal periods, the connection stays up |
| P1-V7 | A file backup restores identically | Tree generator, backup, restore with upstream `restic`, manifest diff is empty |
| P1-V8 | The repository API follows the restic REST protocol | Differential test that sends the same request sequence to the server and to a stock `rest-server` and compares status codes and bodies |
| P1-V9 | `append-only` allows writes and deletion of own lock files only | Upstream `restic backup` succeeds, own lock removal succeeds |
| P1-V10 | Repositories open with upstream tools | `restic` and `rustic` CLIs list, check and restore every repository, which verifies G6 |
| P1-V11 | The agent container is confined as specified | Inside the container `/host` is read-only, `/proc`, `/sys`, `/dev`, `/run` and the Docker socket are absent under `/host`, and `CapEff` contains only `DAC_READ_SEARCH` |
| P1-V12 | The default exclude list applies | The tree contains every default excluded path, and the snapshot contains none of them |
| P1-V13 | Agent and UI listener are separated | Every UI route returns 404 on the agent listener and every agent route on the UI listener |
| P1-V14 | Repository passwords are stored encrypted | Canary passwords are absent from the SQLite file and present only after decryption with the master key |

### Falsification

| ID | Attack or fault | Expected result |
| --- | --- | --- |
| P1-F1 | Reuse of a used token, an expired token, a token with a changed byte | Every enrollment fails and is logged |
| P1-F2 | Token guessing at a high rate | Rate limiting starts at the configured threshold |
| P1-F3 | Man in the middle with a different server certificate during enrollment | The agent aborts because the fingerprint does not match and sends no token |
| P1-F4 | Client certificate from another CA, self-signed, expired or revoked | The connection is rejected, a revoked agent is disconnected immediately |
| P1-F5 | Evil agent A requests paths of agent B, including `..`, `%2e%2e`, doubled slashes and absolute paths | Every request is rejected and never reaches the files of agent B |
| P1-F6 | Evil agent in `append-only` mode sends `DELETE` on data, index, snapshot, key and config files, on foreign locks and overwrites an existing file with different content | Every request is rejected, I2 and I3 hold |
| P1-F7 | Evil agent uploads a file whose name is not the SHA-256 of its content | Rejected, as defined in the data path section of the concept |
| P1-F8 | Evil agent sends wrong `Content-Length`, a truncated body or an endless body | The request fails, no file appears under its final name |
| P1-F9 | Failpoint kills the agent after every step of a backup | No partial file exists under a final name, the next run succeeds, I1 to I3 hold |
| P1-F10 | Server restart during enrollment and during an upload | Enrollment can be repeated with a new token, the backup is retried and succeeds |
| P1-F11 | Every write attempt below `/host` from inside the agent container | Every attempt fails with `EROFS` or `EACCES` |
| P1-F12 | A symlink in the backed up tree points outside it, such as to `/etc/shadow` | The link is stored as a link and its target is not read |
| P1-F13 | Fuzzing of the protocol decoder, the repository API request parser and the CSR parser | No crash, no panic, no unbounded memory |

Mutation testing covers the mode enforcement, the path resolution of the repository API and the token validation.

## Phase 2

Management and worker agents, scheduling, retention, prune, check, alert rules, anomaly detection, PostgreSQL and MariaDB targets.
The worker cache is tested here, because it arrives with the workers.

### Verification

| ID | Claim | Method and oracle |
| --- | --- | --- |
| P2-V1 | Retention selects the same snapshots as restic | Differential property test against `restic forget --dry-run` with the same snapshot set and policy |
| P2-V2 | Cron schedules honor time zones | Simulated clock across both daylight saving transitions in several time zones, each slot runs exactly once |
| P2-V3 | Jitter stays within its bounds and spreads the start | Seeded jitter for 1000 agents, every start lies in the window |
| P2-V4 | Concurrency limits per agent and globally hold | More due jobs than slots, the number of running jobs never exceeds the limit |
| P2-V5 | The missed-run policy runs once after an outage | Server unreachable for ten slots, exactly one run follows |
| P2-V6 | Prune keeps every referenced blob | After prune, I1 and I3 hold for every remaining snapshot |
| P2-V7 | The `check` rotation reads the whole repository over its period | Read pack names over one rotation period cover every pack |
| P2-V8 | Stale locks are removed and live ones are kept | Lock of a killed agent is removed, lock of a running backup stays |
| P2-V9 | The storage quota is enforced | Upload beyond the quota is rejected and a quota warning is raised |
| P2-V10 | All three topologies work | Backup and restore in single worker, single worker with dedicated management and several workers |
| P2-V11 | A worker receives only its own configuration | The messages captured on the socket contain only the targets and credentials of the receiving worker |
| P2-V12 | Topology migration keeps the identity | Migration from one to two workers and back, with and without keeping the management agent, the certificate fingerprint and the repository stay the same and no enrollment happens |
| P2-V13 | Configuration versions are acknowledged and cached | Config change while the agent runs is acknowledged, and an agent restart while the server is down uses the cached version |
| P2-V14 | PostgreSQL and MariaDB dumps restore with the same logical content | Every supported major version, logical export of source and restored database are identical |
| P2-V15 | Dumps need no temporary file | A dump ten times larger than the `tmpfs` succeeds |
| P2-V16 | Dump verification detects truncation | A dump stream cut by the fault proxy fails verification |
| P2-V17 | Bandwidth limits hold per time window | Measured throughput stays below the limit within a tolerance |
| P2-V18 | Alert rules select by kind, scope and severity | Every combination of rule fields against generated events, deliveries checked in Mailpit, ntfy, Gotify and the webhook receiver |
| P2-V19 | Every anomaly signal and reaction triggers | Ransomware simulator after ten normal runs, each signal fires and each configured reaction takes effect |
| P2-V20 | The cache works and stays bounded | Runs with a deleted and a corrupt cache succeed, the size cap holds, and a warm run downloads no cached index or tree file according to the server request log |

### Falsification

| ID | Attack or fault | Expected result |
| --- | --- | --- |
| P2-F1 | Seeded random snapshot sets with identical timestamps, snapshots in the future, leap days and agent clock skew | Retention still matches restic |
| P2-F2 | Failpoint crash at every step of prune | The repository stays consistent and I1 to I3 hold after the next prune |
| P2-F3 | Backup and prune of the same repository at the same time | The backup succeeds or fails cleanly, and no blob of the new snapshot is pruned |
| P2-F4 | Parallel uploads that each fit the quota but exceed it together | The quota is not exceeded by more than one in-flight file |
| P2-F5 | Evil worker requests the configuration of other targets, uses the name of another worker or an unknown name | Requests are refused and the unknown worker is reported |
| P2-F6 | Two management roles start on one state volume, and two connections use one certificate | The second one fails with the specified error, no job runs twice |
| P2-F7 | Migration stopped after every step of the guide | The guide shows the step as not done, and the host recovers by finishing or reverting the step |
| P2-F8 | Clock jumps forward and backward, server restart at the minute of a scheduled run | No slot runs twice and none is lost except by the missed-run policy |
| P2-F9 | Transactional workload with a constant total during the dump | The restored database satisfies the invariant |
| P2-F10 | Database killed or network cut during a dump | The job is marked failed and the partial snapshot never counts as a good state, because no snapshot is written for a failed stream |
| P2-F11 | Search for database credentials after a job | No canary credential remains in the state volume, the cache or `/proc/<pid>/environ` of the worker |
| P2-F12 | Compromised agent reports low entropy while encrypting | The server-side size and deduplication signal still fires |
| P2-F13 | Slow ransomware that encrypts a small share per run below every threshold | The test records the share at which detection starts, which documents the detection limit instead of claiming none |
| P2-F14 | Legitimate large changes such as a distribution upgrade or log rotation | Only the alert fires, no hold and no pause, as specified for the default reactions |

Mutation testing covers retention, prune selection, quota accounting and worker configuration filtering.

## Phase 3

Snapshot browser, download, one-shot restore agent, database rollback, test agent with scheduled restore tests.

### Verification

| ID | Claim | Method and oracle |
| --- | --- | --- |
| P3-V1 | The browser shows the snapshot tree correctly | Tree of every directory compared with `restic ls` |
| P3-V2 | Snapshot comparison is correct | Diff of two snapshots compared with `restic diff` |
| P3-V3 | Downloads contain exactly the selected entries | `tar.gz` and `zip` with files above 4 GB and non-ASCII names, unpacked and diffed against the manifest |
| P3-V4 | The restore token is bound to one snapshot, one path and a short lifetime | Restore within the bounds succeeds |
| P3-V5 | Restores go to a staging directory by default | Original path unchanged, staging content equals the manifest including ownership and extended attributes |
| P3-V6 | In-place restore needs explicit confirmation | Without the confirmation flag no command for in-place restore is generated |
| P3-V7 | Database rollback restores into a new database | New database with suffix equals the snapshot content, the current database is unchanged |
| P3-V8 | Every restore test check works in both sandboxes | Checksums, database query and custom script in a Docker and a VM sandbox |
| P3-V9 | Sandboxes are isolated and removed | A canary listener on the test network receives no connection, and I6 holds after success and failure |

### Falsification

| ID | Attack or fault | Expected result |
| --- | --- | --- |
| P3-F1 | Restore token used for another snapshot, another path, after expiry and a second time | Every attempt is rejected |
| P3-F2 | Evil agent writes a snapshot into its own repository with entries named `../etc/cron.d/x`, absolute paths, and a symlink followed by a file below it | The restore writes nothing outside the target path, because an `append-only` agent can write such a snapshot today and restore it later |
| P3-F3 | The same crafted snapshot downloaded as `zip` and `tar.gz` | The archive contains no entry that escapes the extraction directory |
| P3-F4 | A byte flipped in a pack, a missing pack, a truncated dump | The restore test fails and raises an alert |
| P3-F5 | Restored data contains a script that tries to reach the network and the host | The canary listener records nothing |
| P3-F6 | Test agent killed during a test | Leftover sandboxes are removed at the next start, I6 holds |
| P3-F7 | Test agent requests a snapshot of an agent not assigned to it, or tries to write | Rejected by the server |
| P3-F8 | Restore agent in `restore` mode requests a second snapshot | Rejected by the server |

Mutation testing covers the restore path sanitizing and the restore token validation.

## Phase 4

MongoDB, Redis, SQLite, configuration presets, Docker volumes, custom commands, custom worker with custom exports.

### Verification

| ID | Claim | Method and oracle |
| --- | --- | --- |
| P4-V1 | MongoDB and Redis restore with the same logical content | Logical export of source and restored instance are identical |
| P4-V2 | SQLite copies database, `-wal` and `-shm` together and checks integrity | Restored copy passes `PRAGMA integrity_check` and equals the logical export |
| P4-V3 | Presets contain their specified paths | Snapshot of each preset contains every listed path that exists on the test host, the package list matches the package manager |
| P4-V4 | Docker volumes are backed up, database volumes are skipped by default | Volume content equals the manifest, a PostgreSQL volume is absent |
| P4-V5 | Custom commands back up their standard output | Snapshot content equals the command output |
| P4-V6 | Custom exports back up the export directory after a successful command | Snapshot equals the directory manifest |
| P4-V7 | Every cleanup mode behaves as specified | `keep`, `delete` and `keep-latest` over several runs, directory content checked after each |

### Falsification

| ID | Attack or fault | Expected result |
| --- | --- | --- |
| P4-F1 | Seeded SQLite writer with checkpoints during the copy | Every successful snapshot passes the integrity check, or the job fails after three attempts, never an inconsistent snapshot |
| P4-F2 | MongoDB replica set under transactional load with oplog enabled | The restored data satisfies the invariant |
| P4-F3 | A command that is not in the signed configuration, sent by a man in the middle or written into the cached configuration of the state volume | The agent refuses to run it |
| P4-F4 | A custom command or export tries to write below `/host` | The write fails |
| P4-F5 | Export command exits non-zero, upload fails at every failpoint | No snapshot counts as good and no cleanup runs |
| P4-F6 | Export command never exits | The process is killed after the configured timeout and the job fails |
| P4-F7 | Export directory contains a symlink to a path outside it | The link is stored as a link, `delete` cleanup does not follow it |

Mutation testing covers the configuration signature check and the cleanup decision.

## Phase 5

Roles with scopes, API tokens, `lbackupctl`, TOTP, OIDC, audit log, metrics, configuration export, recovery kit, legal hold, key rotation, summary reports.

### Verification

| ID | Claim | Method and oracle |
| --- | --- | --- |
| P5-V1 | Every role and separate permission grants exactly its rights per scope | A declarative permission matrix of endpoint, role, scope and separate permission drives the test through the REST API and through `lbackupctl` |
| P5-V2 | No endpoint is missing from the permission matrix | Every route of the OpenAPI document must have a matrix entry, so a new endpoint without one fails the build |
| P5-V3 | TOTP follows RFC 6238 | Test vectors of the RFC with the simulated clock |
| P5-V4 | OIDC login and group mapping work | Dex with test users and groups, resulting role assignments compared with the mapping |
| P5-V5 | API tokens carry role, scope and expiry | Token use within and after expiry, and after revocation |
| P5-V6 | Every action is audited with actor and role | I5 over the full test run |
| P5-V7 | Configuration export and import round-trip | Export, import into an empty server, export again, both TOML files are identical |
| P5-V8 | The recovery kit restores access to all data | Server database deleted, repositories opened with `restic` using only the kit |
| P5-V9 | A legal hold protects snapshots from retention and prune | Held snapshots survive a policy that would remove them, also snapshots created later inside a rule range |
| P5-V10 | Password rotation invalidates the old password | The old password fails with `restic`, the agent continues with the new one |
| P5-V11 | A re-key keeps every snapshot and every hold | Manifests of every snapshot before and after are identical, holds exist in the new repository, `restic` opens it |
| P5-V12 | Summary reports contain the specified content and respect the scope | HTML and PDF text extracted with `pdftotext` compared with the expected runs, tests and anomalies |
| P5-V13 | Metrics expose the documented series | Scrape and compare with the documented list |

### Falsification

| ID | Attack or fault | Expected result |
| --- | --- | --- |
| P5-F1 | An operator assigns roles, a scoped admin creates a global user, an admin grants itself `legal-hold` | Every attempt is rejected and audited |
| P5-F2 | Object IDs of another scope in URLs, query parameters and bodies | Rejected without revealing whether the object exists |
| P5-F3 | Revoked, expired or modified API token, token used to widen its own scope | Rejected |
| P5-F4 | TOTP code replayed within its window | The second use is rejected |
| P5-F5 | OIDC token unsigned, with `alg: none`, wrong audience, wrong issuer or expired | Login fails |
| P5-F6 | Login brute force | Rate limiting starts and a security alert is raised |
| P5-F7 | Failpoint crash at every step of a re-key | The old repository stays complete and usable, and the agent keeps working |
| P5-F8 | Legal hold expiry exactly at a retention run | The snapshot is kept or removed exactly as the expiry defines, never both on two runs |
| P5-F9 | Report for scope A while agents of scope B fail | The report mentions no agent of scope B |
| P5-F10 | Fuzzing of the configuration import | No crash and no partially applied import |
| P5-F11 | A `viewer` browses file content or starts a download | Rejected |

Mutation testing covers the permission check, the scope resolution, the token validation and the hold check in retention.

## Phase 6

Off-site copy with backend protection and heartbeat, peer servers, cold disks, remote primary storage, native agent with install script and systemd unit.

### Verification

| ID | Claim | Method and oracle |
| --- | --- | --- |
| P6-V1 | Off-site copies work in both modes on every backend | Mirror and re-encrypted copy to MinIO, SFTP and an `rclone` local remote |
| P6-V2 | A restore works from the off-site copy alone | Server and primary storage destroyed, restore with `restic` from the copy and the recovery kit, I1 holds |
| P6-V3 | Protection levels are verified on save and before every copy | MinIO with object lock and with versioning, `rest-server` in `--append-only` mode, probe deletion fails in each case |
| P6-V4 | Locks of referenced packs are extended | Short protection period, lock of every referenced pack is extended before half of it remains |
| P6-V5 | The heartbeat follows every successful copy | Request count at the local receiver equals the number of successful copies |
| P6-V6 | Held snapshots carry the S3 legal hold flag | Flag read back from MinIO for every pack of a held snapshot |
| P6-V7 | A peer stores without passwords and delays deletion | Peer database and files contain no canary password, deleted files stay in the trash for the protection period |
| P6-V8 | Cold disks are unlocked, copied and locked on connect | VM with a LUKS loop device attached and detached, the disk is closed after the copy and its repository passes I3 |
| P6-V9 | Remote primary storage behaves like local storage | The mode enforcement and quota tests of phase 1 and 2 run unchanged against a local directory, an NFS share and MinIO |
| P6-V10 | The install script installs and updates the native agent | VM on both architectures, install with pinned key, rerun without token keeps the identity |
| P6-V11 | The native agent is confined | Write attempts outside state and cache directory fail, pseudo filesystems are absent from the snapshot |

### Falsification

| ID | Attack or fault | Expected result |
| --- | --- | --- |
| P6-F1 | The test uses the off-site credentials of the server to delete and overwrite objects, simulating a compromised server | `object-lock` refuses, `versioning` keeps old versions, `append-only-remote` refuses, the peer delays |
| P6-F2 | Misconfigured backends, such as no object lock, governance instead of compliance mode, suspended versioning, credentials that may delete versions | Verification fails and the copy stops with a notification |
| P6-F3 | Simulated time just before and after half of the protection period | No referenced pack is ever without a lock |
| P6-F4 | Prune of a pack whose lock has not expired | No deletion attempt is made |
| P6-F5 | A primary reads or deletes data of another primary on the peer, bypasses the trash or exceeds its quota | Rejected |
| P6-F6 | Cold disk removed during a copy, an unregistered disk, a disk without LUKS | The next copy repairs the first case, the other two are ignored and reported |
| P6-F7 | Man in the middle serves a modified install script or binary, or a wrong checksum | `curl` fails on the pin, or the script aborts on the checksum, nothing is installed |
| P6-F8 | The native agent opens `AF_NETLINK` sockets, writes to `/proc` or loads a module | Blocked by the unit |
| P6-F9 | Network loss to a remote primary backend during an agent upload | The upload fails cleanly and is retried, I1 to I3 hold |

Mutation testing covers protection verification, lock extension and the peer trash logic.

## Phase 7

Hardening review, VM test suite on `x86_64` and `aarch64`, documentation, first release.

### Verification

| ID | Claim | Method and oracle |
| --- | --- | --- |
| P7-V1 | Every test of every phase passes on both architectures | Full regression on `x86_64` and `aarch64` |
| P7-V2 | Every command in the documentation works | Commands extracted from the documentation run in CI |
| P7-V3 | Release builds are reproducible | Two builds from the same commit in clean environments produce identical binaries and image layers |
| P7-V4 | Dependencies have no known vulnerabilities and acceptable licenses | `cargo audit` and `cargo deny` |
| P7-V5 | An agent outside the supported protocol range is rejected | Agents of the previous and an invented future protocol version |

### Falsification

| ID | Attack or fault | Expected result |
| --- | --- | --- |
| P7-F1 | Soak test over 72 hours with a seeded schedule of agent kills, server restarts, network loss, full disk and certificate expiry | I1 to I6 hold after every event |
| P7-F2 | Long fuzzing campaigns of all fuzz targets | No new crash, corpora committed |
| P7-F3 | Hardening review against a checklist derived from G4 and the container and native confinement sections | Every finding becomes a test before the release |
| P7-F4 | Release binaries searched for failpoints, debug endpoints and test certificates | None found |

## Exit criteria

A phase is done when all of the following hold.

- Every verification and falsification test of the phase passes on `x86_64` and `aarch64`.
- Every test of earlier phases still passes, because later features change code that earlier guarantees depend on.
- The global invariants hold for every integration, VM and soak test of the phase.
- Every surviving mutant in the listed modules is either killed by a new test or documented with the reason why it is equivalent.
- The nightly determinism run shows identical verdicts for two runs with the same seed.

## Open points

### Content verification of uploaded files

#### Problem

The concept does not say whether the server checks that the name of an uploaded file is the SHA-256 of its content.
Without the check, an `append-only` agent can upload a file under a name whose content does not match,
which `restic check` later reports as corruption of its own history.
P1-F7 has no defined expected result until this is decided.

#### Options

- Verify the hash of every upload on the server
- Leave verification to `check`

#### Approach

The server verifies every upload, because the hash is computed while streaming at no extra I/O cost,
and a mismatch found during `check` weeks later can no longer be traced to the request that caused it.

### Partial snapshots from interrupted streams

#### Problem

A database dump or custom command is a stream, and when it breaks, `rustic_core` may still write a snapshot of the truncated data.
The concept does not define whether such a snapshot is deleted, tagged or kept, which P2-F10 and P4-F5 need.

#### Options

- The agent writes no snapshot unless the source process exits successfully
- The snapshot is written and tagged as incomplete
- The server forgets incomplete snapshots

#### Approach

The agent writes the snapshot only after the source process has exited successfully and the dump verification passed,
because a truncated dump is not a usable state and the server must not forget snapshots on behalf of an agent.

### Timeout of commands and exports

#### Problem

The concept defines custom commands and exports but no timeout, so a hanging export blocks its worker without end.
P4-F6 has no defined expected result until this is decided.

#### Options

- A timeout per target with a default
- No timeout, with an alert for long-running jobs only

#### Approach

A configurable timeout per target, 6 hours by default, after which the process is killed and the job fails,
because an alert alone leaves the worker blocked and every later job of that worker missed.

### Execution of `aarch64` tests

#### Problem

The exit criteria require every integration and VM test on `aarch64`, which emulated with QEMU is slow enough to make the suite impractical.

#### Options

- Native `aarch64` CI runners
- Emulation with QEMU for all tests
- Native runners for integration and VM tests, emulation only for unit tests

#### Approach

Native `aarch64` runners for integration and VM tests,
because emulation also changes timing and could hide or create the race conditions the falsification tests look for.
