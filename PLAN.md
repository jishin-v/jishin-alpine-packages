# Plan: package `kumo` (+ prerequisite `wpewebkit-kumo`)

Goal: ship two new aports under `incubating/` for the [kumo](https://git.sr.ht/~undeadleech/kumo)
Wayland mobile browser, on the `v3.24` branch. kumo depends on a forked WPE WebKit,
so we must add that first.

This is a DIY repo — standards can be relaxed (e.g. vendoring blobs in-repo is acceptable,
no need for perfect subpackage splits).

## Progress so far

- [x] **Commit 1 — switch branch v3.23 → v3.24.** Done (`6a7c327`). Changed
  `:repo-branch:` in `README.adoc`, `BRANCH` env and the CI container `image:` in
  `.github/workflows/ci.yml`. **Why:** v3.24/main ships `rust`/`cargo` 1.96.0; kumo's
  MSRV is `1.92.0` (edition 2024) and v3.23 only had 1.91.1.
- [x] **Bulk source repo scaffolded** — sibling repo at `../alpine_packages_bulk/`
  (commits `9878174`, `a3f3de3`). Git LFS for `*.tar.zst`; `.clones/` + `.staging/`
  gitignored. Hosts tarballs that `abuild` can't fetch directly.
- [x] **`wpewebkit-kumo` source tarball generated** (`alpine_packages_bulk/
  wpewebkit-kumo/wpewebkit-2.52.4.tar.zst`, 220M). **Reproducible** build
  (`--sort=name`, `--mtime=@<tag-commit-time>`, numeric owner/group) so the sha256 is
  stable across regenerations:
  `fd6091a4cb19e8428a2111dfb6280b39e8a793b4df135c604f9f6c8cee523c47` (77062 entries).
  Generator: `alpine_packages_bulk/wpewebkit-kumo/prepare-source.sh` (reuses a
  gitignored shallow clone across runs; prunes `LayoutTests Websites JSTests
  PerformanceTests ManualTests WebDriverTests WebKit.xcworkspace` per WebKit's
  top-level `CMakeLists.txt`).
- [x] **Commit 2 — `incubating/wpewebkit-kumo/APKBUILD`** (the WPE WebKit fork). Done
  (`17ba479`). Mirrors Alpine's own `community/webkit2gtk-4.1` recipe (clang+lld,
  samurai, `-U_FORTIFY_SOURCE -g1`), adapted to `PORT=WPE` + `ENABLE_WPE_PLATFORM=ON`.
  `source=` points at the bulk LFS tarball; full SPDX license list. **Bulk repo pushed**
  at `jishin-v/jishin-alpine-repository-bulk` (branch `master`); raw URL verified live
  (HTTP 200, LFS redirect).
- [x] **Commit 3 — `incubating/kumo/APKBUILD`** (the browser). Done (`3f96df2`).
  **No vendoring**: `cargo vendor` is unworkable for kumo because Servo + the
  `[patch.crates-io]` on `servo_arc` resolve two different stylo revs to the same
  `servo_arc` version, which vendor rejects ("duplicate version ... from two sources").
  Instead follows Alpine's `community/helix` idiom: `options="net !check"` +
  `cargo fetch --target="$CTARGET" --locked` (prepare) + `cargo auditable build --frozen
  --release` (build). Both engines enabled (upstream default) → runtime per-tab switching.
  Source is the fetchable sourcehut tarball (sha512 pinned); desktop/icon/metainfo
  installed from in-tree files. No bulk-repo asset needed for kumo.

### Resumable CI builds (local ccache + tarball on sshfs) — DONE
WebKit's ~9.4k translation units don't finish inside GitHub's 6h job limit on a
4-core runner. ccache makes the build resumable, BUT the cache must live on a
**fast local disk during the build**. A first attempt put `CCACHE_DIR` directly
on the sshfs mount, which made the build *degenerate*: every hit is several
serial SFTP round-trips, so a run that had already timed out once spent almost
its whole 6h budget replaying ~8k hits and compiled only ~700 new objects
(7922 → 8607); each successive run covers *less* ground and it never converges.

Current scheme: during the build `CCACHE_DIR=/tmp/ccache` (local runner SSD);
the `wpewebkit-kumo` APKBUILD persists it as a compressed tarball on the remote
mount between runs. All of this lives in the APKBUILD's `build()` and is opt-in
via env that CI sets, so:
- **Zero waste when not building it** — `buildrepo` skips a package whose apk is
  already in `REPODEST`, so `build()` (and thus the tarball download/upload)
  never runs for an already-published wpewebkit-kumo. Bumping `pkgrel` naturally
  invalidates this and re-triggers a build.
