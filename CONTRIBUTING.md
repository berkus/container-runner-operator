# Contributing

This branch develops the standalone Ratings Kubernetes sidecar at the repository
root. The legacy machine charm remains on `main`; see the
[README branch warning](README.md#ratings-kubernetes-sidecar-operator). Do not use
this branch to refresh a machine deployment.

Contributions must meet the existing Canonical CLA requirement, checked by the
repository's CLA workflow. Preserve the GPL-3.0 provenance of the migrated charm
and the Apache-2.0 notices for vendored libraries and retained infrastructure;
see [licensing and provenance](README.md#dependencies-provenance-and-tests).

## Development and offline checks

Use Python 3.12, matching Ubuntu 24.04 and CI. From the repository root:

```sh
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements-dev.txt
.venv/bin/python -m pytest -q --cov=charm --cov-report=term-missing --cov-report=xml
.venv/bin/ruff check src tests
.venv/bin/ruff format --check src tests
```

Dependencies require package-index access on first installation. Tests are offline
and use real vendored relation libraries with Juju/Pebble Harness fakes; no Juju
controller, Kubernetes cluster, Docker daemon, or LXD workload is required.
Library checksum and LIBID/API/patch/PYDEPS checks are included in the unit suite.

If you already have tox 4 installed, these equivalent environments install only
the exact `requirements-dev.txt` lock and use `pyproject.toml`:

```sh
tox -e lint             # Ruff lint and formatting checks
tox -e unit             # pytest and coverage (accepts extra pytest arguments)
tox -e format           # apply Ruff fixes and formatting
tox                     # lint and unit, without modifying source
```

Do not edit vendored libraries or refresh them implicitly during tests/builds.
Intentional updates must keep `libraries.lock.json`, upstream provenance, license
notices, and tests consistent. Runtime dependencies are declared in
`requirements.in`; dev dependencies are in `requirements-dev.in`. Regenerate the
corresponding complete locks together for Python 3.12 when intentionally changing
a dependency, using the `uv pip compile` commands recorded in their headers.

## Build the standalone charm

With Charmcraft 4.x (source validated with 4.5.0), run from this repository root:

```sh
charmcraft pack --platform amd64
```

The result is `ratings_amd64.charm`. `charmcraft.yaml` supplies all metadata,
configuration, actions, and the strict, locked runtime requirements. There is no
separate root `metadata.yaml` or `config.yaml`. The `ratings-image` OCI resource
is supplied separately by the deployment owner; packing does not build the rock.

Normal local packing uses Charmcraft's isolated build environment. The CI pack
job uses destructive mode only on a disposable Ubuntu 24.04 runner; it may install
build packages there. Neither path deploys or publishes the charm.

Application builds, Multipass lab recipes, registry handling, and rollout belong
to the external [application repository](https://github.com/ubuntu/app-center-ratings).
Its [lab guide](https://github.com/ubuntu/app-center-ratings/blob/main/doc/charming.md)
documents the sibling checkout and `just operator-mount` setup used by the local
recipes. This repository deliberately has no machine functional/integration deploy workflow.
