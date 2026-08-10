# mass-registry

The package index for the MASS family: runtime gateways (`.mass`), workers,
and UI themes. A single hand-maintained `index.yml` at the repo root, fetched
raw off the default branch. Artifacts live on GitHub Releases in each package's
own repo, pinned by sha256. MASS reads this index to search, install runtimes,
and mint worker join commands; Grimoire reads it to install themes (its
kernels live in
[grimoire-registry](https://github.com/chinese-room-solutions/grimoire-registry)).
Consumers filter by `kind` and ignore kinds they don't know, so adding a kind
never breaks a released client.

## Schema

`index.yml`:

- `schema_version` — integer. Currently `1`.
- `packages` — list. Each entry:
  - `name` — unique package name: the repo name for runtimes/workers (e.g.
    `mass-runtime-gateway-llama-cpp`), `theme-<id>` for themes.
  - `kind` — `runtime`, `worker`, or `theme`.
  - `runtime_name` — runtime/worker only; the join key. Workers map n:1 onto
    the runtime with the same `runtime_name` (e.g. both llama-cpp packages use
    `llama-cpp`).
  - `display_name` — human label. For workers, do not repeat the word "worker"
    (UIs already label the field, so "llama.cpp worker" renders as
    "Worker: llama.cpp worker") — use the bare product name, e.g. `llama.cpp`.
  - `description` — free text.
  - `versions` — list, one entry per released version:
    - `version` — semver string (e.g. `0.1.0`).
    - `mass` — Semver range of MASS server versions this version works with
      (e.g. `">=0.1"`). Required on runtimes; optional on workers (a worker talks
      to the MASS server directly via the hub protocol) where an empty value
      means unconstrained.
    - `runtime` — **worker only.** Semver range of *runtime* versions whose
      payloads this worker decodes (e.g. `">=0.1 <0.2"`).
    - `grimoire` — **theme only.** Semver range of Grimoire versions the theme
      works with, enforced by Grimoire when it resolves a theme to install.
      Empty means unconstrained (the typical case — themes are token packs).
    - `artifacts` — map, keyed by platform. Each value is `{url, sha256}`:
      - runtime key format: `os/arch` (e.g. `linux/amd64`).
      - worker key format: `os/arch/backend` (e.g. `linux/amd64/vulkan`).
      - theme key: `any` — platform-independent (CSS).
      - `url` — GitHub Release asset download URL.
      - `sha256` — hex digest of the asset. `TBD` is a placeholder for
        unreleased assets (Phase 4 replaces every `TBD` with a real digest).

Only list artifacts the package's release workflow builds today. The worker's
registry artifact is the raw self-contained installer binary
(`mass-worker-setup_<os>_<arch>`) — the join bootstrap fetches and execs it
directly. The AppImage/.app containers on the same release are for manual
double-click installs and are not indexed; the worker-binary `.tar.gz` is not
an installer.

## Theme artifacts

A **theme** artifact is a single self-describing `.css` file per the mass-sdk
uikit contract: the file name (sans `.css`) is the theme id and CSS class
suffix, optional `/* label: ... */` and `/* base: dark|light */` directives
ride in comments, and the body is custom-property declarations only. It
installs into the shared `<config>/mass/themes/` dir, where both MASS and
Grimoire load it — install once, themed everywhere. No archive wrapper: the
index row carries version + sha256, the file carries its own metadata.

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

Ranges in the index are trusted to be well-formed — CI validates every edit
here (see Publishing), so a malformed range never reaches the default branch.
At resolution time an unparseable range is a resolution **error**, not a
version to skip. A malformed *installed* version supplied by the resolving
client is an input error, distinct from "no version covers it".

## Publishing

1. Tag a release in the package repo (`git tag vX.Y.Z && git push --tags`). Its
   release workflow builds and uploads the assets under stable, versionless
   basenames.
2. Open a PR here adding or updating the package's `versions` entry: the new
   `version`, its ranges (`runtime` and/or `mass`), and one `artifacts` entry
   per built platform with the release download URL and the asset's real
   `sha256` (`curl -L <url> | sha256sum`).
3. Add a worker row and the runtime row that reference each other's new minor
   in **one** commit, never separately. A worker needs a runtime reporting the
   version its `runtime` range covers, and that runtime is out of range for
   every earlier worker, so publishing one without the other resolves to a pair
   MASS refuses at Register.

**Skew policy.** A worker floats within one runtime (gateway) minor. A
`runtime` range is expected to span exactly one minor, e.g. `">=0.2 <0.3"`, so
either side can ship patches on its own. Crossing a minor is a breaking pair
and lands under the one-commit rule above.

Every push and pull request runs `registry-validate` from
[mass-sdk](https://github.com/chinese-room-solutions/mass-sdk), the same code
that reads this index at runtime: schema, unique names, ascending semver,
parseable ranges, and artifacts carrying a URL and a real-or-`TBD` digest. A
weekly job (also runnable on demand) downloads every artifact and re-checks its
`sha256`, which catches a release asset re-uploaded after publication.