- **Scoped to the package** — kumo / jishin-dummy have none of this code.

Env (CI → APKBUILD): `CCACHE_DIR` (local), `CCACHE_PERSIST_DIR` (remote mount dir
for the tarball), `CCACHE_BUILD_BUDGET` (inner time cap, seconds). Mechanics
(helper functions in the APKBUILD):
- `_ccache_restore` (start of `build()`): download+extract the tarball into the
  local dir; a one-time migration stream-copies the old raw `/.ccache` dir if the
  tarball isn't there yet, so the ~8k objects already cached aren't lost.
- inner `timeout $CCACHE_BUILD_BUDGET ninja -C build` (4h): signals ninja directly
  so it tears down its compile children (a SIGTERM to `cmake --build` would orphan
  them and keep mutating the cache mid-upload). Well under the 6h job limit.
- `_ccache_save` (after the build, **always**, even on rc=124 timeout): re-tars
  the local cache and atomically publishes on the remote (temp file + SFTP
  posix-rename). Skipped automatically when the cache fingerprint is unchanged
  (all-hits run / early configure failure) so identical data isn't re-uploaded.
  Stale `.tmp.*` uploads from a crashed run are cleaned on the next pass.
A poller prints `ccache -s` every 15 min for live visibility. With local hits the
build now converges: a timed-out run caches ~6k objects, the next replays those in
minutes and finishes. If the remote is ever flaky, the whole tarball dance is
gated on `CCACHE_PERSIST_DIR` being set, so local `abuild` is unaffected.

### Source hosting (wpewebkit-kumo: RESOLVED)
The `chrisduerr/WebKit` fork's tarball is committed to the **`_bulk` LFS repo**
(`jishin-v/jishin-alpine-repository-bulk`, branch `master`) and the APKBUILD `source=`
points at the raw LFS object, verified live:
`https://github.com/jishin-v/jishin-alpine-repository-bulk/raw/refs/heads/master/wpewebkit-kumo/wpewebkit-2.52.4.tar.zst`
(kumo itself needs no bulk asset — it uses the sourcehut tarball + network fetch.)
(GitHub serves the real LFS content at the raw URL). Top-level dir inside the tarball
is `wpewebkit-2.52.4/`, so `builddir="$srcdir/wpewebkit-2.52.4"`.

