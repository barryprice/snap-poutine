# snap-poutine

**[poutine](https://github.com/boostsecurityio/poutine) as a [Snap](https://snapcraft.io) package.**

poutine is a security scanner that detects misconfigurations and vulnerabilities in the build pipelines of a repository. It parses CI workflows from GitHub Actions and GitLab CI/CD (plus Azure DevOps and Pipelines as Code Tekton), and given an access token with read-level access it can analyze every repository in an organization to give a view of that organization's software supply chain posture. This repo packages it as a strictly-confined Snap.

## Install

[![Get it from the Snap Store](https://snapcraft.io/en/dark/install.svg)](https://snapcraft.io/poutine-bp)

```bash
sudo snap install poutine-bp
```

The snap is named `poutine-bp` (the `-bp` suffix marks it as a community build rather than an upstream publication), but the app itself is `poutine`. Either invoke it as `poutine-bp.poutine`, or add an alias so the command reads like upstream:

```bash
sudo snap alias poutine-bp poutine
poutine version
```

## Usage

Analyze a local checkout:

```bash
poutine analyze_local .
```

Analyze a remote repository, or every repository in an organization (both need a token):

```bash
poutine analyze_repo org/repo --token "$GH_TOKEN"
poutine analyze_org org --token "$GH_TOKEN"
```

Analyze a self-hosted GitLab instance:

```bash
poutine analyze_org my-org/project --token "$GL_TOKEN" \
  --scm gitlab --scm-base-url https://gitlab.example.com
```

`analyze_local` reads only local files, so it needs no token and no network.

### Exit codes

| Code | Meaning |
| ---- | ------- |
| `0` | Scan succeeded, nothing to report — or findings, without `--fail-on-violation` |
| `1` | Error during the scan |
| `2` | Interrupted (SIGINT/SIGTERM) |
| `10` | Findings, **only** with `--fail-on-violation` |

Note that unlike some scanners, findings alone do not change the exit code: pass `--fail-on-violation` if you want `10` to signal them in CI.

### Version check

By default poutine contacts a telemetry endpoint at most once every 24 hours to check for a newer release, reporting the current version, an anonymous instance id and an invocation count — never source or finding data. Disable it with `--disable-version-check`, `POUTINE_DISABLE_VERSION_CHECK=1`, or `disableVersionCheck: true` in `.poutine.yml`.

## Interfaces

The snap runs under **strict confinement** and declares:

| Interface | Notes |
| --------- | ----- |
| `home` | auto-connected; this is what lets poutine read checkouts under `$HOME` |
| `network` | auto-connected; needed by `analyze_repo` / `analyze_org` and the version check |
| `removable-media` | **not** auto-connected — `sudo snap connect poutine-bp:removable-media` to scan a checkout under `/media` or `/mnt` |

> **Note on paths.** The `home` interface excludes top-level hidden directories in `$HOME` (and `~/snap`), so a checkout in e.g. `~/.local/src/repo` is not readable. Only the first path component matters — a repository's own `.github/workflows` is fine, as long as the repository itself is not inside a hidden directory.

Tokens are passed as a flag or environment variable, not read from a file, so no `personal-files` or `system-files` interface is required.

## How it works

The snap builds a **pinned upstream release tag**, recorded in `snap/snapcraft.yaml`. Renovate raises a pull request whenever boostsecurityio/poutine publishes a new release, so upgrades are reviewed rather than picked up silently on the next build; non-release tags are excluded by a rule in [`renovate.json`](renovate.json).

Upstream's `Makefile` `build` target is a bare `go build -o poutine .`, and upstream's release pipeline is what normally injects the version through `-X main.version=`. The snap passes that flag itself, so the packed binary reports the pinned release rather than the source default of `development`. The build uses the `go/latest/stable` snap (upstream requires Go 1.26.0).

### Build gates

Every build exercises the **packed tree**, not the build tree, so `apps.poutine.command` is proven to resolve as well:

- `poutine version` must report exactly the pinned release, tying the binary back to the tag.
- A repository whose workflow interpolates `github.event.issue.title` must exit with poutine's findings code (`10`) under `--fail-on-violation`.
- A benign repository must exit `0`, so the gate also fails on a scan that fires indiscriminately.
- The binary scans this repo's own workflows and must find nothing.

Any of these failing aborts the build instead of shipping an unexercised binary.

The last gate runs from the project directory so poutine picks up this repo's committed [`.poutine.yaml`](.poutine.yaml), which skips the findings we have deliberately accepted (the two GitHub Actions we trust: `snapcore/action-build` and `zizmorcore/zizmor-action`). The two synthetic repositories are scanned from a scratch directory with no config, so that skip list cannot mask a regression in either of them — poutine resolves its config relative to the working directory, never relative to the scanned path.

## Building locally

Requires `snapcraft` (`sudo snap install snapcraft --classic`) and a working LXD:

```bash
snapcraft
snapcraft lint poutine-bp_*.snap
```

## Continuous integration

- [`.github/workflows/snap.yml`](.github/workflows/snap.yml) — builds the snap on every push and pull request, then lints it with `snapcraft lint` in the same job. This is a **merge gate only**; it never publishes. Building **and** publishing to `latest/edge` is handled by [snapcraft.io](https://snapcraft.io) on its own schedule, and this repo is intentionally never tagged — the release tags live upstream.
- [`.github/workflows/zizmor.yml`](.github/workflows/zizmor.yml) — audits this repo's own workflows with [zizmor](https://github.com/zizmorcore/zizmor), since the CI definitions are themselves attack surface.

## Disclaimer

This is a **community-maintained Snap**, not an official BoostSecurity.io project. Report issues with the packaging here; report issues with the tool itself [upstream](https://github.com/boostsecurityio/poutine/issues).

## Credits

- **[boostsecurityio/poutine](https://github.com/boostsecurityio/poutine)** — the supply chain scanner this Snap packages.
- **[barryprice](https://github.com/barryprice)** — Snap packaging and maintenance.

License: Apache-2.0, as declared in `snap/snapcraft.yaml`.
