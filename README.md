# Ratings Kubernetes sidecar operator

> **Branch warning:** `feat/ps7` replaces this repository's root machine charm
> with the Ratings Kubernetes sidecar. The legacy Docker/Watchtower machine charm
> remains on [`main`](https://github.com/canonical/container-runner-operator/tree/main).
> This is not an in-place machine-to-Kubernetes upgrade; do not refresh an existing
> machine deployment with this branch's artifact.

A standalone sidecar operator for the existing
[App Center Ratings](https://github.com/ubuntu/app-center-ratings) rock
(`/bin/ratings`, Pebble service `ratings`). All charm code, vendored libraries,
dependency locks, and tests are self-contained at this repository's root. The
charm does **not** deploy PostgreSQL, change the rock, create an ingress, or install
observability agents. It is specific to Ratings, not a generic container runner.
The same charm artifact is intended for local development and Prodstacks 7 (PS7)
Juju deployments on Kubernetes; environment-specific configuration, database offers,
and OCI resources are supplied at deployment time. See the TLS and ingress
limitations below before exposing the service.

`charmcraft.yaml` is the sole source of charm metadata, configuration, and actions;
there are no separate `metadata.yaml`, `config.yaml`, or `actions.yaml` files.

## Requirements and packing

- Juju **3.6.4+**, including the lab's **3.6.29**. Check start/stop support requires
  3.6.4; the charm declares that minimum in `assumes`.
- Charmcraft **4.5.0**, Ubuntu **24.04**, Python **3.12**, Kubernetes sidecars.
- The declared build/deployment platform is **amd64** only. An arm64 rock and
  additional platform declaration would be needed for arm64.
- A compatible `postgresql_client` provider. The local lab uses PostgreSQL-k8s
  **16/stable** via its modern `database` endpoint; PS7 uses an approved DBaaS offer.

Build from this repository's root, independently of the application checkout:

```sh
charmcraft pack --platform amd64
```

This produces **`ratings_amd64.charm`**. Repack after changing the charm. Normal
packing uses Charmcraft's isolated build environment; it does not deploy or
publish anything. CI packs on a disposable Ubuntu 24.04 runner using the
Charmcraft `4.x/stable` channel and `--destructive-mode`, without machine-charm
integration tests. Destructive mode can install build packages on that runner;
it is not a command to run on your everyday host.

The image is **not** included in the charm archive. Deployment must supply the
`ratings-image` OCI resource: the registry reference of the already-built rock
that is accessible to the Kubernetes nodes. Registry/import/rollout tooling is
deliberately outside this operator repository's scope.

The Multipass lab and `just vm charm-pack`, `just vm charm-test`, and
`just vm rollout` recipes belong to the separate
[application repository](https://github.com/ubuntu/app-center-ratings), not this
repository. Those recipes now use a sibling standalone operator checkout instead
of the former application-local `charm/` directory. Set host `RATINGS_OPERATOR_REPO`
if the checkout is elsewhere; `just create` mounts both repositories, while
`just up` followed by `just operator-mount` attaches the operator to an existing
lab. No recipe switches this checkout's branch. See the application's
[lab guide](https://github.com/ubuntu/app-center-ratings/blob/main/doc/charming.md)
for registry setup, dependency models, host forwarding, and smoke checks. Rollout
can publish images and deploy/refresh applications; it is not part of this
repository's build/test workflow. Keep deployment artifacts outside hidden
folders because strictly confined Juju cannot read archives under `.cache`.

## Endpoints and configuration

| Endpoint | Role / interface | Purpose |
| --- | --- | --- |
| `database` | requires / `postgresql_client`, limit 1 | The configured SQL database (`ratings` by default) and relation-scoped credentials |
| `logging` | optional requires / `loki_push_api` | Native Pebble workload stdout/stderr forwarding to COS Loki |
| `ratings-peers` | peer / `ratings_peers` | Shared JWT secret reference and rotation notification |

After deployment by the rollout owner, the database relation is:

```sh
juju integrate -m ratings-k8s:ratings-dev \
  ratings:database postgresql:database
```

The lab's PostgreSQL application is named `postgresql`. No admin password,
extra roles, or superuser privileges are requested. In the lab, the provider
creates the database/user; the app owns its schema.

PS7 staging instead uses an existing **DBaaS-managed SQL database**,
`stg-app-center-rating`. Set `database-name` before relating the application and
use the approved staging provider/offer. The production SQL database is
`prod-app-center-rating`, recorded for inventory/export only; it is not the
staging connection target. SQL names are not Juju offer URLs or DBaaS service
identifiers. The managed provider must expose a compatible `postgresql_client`
relation and authorise this application to use that database; confirm credentials,
TLS, network access, and migration permissions with the DBaaS owners. This charm
does not provision a DBaaS instance or support arbitrary static connection secrets.

For COS in the separate `cos-lite` model, offer the Loki provider's `logging`
endpoint from that model and integrate/consume that offer at **`ratings:logging`**
in `ratings-dev`. Use the actual Loki application and offer URL from that model.
No metrics or tracing endpoint is advertised: Ratings has no implemented metrics
or trace integration here. The Canonical log-forwarding library adds Juju model,
application, unit, and charm labels and disables targets when the relation goes
away. It uses Pebble's existing Loki support, not a Promtail binary or sidecar.
Private-CA Loki HTTPS is not configured by this charm; use an HTTP lab endpoint
or a certificate trusted by Pebble's container trust store.

Config options:

- `database-name`: defaults to `ratings` to preserve the local lab. Requests this
  SQL database through the relation, verifies the provider's returned name, and
  uses it in `APP_POSTGRES_URI`. Supports 1-63 ASCII letters, digits, underscores,
  or hyphens, with no leading hyphen. PS7 staging sets `stg-app-center-rating`.
  Set it at initial deployment, before creating the database relation. Changing
  it on an existing relation **blocks and stops the workload without retargeting
  the relation**; restore the previous value to recover. A database move needs a
  separately reviewed data/credential plan, not an ordinary config toggle. Do not
  remove/recreate a managed database relation without checking its data lifecycle.
- `snap-store-url`: defaults to `https://api.snapcraft.io/v2/`; must be an absolute
  HTTP(S) URL ending in `/`, with no credentials, query, or fragment. Requests
  from the service go to that URL: the default requires network access to the
  public Snap Store.
- `rust-log`: defaults to `info,sqlx=warn,hyper=warn,reqwest=warn`, passed as
  `RUST_LOG`. Non-empty single-line Rust EnvFilter directives are supported.
  Arbitrary EnvFilter grammar is interpreted by Rust, not reimplemented in
  Python. Avoid verbose SQL/client debug filters: workload logs may then contain
  sensitive request data. Those logs are shared with COS when related.

The service receives:

```text
APP_HOST=0.0.0.0
APP_PORT=8080
APP_POSTGRES_URI=<relation-generated, percent-encoded SQLx URI>
APP_JWT_SECRET=<application-owned Juju secret>
APP_SNAPCRAFT_IO_URI=<snap-store-url>
RUST_LOG=<rust-log>
```

### TLS scope

The Ratings **gRPC listener is currently plaintext in all environments**, including
PS7. `APP_TLS_KEY_PATH` and `APP_TLS_KEYCHAIN_PATH` are left unset. Port **8080/TCP** is opened using
`unit.set_ports(8080)`; that does not create external ingress or imply TLS.
Do not expose this deployment on an untrusted network. Production listener TLS,
certificate relations, ingress, and access policy are not implemented here.

Database TLS is handled separately. With `tls=False` (or no TLS flag), the SQLx
URI uses `sslmode=disable`, matching the lab's default. With `tls=True`, the charm
requires the provider's `tls-ca`, writes it to
`/etc/ratings/postgresql-ca.crt`, and uses `sslmode=verify-full` plus
`sslrootcert`. The provider endpoint hostname must match the certificate. Missing
CA data blocks the service rather than silently disabling verification. CA
rotation restarts the app even when the URI is unchanged. Credentials and TLS
fields are read through Canonical `data_interfaces`, including Juju secrets
where negotiated; passwords/usernames are percent-encoded independently.

### JWT ownership and rotation

The leader generates 48 cryptographically random bytes and encodes them with
**standard Base64** (`base64.b64encode`, not URL-safe Base64), matching the
`from_base64_secret` decoders in the application's
[`src/jwt.rs`](https://github.com/ubuntu/app-center-ratings/blob/main/src/jwt.rs). The signing key is stored once in an
**application-owned Juju secret**, labelled `ratings-jwt`. Rotation uses the same
standard Base64 encoding. Only its ID and a non-secret generation marker are
published in the peer application databag.
All units of the application can access the same secret; no per-unit keys and no
plaintext peer credentials are used. Leader changes, charm upgrades, and ordinary
configuration hooks reuse it. The leader also recovers the existing labelled
secret if publication was interrupted. A missing *published* secret is never
silently regenerated.

Run the action on the **current leader** to rotate deliberately:

```sh
juju run -m ratings-k8s:ratings-dev ratings/0 rotate-jwt-secret
```

Replace `ratings/0` with the leader if necessary. The action returns only
`rotated: true`, not the secret. It updates the existing secret and peer generation
marker; the leader reconciles immediately and other units restart as peer changes
propagate. Application secret owners do not receive Juju `secret-changed`, so
peer notification is essential. Units use Ops `Secret.peek_content()` to read the
latest revision without the owner-invalid `refresh=True`. Relevant external
`secret-changed` events and database secret rotations are also handled.

**Rotation invalidates existing JWTs**, and units may temporarily use different
revisions while hooks propagate. There is no dual-key grace period. Accidental
secret deletion stops the workload; restoring secret availability or an explicit
operator-directed recovery is required. Database/JWT values necessarily enter the
workload's Pebble environment, which must be treated as sensitive. Do not share
unredacted `pebble plan`, environment dumps, or debug logs. The charm neither logs
credentials nor copies them into stored state or relation databags.

## Lifecycle and migrations

Before database credentials and the shared JWT are available, the charm disables
and stops even the image's default startup-enabled service. Invalid configuration
blocks; pending database data, unavailable secrets, container connectivity, or
readiness waits. Database relation removal stops Ratings, disables its check,
erases the stale service environment/CA file, and closes the port. If Pebble is
unreachable at removal, the port is closed immediately and stopping is retried
once it is reachable. Database restoration reconfigures and starts the service.

A `ready`-level **TCP Pebble check** probes `127.0.0.1:8080` every five seconds,
with a two-second timeout and failure threshold one. A unit becomes Active only
when Pebble reports the service running and its check up. If an UP check has
zero successes or no success count, the charm also confirms the listener with a
bounded two-second TCP connection from the same pod before setting Active. A
failed or timed-out connection keeps the unit Waiting; missing, DOWN, or inactive
checks are never accepted as ready. This permits Active during the initial
post-replan health assessment without waiting for `update-status`: the first
success of an initially UP check does not emit a check-recovered event. There are
no sleeps or probe retry loops. Checks are reset when a service is restarted so
successes from the previous process cannot establish readiness for the new one.
Check-failed/recovered events and `update-status` update health; a check-failed
event reconciles without restarting an unchanged, still-running service. Pebble
restarts crashed services with backoff. Unchanged configuration does not restart
Ratings.
This is listener readiness, **not** a continuous DB transaction or end-to-end
Snap Store health test. DB/Snap Store outages after startup can affect requests
without bringing the TCP check down.

In the existing application source,
[`src/main.rs`](https://github.com/ubuntu/app-center-ratings/blob/main/src/main.rs)
calls `check_db_conn()` before `grpc::run_server()`, and
[`src/db/mod.rs`](https://github.com/ubuntu/app-center-ratings/blob/main/src/db/mod.rs)
executes
`sqlx::migrate!("sql/migrations").run(&pool)` while initialising the pool. The rock
therefore embeds migrations and runs them before opening the listener. In the
locked **SQLx 0.8.6**, `sqlx-core/src/migrate/migrator.rs` enables locking by default
and calls `conn.lock()`/`unlock()`; `sqlx-postgres/src/migrate.rs` uses
`pg_advisory_lock`/`pg_advisory_unlock` with a database-derived lock ID. Concurrent
units serialize migrations through PostgreSQL itself. **No explicit charm
migration leader, migration Job, or extra PostgreSQL Python driver is necessary.**

## Dependencies, provenance, and tests

Unmodified, tracked Canonical libraries are pinned by commit, `LIBID`, API/patch,
and SHA-256 in `libraries.lock.json`:

- `charms.data_platform_libs.v0.data_interfaces`, **API 0 / patch 62**, commit
  `87e246a5392e9d3dd5c084a824a83c4bc1c34b77`; `PYDEPS`: `ops>=2.0.0`.
- `charms.loki_k8s.v1.loki_push_api`, **API 1 / patch 35**, commit
  `dbc2e7d78fcde4be11ae5328e7c882904befadf3`; `PYDEPS`: `cosl`.

These pinned versions have no other mandatory charm-library imports. The Loki
library imports `cosl` for topology/rules helpers (the library's older prose about
fetching `juju_topology` does not match these current imports). Optional
`cryptography`/`psycopg` code paths in `data_interfaces` are not needed for this
same-model PostgreSQL consumer; it performs no direct database operations.
The sidecar code and tests were copied from `ubuntu/app-center-ratings/charm`
without modification, including its **GPL-3.0** [`LICENSE`](LICENSE). This migration
does not relicense that code as Apache. Apache-2.0 licensing for vendored Canonical
libraries is preserved in their headers, [`lib/LICENSE.loki-k8s`](lib/LICENSE.loki-k8s),
and [`LICENSE.apache-2.0`](LICENSE.apache-2.0). The latter also preserves the original
repository's Copyright 2023 Canonical license notice for retained repository
infrastructure. The legacy machine implementation remains on `main` under its
original license. The repository's Canonical CLA policy still applies; see
[CONTRIBUTING.md](CONTRIBUTING.md).

`requirements.in` pins **Ops 2.23.2** and **cosl 1.11.3**. `requirements.txt` pins
all 14 runtime packages, including cosl's pydantic, diskcache, tenacity, PyYAML,
and typing dependencies and Ops' websocket/import metadata/OpenTelemetry API
dependencies. The latter is an Ops dependency, not a tracing integration.
Charmcraft uses strict dependency resolution; `pydantic-core` is binary-installed.
Development tools and their closure are separately locked in
`requirements-dev.txt`.

From this repository's root (Python 3.12 is the build/CI baseline):

```sh
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements-dev.txt
.venv/bin/python -m pytest -q --cov=charm --cov-report=term-missing
.venv/bin/ruff check src tests
.venv/bin/ruff format --check src tests
```

The migrated suite contains **110 unit tests** and targets **99% charm-code
coverage**. It includes immutable database-name guards before the relation
library's eager writes, standard-Base64 JWT creation/rotation, zero/unknown
success counts with successful, refused, and timed-out TCP probes, non-UP checks,
check events without service restarts, and readiness reset after restart.
CI runs these offline tests, Ruff lint/format checks, and packs the charm; it does
not deploy, run Docker/LXD workload tests, or publish artifacts to a registry or
Charmhub. Optional `tox` commands use the same locked dependencies and
`pyproject.toml` configuration; see [CONTRIBUTING.md](CONTRIBUTING.md).

The Harness tests run **offline after dependency installation**, using the real
vendored libraries and fake Juju/Pebble/database data. They cover provisioning,
secret-backed and plaintext credentials, URI encoding, TLS/CA rotation, stable
leader/follower JWT ownership and rotation, invalid configuration, readiness,
ports, stop/recovery, disconnected containers, safe actionable failures, and
real Loki target/label/removal behaviour. The vendored checksums are also tested.
Upstream `JujuVersion.from_environ()` deprecation warnings are expected with this
Ops version; vendored libraries are not modified to suppress them.
