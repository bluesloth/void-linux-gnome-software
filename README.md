# void-linux-gnome-software

GNOME Software with a native **xbps** backend for Void Linux — the packaging
source **and** the served binary repository for this store.
One repository per store: KDE Discover lives in `private/void-linux-discover/`,
cosmic-store in `private/void-linux-cosmic/` (epoch-managed).

## Status: rev9 PUBLISHED — `stores-r6` served from GitHub Pages (2026-11-03)

The plugin is written, meson-wired, patch-extracted, compiled and **validated on
a real Void GNOME DE + GDM box** (rev7/rev8 feature matrix, handoff §13.3).
rev9 is `gnome-software-50.2_9.x86_64.xbps` (2686628 B, sha256
`adf2c10a…96ed`); the plugin md5 is `d6da8c7b…6ed0` and the patch is the same
1595 lines, sha256 `b23e9995…c4bc`, as rev7/rev8. rev7's only source change: a
query the backend cannot serve now returns an empty `GsAppList` instead of
`G_IO_ERROR_NOT_SUPPORTED`. rev8's only change: the `install`/`update` polkit
actions annotate `/usr/local/sbin/void-xbps-snap`, the wrapper the
`xbps-install` symlinks resolve to on an installer-provisioned box, so the
prompt shows this store's own wording instead of pkexec's generic raw-argv
fallback.

**rev9 adds the repodata refresh** (`[XBPS-REFRESH]`, template
`revision=9`). `xbps-install -nSu` answers from the **cached** index and
nothing on a stock Void box ever syncs it, so the update list could report a
system as up to date indefinitely. The plugin now implements
`refresh_metadata_async`/`_finish`: it runs
`pkexec /usr/libexec/gnome-software-xbps-refresh` (a package-owned argless
helper, fourth polkit action `org.gnome.Software.xbps.refresh`,
`allow_active=yes`, installed by the same package) and then re-reads the
update list. `_finish` always returns `TRUE` — a failed sync must not break
the refresh, it only means the list stays as it was. The helper path is what
the action annotates because pkexec matches only `exec.argv1`, so no
`xbps-install`-based `argv1` rule can distinguish a sync from an install.

The important design change since the first build is that the backend **no
longer depends on AppStream catalogs**: it builds a `GsApp` per xbps package
from `xbps-query` output, with synthetic id `org.voidlinux.xbps.<pkgname>`,
generic icon, bundle `package`, scope `system`, origin `void` (UI label
`Void Linux`). Existing AppStream identity is never overwritten. Browse and
categories stay empty without catalogs — accepted; search, installed, updates
and details are the xbps surface. The `gs_xbps_run_finish()` exit-status fix
from the first build is retained (126/127 → `G_IO_ERROR_PERMISSION_DENIED`).

