# Changelog

All notable changes to **void-linux-gnome-software** — the `xbps-src`
packaging source for GNOME Software with a native xbps backend — are
documented in this file.

## What this repository is

One repository per store: this repo carries **GNOME Software only**. KDE
Discover lives in `private/void-linux-discover/`, cosmic-store in
`private/void-linux-cosmic/` (epoch-managed). The two store repos carry
byte-identical copies of `tools/`; edit them together.

**2026-10-21 — created by the per-store split (Option C).** The original
plan (2026-10-20, Decision 2 / Option B) grouped Discover and GNOME Software
in one shared `void-linux-stores` repo; the operator split them before the
first publish — the only cheap window, since a served repo's repodata,
signing key, Pages site, installer repo URL and PINS rows would otherwise
all need re-cutting. Per-store repos make a one-store PR trivial, keep
provenance store-pure and isolate publish blast radius.

## Release model

Each template carries its own upstream `version=` and `revision=`; a store's
release is identified by `<upstream-version>_<revision>`. The repository
release tag stamped by `tools/publish.sh` is `stores-rN`, a monotonic
per-repository counter (independent of the discover repo's tags) — there is
no shared epoch to key a tag on, unlike the COSMIC repository's
`epoch-N.M.P`.

**Caution — repodata must be re-signed.** Any repack of this repository's
repodata MUST be signed with the repository signing key; an unsigned repack
makes clients abort (the 2026-10-09 `void-cosmic-repo` incident).

## stores-r4 (2026-10-30)

### `[GNOME-SOFTWARE-XBPS]` — rev7: unserviceable queries return an empty list, not an error

`revision=7` (`2026-10-28`) changes exactly one branch of the backend. A query
the xbps plugin cannot answer used to fail the `GsPlugin` task outright:

```c
g_task_return_new_error (task, G_IO_ERROR, G_IO_ERROR_NOT_SUPPORTED,
                         "Unsupported query");
```

It now completes successfully with an empty result set:

```c
g_autoptr(GsAppList) empty = gs_app_list_new ();
g_task_return_pointer (task, g_steal_pointer (&empty), g_object_unref);
```

That matches what GNOME Software expects from a plugin that simply has nothing
to add for a given query (the same shape the flatpak and packagekit plugins
use). The previous form turned "I don't serve this category" into a plugin
error, which surfaces as a failure notification rather than an empty list. No
other source file changed; the rest of the backend is byte-identical to rev6.

Artefacts: template `revision=7`; patch 1595 lines, sha256
`b23e999508b3d2fc74511d60f4a23d2a1f29ba3224381c3376d107aae816c4bc`;
package `gnome-software-50.2_7.x86_64.xbps`, 2684150 B, sha256
`ed078a16c2cc4bfa36d84c7e74b5bb5bcd29e02ca5d25427e3b03321dcca2220`;
`gnome-software-devel-50.2_7.x86_64.xbps`, 324466 B, sha256
`18e2bbc65b35a05b33d079cc9ba78dd3d2c5654f28f008df834710502a5132d7`;
`x86_64-repodata`, 2036 B, sha256
`c9f450cadf935de4f99a09cc02800c82e0c144b8bde899b42aea4526d25ea649`.
Digests are in `PINS.md`. The key is unchanged (DER sha256 `9bdf4599…0d60`).

**Validation, on a real GNOME install.** An operator GNOME desktop install of
Void succeeded with GNOME Software running from this store — i.e. the chain
the installer's `install_gnome_software_store()` exists to build (store config
written, this store's key imported, `gnome-software` resolved from the store
rather than Void's stock flatpak-only build). The version was not captured on
that box; `50.2_7` is the newest build this store has ever served, so it is
the build that install got. This closes item 13 ("fresh-install story") of the
§5 matrix in `docs/HANDOFF-GNOME-SOFTWARE-XBPS-BACKEND.md`, which the rev6
round left open.

**This release was published on 2026-10-28 without a release tag**, and the
published `main` was then rewritten down to the served artefacts — see below.

### Repository restored to the one-repo model (`stores-r4`, 2026-10-30)

`50.2_7` was pushed as a **five-file repository**: two packages, their two
`.sig2` files and `x86_64-repodata`. Everything else that `stores-r3`
established went missing from the published branch — `srcpkgs/gnome-software/`
(the template, the polkit policy and the 1595-line backend patch), `tools/`,
`README.md`, this file, `PINS.md`, `LICENSING.md`, `.nojekyll` (so GitHub
Pages was free to run the binary repository through Jekyll) and `index.html`.
The documented model for this project is **one repository = packaging source
*and* served xbps repository** (the cosmic and discover precedent); a store
whose source exists only on one developer's disk is not rebuildable from its
own release, and the served set is no longer reproducible from the published
tree.

The restore commit sits **on top of the published tip** — a fast-forward, not
a force-push, no history rewritten — and adds only the missing source and
support files. Not one served byte moves: `gnome-software-50.2_7`,
`gnome-software-devel-50.2_7`, both `.sig2` and `x86_64-repodata` are
tracked unchanged, so the index and package digests above, the installer's
store URL and its pinned fingerprint all stay valid. The repository is tagged
**`stores-r4`** at this state, so the newest published build finally carries a
release tag (it had none).

Two tags from the rewritten-away history were **deleted**: `stores-r1` →
`a207b32` (2026-10-28, the publish that re-introduced ~269 MB of borrowed
cosmic/KDE `.xbps` blobs) and `stores-r3` → `bff1185` (the 2026-10-27
single-commit history). Neither is reachable from `main` any more, and an
unreachable tag keeps its whole commit's objects alive in GitHub's storage —
deleting them is what lets the blob purge of 2026-10-27 actually take effect.
Their content is not lost: `bff1185` is the rev6 release this file still
describes, and a bundle of the pre-restore history was kept outside the
repository.

## stores-r3 (2026-10-27)

### `[GNOME-SOFTWARE-XBPS]` — rev6: AppStream-independent backend, validated end-to-end, published

The xbps backend no longer needs AppStream catalogs: it constructs a `GsApp`
per package from `xbps-query -l` / `-Rs` / `-R` output, with synthetic id
`org.voidlinux.xbps.<pkgname>`, generic icon, bundle `package`, scope
`system`, origin `void` (UI label `Void Linux`). Existing AppStream identity is
never overwritten, and the plugin still runs after `appstream`. CLI wildcard
apps (`--details=<data-id>`) are resolved by creating a new app in
`refine_next()` (GS's required pattern), which removes the
“plugins should not set the management plugin” warning.

On a real Void GNOME DE + GDM box with stock polkit (`auth_admin`, no
permissive rule), the §5 matrix passes items 1–9, 11 and 12: install, cancel,
uninstall, self-update, a 14-package full system update via the updater (one
prompt each), flatpak regression, and cold-start prompt hygiene. Item 10
(OctoXBPS) and item 13 (fresh-install story) are not run.

Artefacts: template `revision=6`; patch 1595 lines, sha256
`ae12aa7f392faa4e3e88c18b6cc5cb10e10ab3de0ca5ea92e368d58ed11c2aa2`;
package `gnome-software-50.2_6.x86_64.xbps`, sha256
`758039ea2e37c69b76b09886100e38a4bf6c163669e305ce8c3c66b6c65856f1`.

**Published as `stores-r3`** (2026-10-27): the built package, its `.sig2` and
the signed `x86_64-repodata` are committed at this repository's root and served
by GitHub Pages at `https://bluesloth.github.io/void-linux-gnome-software/`.
The index is signed with this store's own RSA-4096 key (DER sha256
`9bdf4599…0d60`); the key stays off-repo and only its fingerprint travels (the
installer pins it). Digests are in `PINS.md`.

**History purged by delete + recreate** (2026-10-27, same day): the first
publish had carried ~269 MB of borrowed cosmic/KDE `.xbps` blobs (a bad copy at
scaffold time). Deleting the files in a later commit does not remove the
content — the blobs stay reachable through `main`'s ancestry — so the GitHub
repository was deleted and recreated with a single commit (`bff1185`), and the
pre-publication local tags `stores-r1`/`stores-r2` were dropped. The served
bytes are byte-identical to what `stores-r3` describes, so nothing downstream
changed.

### `[STORES-SPLIT]` — repository scaffolded (2026-10-21)

`tools/` (build-and-stage, check-maintainer, publish, sync-upstream),
`.gitignore`, README and this file — copied byte-identically from the
discover repo. No template yet: `[GNOME-SOFTWARE-XBPS]` implementation has
not started (plan in
`docs/HANDOFF-GNOME-SOFTWARE-XBPS-BACKEND.md`, whose §8 Q4 now resolves to
this repository).
