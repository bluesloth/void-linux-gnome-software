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

## stores-r6 (2026-11-03)

### `[XBPS-REFRESH]` — sync the repodata before an update check

`revision=8 → 9`. Software's update list comes from `xbps-install -nSu`,
which is a **dry run**: it answers from the cached repodata and never syncs
it. Nothing else on a stock Void box syncs either (there is no xbps cron
job), so the backend reported "up to date" indefinitely while the mirrors
moved on — the defect the COSMIC store carried until `epoch-1.11.0-r7`, fixed
here in the same shape.

**What is new.** The plugin now implements the plugin vfunc pair Software
calls when it checks for updates:

* `gs_plugin_xbps_refresh_metadata_async()` (`guint64 cache_age_secs`,
  `GsPluginRefreshMetadataFlags`, `GsPluginEventCallback`, `GCancellable` —
  the 50.2 `lib/gs-plugin.h` signature) spawns
  `pkexec /usr/libexec/gnome-software-xbps-refresh`, `g_timeout_add_seconds
  (120)` guards the result, and `gs_plugin_xbps_refresh_metadata_finish()`
  consumes the child's output.
* The child's stdout/stderr are **collected, never forwarded** — Software
  raises its passive-message sheet on any plugin message, the same trap the
  KDE backend hit. Nothing here becomes a user-visible message.
* It **always** returns `TRUE`. The refresh is best effort: a refused
  authorisation, a missing helper or an unreachable mirror leaves the cached
  index in place, and the update list still populates from it. Failing the
  vfunc would have replaced a stale list with no list.
* The helper is given `/dev/null` on fd 0 through
  `g_subprocess_launcher_take_fd()`, so a repository key-import question
  reads EOF and declines exactly as it does when the helper is run alone.
  This is why the refresh path builds its own `GSubprocessLauncher` instead of
  reusing `gs_xbps_run_async()`, whose child inherits stdin.
* Exit codes are named in the plugin log rather than collapsed by
  `gs_xbps_run_finish()` (which maps anything non-zero to
  `G_IO_ERROR_FAILED`): **126** authentication cancelled, **127** pkexec
  could not run the helper, **75** the helper's `EX_TEMPFAIL` throttle
  ("index synced recently"), signal death, and the 120 s timeout.

**The helper** is `/usr/libexec/gnome-software-xbps-refresh`, package-owned,
mode 755, `#!/bin/sh`, `set -u`: no arguments, root-only (it refuses
otherwise, because `xbps-install` as a normal user fails every reposync with
EPERM *and still exits 0*), one hardcoded `xbps-install -S`, a `/run` stamp
throttle returning 75, and a preference for `/usr/bin/xbps-install.real` over
the `void-xbps-snap` symlink so an update check does not take a snapper
snapshot pair.

**The privilege model.** A fourth action, `org.gnome.Software.xbps.refresh`,
is `allow_active=yes` / `allow_any=no` / `allow_inactive=no`, annotated on
`org.freedesktop.policykit.exec.path=/usr/libexec/gnome-software-xbps-refresh`.
The three transaction actions are **unchanged from revision 8**:
`auth_admin` on all three defaults, `exec.argv1` `--yes` / `--yes` /
`--sync`, annotated on `/usr/local/sbin/void-xbps-snap` (install, update) and
`/usr/bin/xbps-remove` (remove). Annotating `argv1=-S` on `xbps-install`
itself was rejected for the same reason as in the other two stores: pkexec
matches only the first argument, so that grant would have allowed a
passwordless full transaction from any active session. A prompt on every
update check would have made the feature unusable, and a refused refresh is
harmless by construction.

**A build-time catch worth remembering.** `g_subprocess_launcher_spawnv()`
returns a `GSubprocess *`, not a `gboolean` — it takes `(launcher, argv,
&error)` and yields the child. The first revision-9 build died on
`error: too many arguments to function` and
`passing argument 3 … from incompatible pointer type`; the patch now reads
`subprocess = g_subprocess_launcher_spawnv (launcher, argv, &local_error); if
(subprocess == NULL) { … }`. The new-file hunk header had to be recomputed
(`@@ -0,0 +1,1201 @@`) and `git apply --check` re-run against a pristine 50.2
tree.

**Verified at build time** (this box runs COSMIC, so Software does not exist
here — the packaged bytes are what could be checked):

* `gnome-software-50.2_9.x86_64.xbps`, 2686628 B,
  `sha256 adf2c10a…96ed`; `gnome-software-devel-50.2_9.x86_64.xbps`,
  324467 B, `sha256 35d1f484…e954`. `./xbps-src pkg` `EXIT=0`.
