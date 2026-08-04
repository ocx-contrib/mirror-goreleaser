# mirror-goreleaser

OCX mirror for [GoReleaser](https://github.com/goreleaser/goreleaser), release
automation for Go, Rust, Zig, TypeScript and Python projects. One repository,
one spec directory per package.

| Package | Spec | Publishes to | Announced as | Upstream SPDX |
|---|---|---|---|---|
| [goreleaser](https://github.com/goreleaser/goreleaser) | [`goreleaser/mirror.yml`](goreleaser/mirror.yml) | `ghcr.io/ocx-contrib/goreleaser/goreleaser` | [`ocx.sh/goreleaser/goreleaser`](https://index.ocx.sh/goreleaser/goreleaser) | `MIT` |

Each upstream release is discovered, re-bundled, smoke-tested per
`(version, platform)` and only then pushed with cascade tags, after which the
result is announced into the OCX index.

`goreleaser` is the project's own brand as well as its GitHub org, so the org
is the namespace: the package is `goreleaser/goreleaser`. Provenance lives in
the index claim's `upstream` block, not in the namespace.

Only the **open-source** distribution is mirrored. GoReleaser Pro ships under a
separate EULA and is out of scope — see [`NOTICE.md`](NOTICE.md).

## Layout

```
goreleaser/
├── mirror.yml          the spec — never at the repo root
├── metadata.json       bundle interface
├── CATALOG.md          → ocx package describe
├── logo.svg / logo.png describe assets, 512px PNG
└── tests/smoke.star    Starlark smoke test
```

`LICENSE` and `NOTICE.md` are shared at the root. Logos are **not** — each
package carries its own, because a repo-root `logo.*` sits in no workflow's
`paths:` filter, so replacing it would publish nothing until some unrelated
edit happened to fire.

There is **no `mirror-base.yml`** and no `extends:` chain: every key this
single package needs is package-owned. That is deliberate — `extends:` is a
shallow merge of top-level keys, and a spec that restates `platforms:` to
change one runner drops every `containers:` entry with it while nothing reds.
With one spec there is nothing to merge and the trap does not exist. A second
package here would move the genuinely repo-wide keys into a base; that is a
content move, not a rename, and changes no generated workflow filename.

## Platforms

`goreleaser` publishes six platform entries: both Linux arches, both macOS
arches and both Windows arches. All six are shipped by upstream on every
in-range release.

Upstream builds are **static Go binaries**. Measured with `file` / `readelf` on
both declared Linux arches at both ends of the mirrored range — 2.16.0, 2.17.0
and 2.17.1, whose Go toolchain moves 1.26.3 → 1.26.4 → 1.26.5 mid-series — all
show no `PT_INTERP`, no `DT_NEEDED` and no versioned GLIBC symbol references at
all. UPX was ruled out by measurement rather than assumption: `readelf -h`
reports 16 real section headers (a packer stub reports 0), and the single
`^UPX` string in the binary is goreleaser's own `upx:` config key, not a packer
signature. `os.features` states what an artifact requires *of the host*, so
both Linux keys are **bare**: `+libc.glibc` would hide the package from every
musl host and `+libc.musl` from every glibc host, and neither is true. The
`alpine:3.20` container leg on both arches is what turns that claim into
evidence; the measurement transcript is recorded above the `assets:` block in
`goreleaser/mirror.yml`.

Upstream also publishes `armv7`, `i386`, `loong64`, `ppc64` and `riscv64`
archives, a `Darwin_all` universal binary, and `.rpm`/`.deb`/`.apk`/
`.pkg.tar.zst`/`.flatpak` distro sidecars. ocx expresses exactly `amd64` and
`arm64`, so none of the extra arches is a declarable platform — the anchored
`^…$` asset regexes are what keep all of them out.

Three distinct ambiguity traps make those anchors load-bearing, and an OS-only
anchor defuses none of them:

1. **`goreleaser_Darwin_all.tar.gz`** is a universal (fat) binary covering both
   darwin arches. A pattern anchored only on `Darwin` matches it as a third
   darwin candidate and the resolver hard-errors as ambiguous. The per-arch
   assets are preferred; the literal arch token excludes `_all`.
2. **Every archive has a same-named `.sbom.json` sidecar**
   (`goreleaser_Linux_x86_64.tar.gz.sbom.json`), so a prefix-matching pattern
   hits two assets per platform — hence the `$` on every pattern.
3. **The package-manager assets use a different naming scheme that shares
   tokens with the archives** (`goreleaser-2.17.1-1.x86_64.rpm`,
   `goreleaser_2.17.1_amd64.deb`, `goreleaser_2.17.1_x86_64.apk`,
   `goreleaser-2.17.1-1-x86_64.pkg.tar.zst`). They are excluded by
   **extension**, never by token overlap — `x86_64` and `amd64` appear on both
   sides of that line, and `.pkg.tar.zst` would even satisfy a naive "is it a
   tarball" test.

Resolution was verified **both ways on every in-range release**: each pattern
matches exactly one asset, every time. A pattern matching zero would be
silently skipped rather than reported, so this check is not optional.

Asset names here are **not** version-suffixed — `goreleaser_Linux_x86_64.tar.gz`
is identical in name across every release — so "does this asset belong to this
tag" cannot be answered from the filename. It was answered by extracting and
running each tag's binary: `goreleaser --version` self-reports `GitVersion:
2.16.0`, `2.17.0` and `2.17.1` respectively. No release re-ships a
predecessor's build.

Upstream also publishes a rolling `nightly` tag and dated
`v2.18.0-<shorthash>-nightly` tags, all flagged `prerelease=true`. Both
`skip_prereleases: true` and the `^v\d+\.\d+\.\d+$` tag anchor exclude them
independently; both are kept so neither is a single point of failure.

## Editing

| File | Edit | Regenerate after |
|------|------|------------------|
| `goreleaser/mirror.yml` | hand | yes — see below |
| `goreleaser/{metadata.json,CATALOG.md,logo.*}` | hand | — |
| `goreleaser/tests/smoke.star` | hand | — |
| `.github/workflows/*.yml` | **generated — never hand-edit** | re-run when a spec changes |

```bash
ocx-mirror package pipeline generate ci --spec goreleaser/mirror.yml
```

**Name every spec.** `--spec` *appends* rather than replaces, so a command
naming a subset silently stops rendering the rest while staying green — and the
drift guard reds on a generated workflow the current spec set no longer
produces.

`verify-generated.yml` exits 65 on drift. If a generated workflow is wrong, the
spec or the renderer template is wrong — fix it there and regenerate.

Run `direnv allow` once to put the pinned toolchain on `PATH`, and invoke
`ocx-mirror` directly — never `ocx run -- ocx-mirror`, which pins
`OCX_BINARY_PIN` to the bootstrap `ocx` and false-reds the nested push.

## The binaries claim

`goreleaser/metadata.json` declares `binaries: ["goreleaser"]` by hand, and
`goreleaser/mirror.yml` sets `bin_scan: "off"` — forced, not preferred. The
scan only inspects an interface-visible `${installPath}/<dir>` PATH entry, and
goreleaser's archives are **flat**: `goreleaser` (`goreleaser.exe` on Windows),
`LICENSE.md` and `README.md` sit at the archive root with no `bin/`. The two
subdirectories that do exist — `completions/` and `manpages/` — hold data, not
executables, and pointing PATH at either would be a lie. With nothing to
inspect the scan would pass green whatever the archive contained, so `auto` and
`verify` both fail spec load at exit 65 rather than offer a hollow check. The
hand-written list is what the error message itself directs, and it is short and
stable — the binary is the only mode-0755 entry in the archive.

`manpages/goreleaser.1.gz` deliberately gets no `MANPATH` entry: `man` resolves
`$MANPATH/man<N>/<page>` and upstream ships the page with no `man1/` level, so
a `MANPATH` pointing there would resolve nothing.

## The smoke test

`goreleaser build` and `goreleaser release` — the headline verbs — need a real
git repository, a working Go toolchain and network access, so neither is
reachable hermetically. The smoke asserts the functional pair underneath them,
in both directions:

- `goreleaser --version` — liveness on the composed PATH plus the version
  *shape* (`\d+\.\d+\.\d+`), never the ASCII banner around it.
- `goreleaser init` in a scratch subdirectory — it writes `.goreleaser.yaml`
  **and** `.gitignore`. The test asserts neither pre-existed, then that both
  appeared and the config is a real v2 config: a real on-disk side effect a
  binary that merely logged "generating .goreleaser.yaml" could not produce.
  The pre-assert is not theoretical — `init` aborts with "already exists" when
  they do, so a stale scratch reds here rather than silently passing.
- `goreleaser check one.yaml two.yaml three.yaml` — three distinct valid
  configs in one invocation, asserting exit 0 **and** that exactly three
  `checking` lines were emitted. The count is what makes this more than
  "exit 0": a run that validated only the first argument and short-circuited
  green would report 1 and fail.

The **negative control** is the same validator in the other direction:
`goreleaser check bad.yaml` on a structurally malformed config (`builds` as a
scalar where a list is required) must exit non-zero. A validator that accepts
everything is indistinguishable from `true`, and every other assertion here
would still pass for a binary that exited 0 unconditionally.

Nothing the smoke runs touches `git`. Measured inside bare `ubuntu:24.04`,
`alpine:3.20` and `fedora:40` — none of which ships git — every verb above
exits as asserted. `init` writes `.gitignore` itself rather than shelling out,
and the configs the test writes set `release.disable: true`, which skips the
scm pipe that is the only git-touching path. That is why no `containers[].setup`
line is declared: a bare `os.features` key claims the artifact needs *nothing*
of the host, and provisioning git would make these legs prove strictly less.

## Required secrets

| Secret | Use |
|--------|-----|
| `OCX_ANNOUNCE_TOKEN` | opens the index pull request from the `ocx-contrib/index` fork |
| `OCX_MIRROR_DISCORD_HOOK` | notify-stage Discord webhook URL |

(Inherited from the `ocx-contrib` org with visibility ALL. GHCR pushes use the
run's own `GITHUB_TOKEN` — no registry secret needed.)

## License

Apache-2.0 — see [`LICENSE`](LICENSE). Upstream assets are out of scope; the
redistribution license is recorded in [`NOTICE.md`](NOTICE.md). The logo is
upstream's own mark — see `NOTICE.md`.
