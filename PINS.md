# PINS - Void_Linux_GNOME_Software

| package | version | checksum |
| --- | --- | --- |
| gnome-software | 50.2 (revision 10) | b6500da19825e59bb4df167dad2792f31387a6b0ff298bc0357903f4fa804e3c |

The checksum is the **source distfile** sha256 taken from the package
template (`srcpkgs/gnome-software/template`, `checksum=`) — the pin on the
build input, not the digest of the built `.xbps`. Same convention as the cosmic
store repo's `PINS.md` and the discover store repo's.

The served index (`x86_64-repodata`) is signed with this store's own
repository signing key (RSA-4096, DER sha256
`9bdf45994696427ede0ca504674066e7276a0cadb6eccba8eb7cd4ae0be70d60`). The key
is never committed — only the public fingerprint travels: on the installer
side in `install_gnome_software_store()`'s `_gnome_software_key_expected`
literal and in the shared `_xbps_keyring_ours` allowlist, next to the COSMIC
(`83ffd154…`) and Discover (`4e40218a…`) store keys.

## Served artifacts

Latest release **`stores-r7`** (2026-11-06):

| artifact | value |
| --- | --- |
| served package | `gnome-software-50.2_10.x86_64.xbps` (2686454 B, `sha256 868a218f…26914`) |
| served package (devel) | `gnome-software-devel-50.2_10.x86_64.xbps` (324473 B, `sha256 6ceb2cc0…37b88`) |
| package signatures | `*.x86_64.xbps.sig2` (512 B each, RSA over the package's sha256) |
| served index | `x86_64-repodata` (2039 B, `sha256 87467bc5…ff98`) |
| index identity | `signature-by = Void Linux Stores <noreply@github.com>` |
| index URL | `https://bluesloth.github.io/void-linux-gnome-software/x86_64-repodata` |
| repository URL | `https://bluesloth.github.io/void-linux-gnome-software/` |
| release tag | `stores-r7` = `d4603890…` (commit `c113f67…`) |

Previous releases: **`stores-r6`** (2026-11-03) served `50.2_9`
(`adf2c10a…96ed`, index `5bc9939b…cc1b`); **`stores-r5`** (2026-11-01) served
`50.2_8` (`379d14be…aaf6`, index `88413789…bac7`); **`stores-r4`**
(2026-10-30) served `50.2_7` (`ed078a16…2220`, index `c9f450ca…a649`). The
`_7`–`_10` `.xbps` and `.sig2` files all remain at the repository root — build
numbers are not pruned — but the index lists only the newest build per package
name, so `_7`, `_8` and `_9` are history rather than current downloads.

`stores-r7` is the **`[UPDATE-NOTIFIER]`** release — template `revision=10`,
`gnome-software-50.2_10` (and its devel split), and it is **purely
packaging**: not one line of the monitor or the plugin is patched. Upstream
50.2's `src/gs-update-monitor.c` already polls for updates, throttles itself to
one notice a day through `org.gnome.Software update-notification-timestamp`,
and raises a `GNotification` titled *"Software Update Available"* whose default
action is `app.set-mode` → `updates`, so activating it opens the Updates page.
The only piece missing on a non-GNOME desktop was that nothing kept the
`org.gnome.Software` D-Bus name alive, so `GsApplication` was never running and
the monitor never started. `post_install()` now installs
`/etc/xdg/autostart/org.gnome.Software.desktop`
(`Exec=gnome-software --gapplication-service`, `NoDisplay=true`,
`X-GNOME-Autostart-enabled=true`, `X-GNOME-UsesNotifications=true`, and
deliberately **no `OnlyShowIn`**, so KDE and every other session pick it up).
The file list grows by exactly that one path (539 → 540) and the plugin md5 is
unchanged from `_9` (`d6da8c7b…6ed0`), so the §5 matrix verdicts carry over to
this build intact. The notification itself is unverified here — this box is
up to date (`xbps-install -nSu` empty), so proving it needs a synthetic
pending update. The same descriptor has not been carried to cosmic-store or
Discover yet (`docs/TODO.md`).

`stores-r6` is the **`[XBPS-REFRESH]`** release — template `revision=9`,
`gnome-software-50.2_9` (and its devel split). GNOME Software's update list
comes from `xbps-install -nSu`, which answers from the **cached** repodata;
nothing on a stock Void box ever syncs it, so `refresh_metadata_async` could
report a system as up to date indefinitely. The plugin now implements
`refresh_metadata_async`/`_finish` by running
`pkexec /usr/libexec/gnome-software-xbps-refresh` — a package-owned argless
helper, and a fourth polkit action `org.gnome.Software.xbps.refresh` with
`allow_active=yes` — then re-reads the update list. The helper path is
annotated rather than `xbps-install -S` because pkexec matches only the first
argument, so an `argv1`-based rule cannot distinguish a sync from an install.
The three transaction actions keep `auth_admin` and their `argv1` annotations.
The plugin binary changes (md5 `d6da8c7b…6ed0`, was `22268a79…888f`), so the
§5 feature matrix recorded for `_7`/`_8` does not carry over to the refresh
path.

This release also carries the `[PUBLISH-INDEX-ORDER]` tooling fix —
`tools/publish.sh` now resolves the newest build per pkgname with
`xbps-uhelper cmpver` instead of trusting a lexical `find` order. `stores-r6`
itself was not affected (`_9` sorts after `_8`), but the next revision would
have been; the same fix corrects `void-linux-discover`'s `stores-r4` index in
that repository's `stores-r5`. See `CHANGELOG.md` for both analyses.

The `50.2_7` build was published on **2026-10-28 without a release tag**, and
that publish rewrote `main` down to five files — the two packages, their two
`.sig2` and the index — dropping the packaging source, `tools/`, the docs,
`LICENSING.md` and `.nojekyll`. `stores-r4` (2026-10-30) restores them as a
**fast-forward commit on top of the published tip** (no force-push, no history
rewritten) and tags the result. **Not one served byte moved**, so the digests
above, the installer's store URL and its pinned fingerprint are unchanged.

The orphaned tags `stores-r1` → `a207b32…` and `stores-r3` → `bff1185…` were
deleted at the same time: neither is reachable from `main`, and an unreachable
tag keeps its commit's objects — including the ~269 MB of borrowed cosmic/KDE
`.xbps` blobs that the 2026-10-27 delete-and-recreate was meant to purge —
alive in GitHub's storage. `bff1185` remains described in `CHANGELOG.md` as
the rev6 release, and a bundle of the pre-restore history is kept outside the
repository.

The index lists only `gnome-software` and `gnome-software-devel` at `50.2_10`
(two entries, both local digests matching their index entries).
`gnome-software-devel` exists because the template's `-devel` subpackage is
generated by xbps-src; it is a build-time artefact and no end user needs it.
Build numbers are not pruned: the `_7`, `_8` and `_9` pairs stay at the
repository root alongside `_10` while the index advertises `_10` only (the
cosmic precedent). The `50.2_2`–`_6` rollback builds are not in this
repository.

## What the served package contains

- `usr/lib/gnome-software/plugins-23/libgs_plugin_xbps.so` — the xbps backend,
  md5 `d6da8c7b80c8df66f10cd08ef8016ed0`, sha256
  `977b090834a4bdde8661793d1b258e4ab5137b43cf088cf44bc827d419ad2964`.
  It differs from the `_8` plugin (`22268a79…`) by exactly the revision-9
  change: `refresh_metadata_async`/`_finish` are implemented by running the
  refresh helper and then re-reading the update list. That is a **behaviour
  change**, so the §5 matrix verdicts recorded for `_7`/`_8` carry over to
  everything except the refresh path and cold-start prompt behaviour, which
  are unverified until a GNOME box runs them. **rev10 does not touch the
  plugin**: the `_10` plugin is byte-identical to `_9`'s (same md5 and sha256
  above), so every rev9 verdict carries over unchanged.