* inside the main package: `usr/libexec/gnome-software-xbps-refresh`
  (4642 B), `usr/share/polkit-1/actions/org.gnome.Software.xbps.policy`
  (3936 B, **exactly 4 actions**), and
  `usr/lib/gnome-software/plugins-23/libgs_plugin_xbps.so` (md5
  `d6da8c7b…6ed0`, was `22268a79…` at revision 7 — the difference is the
  refresh vfunc pair and its message strings).
* `strings` on the shipped plugin (plain C literals, unlike Qt's UTF-16
  `QStringLiteral`s) shows `/usr/libexec/gnome-software-xbps-refresh` and all
  thirteen `xbps: repodata …` messages.
* `sh -n` clean on the helper; the policy parses as XML, and no comment
  contains `--` (which would silently break policy loading).

**Not verified here** (needs a GNOME box, owed): the click-driven matrix in
`docs/HANDOFF-GNOME-SOFTWARE-XBPS-BACKEND.md` §15.4, including the refresh
path at runtime.

### `[PUBLISH-INDEX-ORDER]` — the publish tooling fix that rode along

Before this release was published, `tools/publish.sh` still built
`xbps-rindex -f -a`'s argument list with `find … | sort`. That is
lexicographic, and `xbps-rindex -a` is last-wins for a given pkgname with `-f`
overwriting unconditionally — so the moment a revision reaches `_10` the index
ends up naming the **older** build. It bit
`void-linux-discover`'s `stores-r4` release exactly that way (served
`discover-6.7.5_10`, index said `_9`); the corrected index there is
`stores-r5`, and the full analysis is in that repository's `CHANGELOG.md`.

The script now resolves the index list per pkgname with `xbps-uhelper cmpver`
and adds only the newest build of each name, printing a note when that differs
from the lexical pick. It is copied byte-for-byte from the COSMIC repository's
`tools/publish.sh`, which has carried the fix since `cosmic-store` `_10`. The
two store repositories' copies stay identical by design (edit both together).

Here the trap would have fired too: `gnome-software-50.2_9` sorts **after**
`_8`, so `stores-r6` was already correct, but the next revision would not have
been. The stage's `find` order is
`_7 _8 _9` for both `gnome-software` and `gnome-software-devel`, so the
resolver adds `_9` for each name and prints no note.

---

## stores-r5 (2026-11-01)

### `[POLKIT-SNAP-PATH]` — annotate the wrapper the installer actually ships

`revision=7 → 8`. The `install` and `update` actions move their
`org.freedesktop.policykit.exec.path` annotation from `/usr/bin/xbps-install`
to `/usr/local/sbin/void-xbps-snap`. **Nothing else changes** — the action ids,
every `exec.argv1` (`--yes` / `--yes` / `--sync`), the `auth_admin` defaults,
the 1595-line backend patch and the template's build recipe are byte-identical
to revision 7. The `remove` action deliberately keeps `/usr/bin/xbps-remove`:
the installer's snapshot shadow only covers `xbps-install`, so `remove` already
matched.

**Why.** On every installer-provisioned box, `/usr/bin/xbps-install` and
`/usr/sbin/xbps-install` are symlinks to the installer-owned
`/usr/local/sbin/void-xbps-snap` (the snapper pre/post wrapper). pkexec
canonicalises the program path (`realpath()`) *before* it looks up an action,
so the old annotation could never match there: lookup fell through to pkexec's
generic fallback and the user saw raw-argv wording ("Authentication is required
to execute /usr/local/sbin/void-xbps-snap --yes") instead of this store's own
"Authentication is required to install software" message. Annotating the
canonical path restores the store's own message.

**Failure mode is graceful, by design.** On a stock Void box (no shadow) the
canonicalised path is the real `/usr/bin/xbps-install`, never matches the
wrapper annotation, and lookup falls back to the same generic prompt as before.
A wrong-path annotation can only degrade the *wording*; it can never authorise
anything different, because every action stays `auth_admin` and the wrapper
passes its argv straight through.

Artefacts: template `revision=8`; policy `gs-xbps.policy`, sha256
`88b69fbb6d86a5149f8ef87c811cbb97504053e07f42d57a4d03a5b27581942e`; package
`gnome-software-50.2_8.x86_64.xbps`, 2684132 B, sha256
`379d14be16c0f3ca5691b12df363857ece5365ffabaa93570e7b0cc52dddaaf6`;
`gnome-software-devel-50.2_8.x86_64.xbps`, 324473 B, sha256
`f9f8d28f9f21bb7b171ae9c226c2735cab777054fd19e5862939bac026c1d932`.
The backend plugin is behaviourally unchanged — this is policy data only. The
signing key is unchanged (DER sha256 `9bdf4599…0d60`).

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
