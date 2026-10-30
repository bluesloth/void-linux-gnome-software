# void-linux-gnome-software

GNOME Software with a native **xbps** backend for Void Linux — the packaging
source **and** the served binary repository for this store.
One repository per store: KDE Discover lives in `private/void-linux-discover/`,
cosmic-store in `private/void-linux-cosmic/` (epoch-managed).

## Status: rev7 PUBLISHED — `stores-r4` served from GitHub Pages (2026-10-30)

The plugin is written, meson-wired, patch-extracted, compiled and **validated on
a real Void GNOME DE + GDM box**. rev7 is
`gnome-software-50.2_7.x86_64.xbps` (2684150 B, sha256 `ed078a16…2220`), the
plugin md5 is `22268a79…888f`, and the patch is 1595 lines, sha256
`b23e9995…c4bc`. rev7's only source change: a query the backend cannot serve
now returns an empty `GsAppList` instead of `G_IO_ERROR_NOT_SUPPORTED`.

The important design change since the first build is that the backend **no
longer depends on AppStream catalogs**: it builds a `GsApp` per xbps package
from `xbps-query` output, with synthetic id `org.voidlinux.xbps.<pkgname>`,
generic icon, bundle `package`, scope `system`, origin `void` (UI label
`Void Linux`). Existing AppStream identity is never overwritten. Browse and
categories stay empty without catalogs — accepted; search, installed, updates
and details are the xbps surface. The `gs_xbps_run_finish()` exit-status fix
from the first build is retained (126/127 → `G_IO_ERROR_PERMISSION_DENIED`).

`srcpkgs/gnome-software/` carries the template (`revision=7`, `-Dxbps=true`),
the `auth_admin` policy, and `patches/add-xbps-plugin.patch`. **Published** as
`stores-r4`: the built artifacts are committed at this repository's root and
served by GitHub Pages at
`https://bluesloth.github.io/void-linux-gnome-software/`, the index is signed
with the store's own key (DER sha256
`9bdf45994696427ede0ca504674066e7276a0cadb6eccba8eb7cd4ae0be70d60`, pinned in
the installer), and the installer installs from it via
`install_gnome_software_store()`. §5 matrix
state is in the handoff's **§13.3**: items 1–9, 11 and 12 PASS with real polkit
authentication on the operator's box (install, cancel, uninstall, self-update,
14-package full system update, flatpak regression, cold-start hygiene); item 10
(OctoXBPS, not installed) is not run. **Item 13 (fresh-install story) is now
PASS**: an operator GNOME desktop install of Void came up with GNOME Software
from this store (version not captured on that box; `50.2_7` is the newest build
this store serves). Resume at
`docs/HANDOFF-GNOME-SOFTWARE-XBPS-BACKEND.md` §13, after
`docs/RESUME-20261027-GNOME-SOFTWARE-XBPS-APPSTREAM-FREE.md`.

The handoff's **§8 Q4 (which repo carries the patch) resolves to this
repository** (2026-10-21 split); **§8 Q5 (installer integration) is resolved and
implemented** — the installer pins this store's key, writes
`/etc/xbps.d/61-gnome-software.conf`, installs `gnome-software` with
`-R <store>` and asserts the recorded origin.

## Contents