**TODO before CI builds:** push this aports repo (`staging` is 5 commits ahead of
`origin/staging`) so the workflow runs `buildrepo` over `incubating/`. Expect to
iterate on missing makedepends (esp. Servo's mozjs/aws-lc-sys C deps) and musl quirks.

---

## Key facts discovered (read these before resuming)

### kumo (the browser)
- Source tarball (sourcehut, **fetchable**): `https://git.sr.ht/~undeadleech/kumo/archive/v1.8.2.tar.gz`
- sha256 of tarball: `afcf72768852b9a04e6319c482a79619b02a8e6331579c65c5d546f6407cfa05`
- Extracts to dir `kumo-v1.8.2`. License `GPL-3.0-only`.
- Local copy staged at `/tmp/kumo-investigation/kumo-v1.8.2/`.
- **All UI/data assets are `include_bytes!`/`include_str!`-embedded** into the binary
  (`adblock.json`, `tlds.txt`, GLSL shaders in `shaders/`, SVGs in `svgs/`,
  `internal_pages/error.html`). → No `/usr/share/kumo` data dir needed at runtime.
- **`gir-files` git submodule is empty & NOT needed** — the `*-sys` crate bindings are
  pre-generated (committed under `wpe-*-sys/src/auto/`); their `build.rs` files only do
  `system-deps` probing at build time.
- `Cargo.lock` is shipped → vendor-friendly.
- Desktop integration files all ship in-tree: `Kumo.desktop`, `logo.svg`,
  `org.catacombing.kumo.metainfo.xml`.
- **Two engines, both default-on** in `Cargo.toml`: `default = ["servo", "webkit"]`.
  - Runtime **per-tab engine switching** is a first-class feature gated on
    `cfg(all(feature="servo", feature="webkit"))`: context menu offers
    "Reload in WebKit"/"Reload in Servo" → `State::switch_engine` in `src/window.rs:457`.
    Choice is sticky per-URI and persisted to SQLite. Default engine is configurable in
    `~/.config/kumo/config.toml` (`[engine] default = "WebKit"|"Servo"`).
  - **We want to KEEP Servo enabled** per user decision → must vendor the git forks below.
- Cargo **git-fork deps** to vendor (from `Cargo.lock` + `[patch.crates-io]`):
  - `servo` ← `github.com/chrisduerr/servo` @ `ebaed340944436dc8807720056774a8cdbf2a967`
  - `stylo` ← `servo/stylo` @ `49e912cf401a7f867d33d61baa2389bb3d6f73e0`
  - `servo_arc` ← patched to `servo/stylo` @ `a556f4cbd15fc289039261661b049a5dc845cd80`
  - `keyboard-types` ← `chrisduerr/keyboard-types` @ `03b47d33d345b35671076bcd8ce4533250997c72`
- Arch: `x86_64 aarch64`.

### wpewebkit-kumo (the WebKit/WPE fork — prerequisite aport)
- Upstream: `https://github.com/chrisduerr/WebKit`, tag **`v2.52.4`**, tag object sha
  `b13d1f54c40ffa2a34c628ab60f8e5467d58150a`.
- ⚠️ **GitHub blocks ALL archive/tarball generation for this repo** with HTTP
  `422 "Content creation is blocked"` — verified across `archive/`, `codeload.github.com`,
  and the API `tarball` endpoint (the catacomb PKGBUILD uses `git+$url#tag=…` precisely
  for this reason). **abuild has no native git source fetcher** → resolved by mirroring a
  reproducible tarball in the `../alpine_packages_bulk` LFS repo (see "Source hosting"
  above and `prepare-source.sh`).
- ⚠️ **Heavy source**: full clone ~7.4G; pruned tarball is ~220M (77062 entries).
- Reference recipe: catacomb `arch/wpewebkit-kumo/PKGBUILD` (was cloned at
  `/tmp/kumo-investigation/catacomb/arch/wpewebkit-kumo/PKGBUILD`; **NOTE: `/tmp` is
  ephemeral and was cleared — re-clone `https://git.sr.ht/~undeadleech/catacomb_packages`
  if the PKGBUILD is needed again**). Key CMake flags:
  `-D PORT=WPE -D CMAKE_BUILD_TYPE=Release -D CMAKE_INSTALL_PREFIX=/usr
   -D CMAKE_INSTALL_LIBDIR=lib -D CMAKE_INSTALL_LIBEXECDIR=lib -D CMAKE_SKIP_RPATH=ON
   -D USE_LIBBACKTRACE=OFF -D ENABLE_DOCUMENTATION=ON -D ENABLE_MINIBROWSER=OFF
   -D ENABLE_WPE_PLATFORM=ON -D ENABLE_SPEECH_SYNTHESIS=OFF`, with `CC=clang CXX=clang++`
  and `-fuse-ld=lld`; also `-fcf-protection=none` and downgrade `_FORTIFY_SOURCE=3→2`.
- `provides="wpewebkit libWPEWebKit-2.0.so libWPEPlatform-2.0.so"`,
  `conflicts="wpewebkit"` (Alpine has no `wpewebkit` apk, but keep for safety).

### v3.24 dependency availability (verified against APKINDEX)
All present in v3.24 main/community: `rust`/`cargo` 1.96.0, `clang20` + `lld20`,
`ninja-build` (1.13.2), `samurai`, `cmake`, `gperf`, `ruby`, `gi-docgen`, `unifdef`,
`hyphen`, `wayland-protocols`, `libwpe-dev` (1.16.3), `libwpebackend-fdo-dev` (1.16.1),
`libsoup3-dev`, `glib-dev`, `cairo-dev`, `pango-dev`, `librsvg-dev`, `libdrm`,
`libinput`, `libxkbcommon-dev`, `mesa-dev`/`libepoxy-dev`, `libseccomp-dev`,
`xdg-dbus-proxy`, `bubblewrap`, `harfbuzz-icu`, `lcms2-dev`, `libavif-dev`,
`libjxl-dev`, `openjpeg-dev`, `woff2-dev`, `libxml2-dev`, `libxslt-dev`, `sqlite-dev`,
`gobject-introspection`, gst stack (`gst-plugins-{base,good,bad}`, `gst-libav`) 1.28.3.
`at-spi2-core`/`atk` exist under names `libatk-1.0` / `libatk-bridge-2.0` (use those).

### Build target / libc
- **musl** (normal Alpine package). Do NOT attempt glibc. `*-unknown-linux-musl` is fine;
  Servo-on-musl risk is "patch whatever crates complain", not "switch libc".

---

## Commit 2 — `incubating/wpewebkit-kumo/APKBUILD`

### Source tarball (DONE)
Produced by `../alpine_packages_bulk/wpewebkit-kumo/prepare-source.sh` and committed
(via LFS) in the `_bulk` repo. Pinned values for the APKBUILD:
- top-level dir inside tarball: `wpewebkit-2.52.4/`
- sha256: `fd6091a4cb19e8428a2111dfb6280b39e8a793b4df135c604f9f6c8cee523c47`
- size: 230174631 bytes
- `source=` URL (once `_bulk` repo is pushed): `https://raw.githubusercontent.com/jishin-v/jishin-alpine-packages_bulk/main/wpewebkit-kumo/wpewebkit-2.52.4.tar.zst`

### APKBUILD shape (mirror catacomb PKGBUILD)
- `pkgname=wpewebkit-kumo`, `pkgver=2.52.4`, `pkgrel=0`.
- `url="https://github.com/chrisduerr/WebKit"`, `arch="aarch64 x86_64"`.
- `license=` → copy the SPDX list from the catacomb PKGBUILD (AFL-2.0 OR GPL-2.0-or-later,
  BSD-3-Clause, LGPL-2.1-or-later, MIT, MPL-2.0, ICU, …) — or simplify to `BSD-3-Clause
  AND LGPL-2.1-or-later` for DIY; user is OK relaxing this.
- `makedepends=` → `clang20 lld20 cmake ninja-build gperf ruby gi-docgen unifdef
  gobject-introspection-dev glib-dev libsoup3-dev libwpe-dev libwpebackend-fdo-dev
  wayland-protocols-dev libdrm-dev libinput-dev libxkbcommon-dev libepoxy-dev
  mesa-dev harfbuzz-icu harfbuzz-dev lcms2-dev libavif-dev libjxl-dev openjpeg-dev
  woff2-dev libseccomp-dev libtasn1-dev sqlite-dev libxml2-dev libxslt-dev
  cairo-dev pango-dev at-spi2-core-dev gst-plugins-base-dev gst-plugins-bad-dev
  bubblewrap xdg-dbus-proxy hyphen` (translate Arch names → Alpine apk names).
- `depends=` (runtime) → `libwpe libwpebackend-fdo libsoup3 glib libtasn1` + the usual
  `.so` providers; can keep lighter and let kumo pull the gst stack.
- `subpackages="$pkgname-dev $pkgname-doc"` (doc is optional; can drop to simplify).
- `provides="wpewebkit libWPEWebKit-2.0.so libWPEPlatform-2.0.so"`,
  `conflicts="wpewebkit"`, `provider_priority=100`.
- `build()`: set `CC=clang CXX=clang++`, `LDFLAGS="$LDFLAGS -fuse-ld=lld"`,
  apply `_FORTIFY_SOURCE` and `-fcf-protection=none` tweaks, then `cmake -S -B build
  -G Ninja <flags>` + `cmake --build build`.
- `package()`: `DESTDIR="$pkgdir" cmake --install build`; collect license files like
  the PKGBUILD does, or just `install -Dm644 ...` a couple of COPYING files.
- WebKit builds are **long**; first CI run will be slow. Expect to iterate on missing
  `-dev` deps / musl-specific CMake failures.
- Suggested commit message: `Add wpewebkit-kumo aport (chrisduerr fork v2.52.4)`.

---

## Commit 3 — `incubating/kumo/APKBUILD`

### Vendoring
Both engines on → must vendor crate graph **including the 4 git forks**. Mirror the
same approach as wpewebkit-kumo: generate the vendor tarball via a `prepare-source.sh`
in `alpine_packages_bulk/kumo/` (gitignored persistent clone/cargo-home reused across
runs), commit it to the **bulk** repo via LFS, and point the APKBUILD `source=` at the
raw GitHub LFS URL. The kumo source tarball itself is fetchable directly from sourcehut,
so **only the vendor tarball** needs mirroring.

**The vendor tarball is arch-independent.** `cargo vendor` (no `--target`) vendored the
*full* graph including all target-conditional deps (verified empirically: it pulled in
aarch64-only and even windows-only crates from a synthetic project). `cargo vendor`
doesn't even accept `--target` (it errors). So a single tarball covers x86_64 + aarch64;
arch only matters at compile time, which runs natively per-arch in CI.

Sketch:
```sh
# in alpine_packages_bulk/kumo/prepare-source.sh (reuses .clones/kumo)
git clone --depth 1 --branch v1.8.2 https://git.sr.ht/~undeadleech/kumo .clones/kumo
cd .clones/kumo
cargo fetch --locked          # populate registry cache (no --target needed)
cargo vendor vendor           # vendored full graph, all targets
# .cargo/config.toml emitted by `cargo vendor`; ship it alongside the vendor dir
tar --zstd -cf ../../kumo/kumo-1.8.2-vendor.tar.zst vendor
```
(`cargo vendor` also prints the `[source.crates-io] replace-with = "vendored-sources"`
config snippet; bake it into the APKBUILD or ship a `.cargo/config.toml`.)

### APKBUILD shape
- `pkgname=kumo`, `pkgver=1.8.2`, `pkgrel=0`.
- `pkgdesc="Wayland mobile web browser (WebKit + Servo engines)"`,
  `url="https://git.sr.ht/~undeadleech/kumo"`, `arch="aarch64 x86_64"`,
  `license="GPL-3.0-only"`.
- `source=("https://git.sr.ht/~undeadleech/kumo/archive/v$pkgver.tar.gz"
  "vendor.tar.gz"
  <cargo config or a sed patch to set vendor path>)`
- `builddir="$srcdir/kumo-v$pkgver"`.
- `makedepends="cargo rust wpewebkit-kumo (or -dev) glib-dev libsoup3-dev cairo-dev
  pango-dev librsvg-dev libxkbcommon-dev mesa-dev sqlite-dev libdrm-dev wayland-dev
  clang20 lld20"` (kumo links libWPEWebKit/libWPEPlatform + gl + sqlite + glib/soup).
- `depends="wpewebkit-kumo gst-plugins-base gst-plugins-good gst-plugins-bad gst-libav
  ttf-dejavu"` (catacomb uses `ttf-font`; pick a concrete font apk).
- `options="!check"` (no real test harness — `cli.rs` tests are toml-parsing only;
  cargo test is possible but likely needs a display/Wayland). If we want check(),
  `cargo test --locked --offline` but expect some to need a compositor.
- `export CARGO_HOME="$srcdir/cargo"`, set `CARGO_BUILD_TARGET` if cross, `--frozen --offline`.
- `build()`: `cargo build --release --frozen --offline` (default features → both engines).
  Catacomb sets `CARGO_INCREMENTAL=0`.
- `package()`:
  - `install -Dm755 target/release/kumo "$pkgdir"/usr/bin/kumo`
  - `install -Dm644 Kumo.desktop "$pkgdir"/usr/share/applications/org.catacombing.kumo.desktop`
  - `install -Dm644 logo.svg "$pkgdir"/usr/share/icons/hicolor/scalable/apps/Kumo.svg`
  - `install -Dm644 org.catacombing.kumo.metainfo.xml
    "$pkgdir"/usr/share/metainfo/org.catacombing.kumo.metainfo.xml`
- Suggested commit message: `Add kumo aport v1.8.2 (WebKit + Servo)`.

---

## Verification (after both commits)
1. `git push` → CI (`.github/workflows/ci.yml`) runs `buildrepo` over `incubating/`.
2. CI builds **alphabetically / by dependency** — `kumo` lists `wpewebkit-kumo` in
   makedepends/depends so buildrepo should order it; if not, may need to land
   wpewebkit-kumo first in a separate push.
3. Expect first runs to fail on: missing `-dev` apks, musl-specific patches (Servo or
   WebKit), cmake flag translation. Iterate. `buildrepo -d ... --keep-going` helps
   (the `/keep-going` commit-trailer already enables it).
4. Local check tooling: `abuild -r` in an Alpine v3.24 container/chroot if available.

## Things deliberately deferred
- GPG signature verification of the WPE fork source (catacomb uses `validpgpkeys`
  `4DAA67A9EA8B91FCC15B699C85CDAE3C164BA7B4`); skip for DIY unless requested —
  especially since we're mirroring the tarball ourselves.
- Splitting gst plugins into `-media` subpackage; just `depends=` them for now.
- `-doc`/`-dbg` subpackage polish.
- Backporting to v3.23 (not wanted; we moved to v3.24).