- `etc/xdg/autostart/org.gnome.Software.desktop` — **new in rev10** (268 B,
  sha256 `384781c3…7b8a`, byte-identical to
  `srcpkgs/gnome-software/files/gnome-software-autostart.desktop`). Starts
  `gnome-software --gapplication-service` in every desktop session with no
  window and no prompt; it carries no `OnlyShowIn`, so KDE, XFCE and COSMIC
  sessions get the update monitor too. Its presence is the entire functional
  delta of this release: `_9` → `_10` adds exactly this one path (539 → 540
  files).
- `usr/libexec/gnome-software-xbps-refresh` — the argless helper the fourth
  polkit action annotates (4642 B, md5
  `428483f02cf2456b6c691d613796c7e8`, `sh -n` clean). It takes no arguments
  and can only ever run `xbps-install -S` (never `-Syu`, never `-y`, with
  stdin on `/dev/null` so a repository-key import question declines instead of
  silently trusting a new key), resolving `/usr/bin/xbps-install.real` first so
  a sync does not take a snapper snapshot pair through the
  `void-xbps-snap` shadow. It throttles itself to one sync per 600 s with a
  `/run` stamp (so it resets on reboot), exiting **75** (`EX_TEMPFAIL`) when it
  skips and the child's status otherwise. The caller treats any non-zero status
  as non-fatal but distinguishes 75 from a real failure so the log never claims
  a sync that did not happen.
- `usr/share/polkit-1/actions/org.gnome.Software.xbps.policy` — four actions
  now (`install` / `remove` / `update` / `refresh`), md5
  `effe371649aaf2c4222def0251f62a58`, sha256
  `8d5395c899b1058856bcf7b87d3a6ee04dcd77cbd1d89905879527e2b83ef9d0`, and
  byte-identical to `srcpkgs/gnome-software/files/gs-xbps.policy`. The three
  transaction actions stay `auth_admin` and keep their `argv1` annotations
  (`--yes` install, `--yes` remove, `--sync` update); `refresh` is
  `allow_active=yes` / `allow_any=no` / `allow_inactive=no` and annotates the
  helper path. Both `.policy` comments avoid `--` so the XML never parses it as
  a comment terminator. The `_8` policy was sha256 `88b69fbb…942e`.
- Package metadata: `pkgver = gnome-software-50.2_10`, `short_desc = GNOME
  Software Center (flatpak and xbps)`, `maintainer = Void Linux Stores
  <noreply@github.com>`.

## Verifying the served set

```sh
# package signature (RSA / PKCS#1 v1.5 over SHA256(package))
openssl dgst -sha256 -verify void-gnome-software-repo-signing.pub.pem \
    -signature gnome-software-50.2_10.x86_64.xbps.sig2 \
    gnome-software-50.2_10.x86_64.xbps
# -> Verified OK

# the signed index must embed the pinned public key
zstd -d < x86_64-repodata | tar -xO index-meta.plist   # public-key + signature-by
```

xbps does **not** sign the repository index: `xbps-rindex -s` embeds the
*public key* plus `signature-by` in the index's `index-meta.plist`, and the
client's trust anchor is that key's fingerprint in
`/var/db/xbps/keys/<hexfp>.plist`. The packages themselves are signed
individually (`.sig2`). So the value worth pinning is the **fingerprint**, which
is what the installer pins — and it recomputes cleanly from the key embedded in
the served index.

**Caution — repodata must be re-signed.** Any repack of this repository MUST
re-sign `x86_64-repodata` (2026-10-09 `void-cosmic-repo` incident: an unsigned
repack makes every client abort).