`srcpkgs/gnome-software/` carries the template (`revision=9`, `-Dxbps=true`),
the four-action policy, the refresh helper and
`patches/add-xbps-plugin.patch`. **Published** as `stores-r6`: the built
artifacts are committed at this repository's root and served by GitHub Pages
at `https://bluesloth.github.io/void-linux-gnome-software/`, the index is
signed with the store's own key (DER sha256
`9bdf45994696427ede0ca504674066e7276a0cadb6eccba8eb7cd4ae0be70d60`, pinned in
the installer), and the installer installs from it via
`install_gnome_software_store()`. The `[POLKIT-SNAP-PATH]` and
`[PUBLISH-INDEX-ORDER]` notes are in `CHANGELOG.md` and `PINS.md`. §5 matrix
state is in the handoff's **§13.3**: items 1–9, 11 and 12 PASS with real polkit
authentication on the operator's box (install, cancel, uninstall, self-update,
14-package full system update, flatpak regression, cold-start hygiene); item 10
(OctoXBPS, not installed) is not run. **Item 13 (fresh-install story) is now
PASS**: an operator GNOME desktop install of Void came up with GNOME Software
from this store running `50.2_7`; `50.2_8` is that build's plugin
byte-for-byte with only the polkit annotations moved, so the proof carries
over. **Not carried over:** the rev9 refresh path and the cold-start prompt
behaviour (`refresh_metadata_async` runs one `pkexec` per refresh by design) —
those are the first things to verify on a fresh GNOME install. Resume at
`docs/HANDOFF-GNOME-SOFTWARE-XBPS-BACKEND.md` §13 and §15, after
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
| `srcpkgs/gnome-software/files/*.policy` | the polkit actions: `install`/`remove`/`update` `auth_admin`, `refresh` `allow_active=yes` |
| `srcpkgs/gnome-software/files/gnome-software-xbps-refresh` | the argless helper the `refresh` action annotates (`xbps-install -S`) |
| `tools/` | build / gate / publish tooling (byte-identical copies of the discover repo's — edit them together) |
| `CHANGELOG.md` | the decision + release record |
| `PINS.md` | served-artifact digests + the source-distfile checksum |
| `LICENSING.md` | licensing statement for the repository |
| `.nojekyll`, `.gitattributes`, `.gitignore` | Pages + binary-hygiene rules for the served repository (mirrors the cosmic store repo) |

## House rules that apply here

- **Never emit a bare `-Sy`/`-Syy`** as a pkexec argument (OctoXBPS's NOPASS
  action keys on `argv1=-Sy`): `-y` for install/remove, `-Syu` for updates.
- **Privilege** is pkexec + a dedicated `.policy`. Every action that performs a
  **transaction** is `auth_admin`. The single exception is this store's
  `refresh` action (`allow_active=yes`): it runs a package-owned argless helper
  that only syncs the repository index — it installs, removes and upgrades
  nothing, so it cannot alter the system, and requiring a password for it would
  make the GUI's routine update check prompt the user for no benefit. Do not
  extend that exception to a `pkexec xbps-install` action.
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
`stores-r6`** — the served repository is live at
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

**History (2026-11-01):** `stores-r5` is the `[POLKIT-SNAP-PATH]` re-cut.
Template `revision=7` → `8`, two lines of the policy changed, the package
rebuilt (`50.2_8`), signed, indexed and published — a plain fast-forward
(`4b26938..fcfcbeb`) with an annotated tag. The plugin is byte-identical to
`_7`'s, so this release changes nothing but which polkit prompt wording the
user sees.

**History (2026-11-03):** `stores-r6` is the `[XBPS-REFRESH]` release.
Template `revision=8` → `9`: the plugin implements
`refresh_metadata_async`/`_finish` by running a new package-owned helper
(`/usr/libexec/gnome-software-xbps-refresh`) behind a fourth polkit action
(`allow_active=yes`), because `xbps-install -nSu` answers from the cached
repodata and nothing on a stock box syncs it. The plugin binary changes
(md5 `d6da8c7b…6ed0`), so this is not a metadata-only re-cut. The same
release carries the `[PUBLISH-INDEX-ORDER]` tooling fix — `tools/publish.sh`
resolves the newest build per pkgname with `xbps-uhelper cmpver` instead of
trusting a lexical `find` order, mirrored byte-for-byte from the discover
repository (which needed it for its `stores-r5`). `stores-r6` itself was
unaffected (`_9` sorts after `_8`).

Served set (root of this repository, 2026-11-03):

| file | sha256 |
| --- | --- |
| `x86_64-repodata` (2042 B, signed) | `5bc9939b9b73ebd10bf653748b8819287669d59ade2e5205dd5cf3d02fc3cc1b` |
| `gnome-software-50.2_9.x86_64.xbps` | `adf2c10aec9cea583bf3b85184fcb3aae96026b59da30b6539d6ebb3834596ed` |
| `gnome-software-devel-50.2_9.x86_64.xbps` | `35d1f484fde849c822168cdc8457301644c5eff2c1602c50c1bcf3f2c087e954` |

The `_7` and `_8` pairs (indexes `c9f450ca…a649` and `88413789…bac7`) stay at
the root alongside these: build numbers are not pruned (the cosmic precedent),
the index advertises `_9` only.

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
