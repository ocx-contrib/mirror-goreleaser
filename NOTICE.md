# NOTICE

This repository packages and redistributes upstream software published by the
[GoReleaser](https://github.com/goreleaser/goreleaser) project. The Apache-2.0
license in [`LICENSE`](LICENSE) covers the OCX pipeline files authored here. It
does **not** cover any upstream-derived asset — each package's redistributed
bytes carry their own license, recorded below.

The logo shipped in `goreleaser/logo.svg` / `goreleaser/logo.png` is
**upstream's own mark** (`www/static/logo.svg` from the goreleaser repository),
reproduced unmodified apart from being rasterised to a 512×512 transparent PNG
for consumers that reject SVG. It is used to identify the mirrored software and
no endorsement is implied.

| Package | GHCR path | Upstream SPDX |
|---|---|---|
| `goreleaser` | `ghcr.io/ocx-contrib/goreleaser/goreleaser` | `MIT` |

---

## `goreleaser`

Upstream: <https://github.com/goreleaser/goreleaser>
Published to `ghcr.io/ocx-contrib/goreleaser/goreleaser`.

| Component | SPDX | Holder |
|---|---|---|
| GoReleaser (`goreleaser`) | **MIT** | Copyright © 2016-2026 Carlos Alexandro Becker |

Verified at the redistribution gate:

```
$ gh api repos/goreleaser/goreleaser/license --jq '{spdx: .license.spdx_id, name: .license.name, path: .path}'
{"name":"MIT License","path":"LICENSE.md","spdx":"MIT"}
```

Note the file name: upstream's license text lives in **`LICENSE.md`**, not
`LICENSE`. The copyright holder above is taken verbatim from the first lines of
that file.

The MIT License grants redistribution of the software, with or without
modification, on the single condition that the copyright notice and the
permission notice accompany every copy. That condition is met by construction:
upstream's `LICENSE.md` ships **inside every mirrored archive**, at the archive
root beside the binary, and is republished unmodified as part of the bundle
content.

**Scope.** Only the open-source GoReleaser distribution is mirrored here.
GoReleaser **Pro** is a separate product distributed under upstream's
[`EULA.md`](https://github.com/goreleaser/goreleaser/blob/main/EULA.md) rather
than the MIT license, and is **not** mirrored, republished or otherwise
redistributed by this repository. The mirrored assets are exactly the
`goreleaser_<OS>_<arch>` archives attached to the OSS GitHub releases.

The published binaries statically link third-party Go modules under permissive
licenses, enumerated in the `go.mod` / `go.sum` of the tagged upstream source
for each mirrored version. Upstream additionally publishes an SBOM sidecar per
release asset (`*.sbom.json`), which is not redistributed here.

No modifications are made to any upstream artifact in this repository; they are
republished byte-for-byte inside an OCX bundle.
