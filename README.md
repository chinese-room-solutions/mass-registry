# mass-registry

The package index for MASS runtime gateways (`.mass`) and workers. A single
hand-maintained `index.yml` at the repo root, fetched raw off the default
branch. Artifacts live on GitHub Releases in each package's own repo, pinned by
sha256. MASS reads this index to search, install runtimes, and mint worker join
commands.

## Schema

`index.yml`:

- `schema_version` — integer. Currently `1`.
- `packages` — list. Each entry:
  - `name` — package repo name (e.g. `mass-runtime-gateway-llama-cpp`). Unique.
  - `kind` — `runtime` or `worker`.
  - `runtime_name` — the join key. Workers map n:1 onto the runtime with the
    same `runtime_name` (e.g. both llama-cpp packages use `llama-cpp`).
  - `display_name` — human label. For workers, do not repeat the word "worker"
    (UIs already label the field, so "llama.cpp worker" renders as
    "Worker: llama.cpp worker") — use the bare product name, e.g. `llama.cpp`.
  - `description` — free text.
  - `versions` — list, one entry per released semver:
    - `version` — semver string (e.g. `0.1.0`).
    - `mass` — Semver range of MASS server versions this version works with
      (e.g. `">=0.1"`). Required on runtimes; optional on workers (a worker talks
      to the MASS server directly via the hub protocol) where an empty value
      means unconstrained.
    - `runtime` — **worker only.** Semver range of *runtime* versions whose
      payloads this worker decodes (e.g. `">=0.1 <0.2"`).
    - `artifacts` — map, keyed by platform. Each value is `{url, sha256}`:
      - runtime key format: `os/arch` (e.g. `linux/amd64`).
      - worker key format: `os/arch/backend` (e.g. `linux/amd64/vulkan`).
      - `url` — GitHub Release asset download URL.
      - `sha256` — hex digest of the asset. `TBD` is a placeholder for
        unreleased assets (Phase 4 replaces every `TBD` with a real digest).

Only list artifacts the package's release workflow builds today. The worker's
registry artifact is the raw self-contained installer binary
(`mass-worker-setup_<os>_<arch>`) — the join bootstrap fetches and execs it
directly. The AppImage/.app containers on the same release are for manual
double-click installs and are not indexed; the worker-binary `.tar.gz` is not
an installer.

## Compatibility

One-directional — resolution is lookup, never constraint solving:

- Workers declare `runtime`, a range over **runtime** versions they decode, and
  may declare `mass`, a range over **MASS** versions they speak the hub protocol
  to (empty = unconstrained).
- Runtimes declare `mass`, a range over **MASS** versions they require.

## Resolution

To resolve a package for a platform: pick the newest `version` whose range
covers the relevant installed version and that has an `artifacts` entry for the
requested platform key. For a worker, `runtime` is matched against the installed
runtime version and `mass` (when set) against the MASS server version — both must
cover. If no version covers every applicable range and has a matching artifact,
resolution fails.

Ranges in the index are trusted to be well-formed (validated at publish time):
an unparseable range is a resolution **error**, not a version to skip. A
malformed *installed* version supplied by the resolving client is an input
error, distinct from "no version covers it".

## Publishing

1. Tag a release in the package repo (`git tag vX.Y.Z && git push --tags`). Its
   release workflow builds and uploads the assets under stable, versionless
   basenames.
2. Open a PR here adding or updating the package's `versions` entry: the new
   `version`, its ranges (`runtime` and/or `mass`), and one `artifacts` entry
   per built platform with the release download URL and the asset's real
   `sha256` (`curl -L <url> | sha256sum`).