| path | what it is |
| --- | --- |
| `srcpkgs/gnome-software/template` | the carried template (revision bump over Void's `gnome-software`) |
| `srcpkgs/gnome-software/patches/add-xbps-plugin.patch` | the xbps plugin (a patch here, unlike Discover's carried sources — GNOME's build tree is stable for it) |
| `srcpkgs/gnome-software/files/*.policy` | the `auth_admin` polkit actions |
| `tools/` | build / gate / publish tooling (byte-identical copies of the discover repo's — edit them together) |
| `CHANGELOG.md` | the decision + release record |
| `PINS.md` | served-artifact digests + the source-distfile checksum |
| `LICENSING.md` | licensing statement for the repository |
| `.nojekyll`, `.gitattributes`, `.gitignore` | Pages + binary-hygiene rules for the served repository (mirrors the cosmic store repo) |

## House rules that apply here

- **Never emit a bare `-Sy`/`-Syy`** as a pkexec argument (OctoXBPS's NOPASS
  action keys on `argv1=-Sy`): `-y` for install/remove, `-Syu` for updates.
- **Privilege** is pkexec + a dedicated `.policy`, every action `auth_admin`.
- **Repacks must be re-signed** (2026-10-09 `void-cosmic-repo` incident).
- Maintainer identity in every template:
  `Void Linux Stores <noreply@github.com>` (gated by
  `tools/check-maintainer.sh`, run as a pre-flight by `tools/publish.sh`).

## Build / publish

Same tooling as the discover repo:

```sh
XBPS_DISTDIR=/path/to/void-packages tools/build-and-stage.sh [pkg ...]
XBPS_SIGNING_KEY=/path/to/key tools/publish.sh --from hostdir/binpkgs [--push]
```

Repository release tags are `stores-rN` (per-repository counter — this repo's
tags are independent of the discover repo's). **Status: published as
`stores-r4`** — the served repository is live at
`https://bluesloth.github.io/void-linux-gnome-software/` and the installer
consumes it.

**History (2026-10-27):** the first publish carried ~269 MB of borrowed
cosmic/KDE package blobs from a bad copy at scaffold time (they were deleted
later, but git keeps deleted content reachable through ancestry). The
repository was therefore deleted and recreated with a **single commit** —
`bff1185`, which `stores-r3` points at — and the pre-publication local tags
`stores-r1`/`stores-r2` were dropped. The served bytes are unchanged by this,
so the installer's URL and key pin are unaffected; the development trail lives
in `CHANGELOG.md` rather than in git ancestry.

**History (2026-10-28 → 2026-10-30):** the `50.2_7` publish landed **without a
release tag** and rewrote `main` down to five files (two packages, two `.sig2`,
the index), dropping `srcpkgs/`, `tools/`, the docs, `LICENSING.md` and
`.nojekyll` — so the published repository no longer contained the source it was
built from, and Pages could run the binary repository through Jekyll. The
`stores-r4` commit **restores them on top of the published tip** (fast-forward,
no history rewritten) and tags that state; **no served byte changed.** The
orphaned tags `stores-r1` → `a207b32` and `stores-r3` → `bff1185` were deleted,
since neither is reachable from `main` and an unreachable tag keeps its
commit's objects — including the cosmic blobs — alive in GitHub's storage.

Served set (root of this repository, 2026-10-30):

| file | sha256 |
| --- | --- |
| `x86_64-repodata` (2036 B, signed) | `c9f450cadf935de4f99a09cc02800c82e0c144b8bde899b42aea4526d25ea649` |
| `gnome-software-50.2_7.x86_64.xbps` | `ed078a16c2cc4bfa36d84c7e74b5bb5bcd29e02ca5d25427e3b03321dcca2220` |
| `gnome-software-devel-50.2_7.x86_64.xbps` | `18e2bbc65b35a05b33d079cc9ba78dd3d2c5654f28f008df834710502a5132d7` |

Both `.xbps` carry a `.sig2` that verifies against the store's public key, and
the signed index embeds that same key (`signature-by Void Linux Stores
<noreply@github.com>`, RSA-4096). xbps does **not** sign the index itself: the
client's trust anchor is the key's DER sha256 fingerprint in
`/var/db/xbps/keys/`, which is what the installer pins.

The served layout follows the cosmic store repo
(`bluesloth/void-linux-cosmic`): the built artifacts are committed at the
repository root (`*.x86_64.xbps`, `*.x86_64.xbps.sig2`, `x86_64-repodata`)
and served from GitHub Pages — hence `.nojekyll`. The signing key stays
off-repo; only its public fingerprint is pinned in the installer.
