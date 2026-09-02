# Plan: package the Quill OS userspace (PorQ-Pine) for postmarketOS

Goal: ship aports under `incubating/` for the userspace components of
[Quill OS](https://github.com/PorQ-Pine) — the Fedora-based PineNote distro — so
they can run on postmarketOS (Alpine/musl, OpenRC) on the Pine64 PineNote.
Research notes below were gathered by reading `quillstrap` (their installer /
build orchestrator) and cloning the repos into `../clone`.

Quillstrap's `quillstrap/src/things/` is effectively the distro's package list;
each module there builds + deploys one component.

## Component inventory

### Board level — NOT for us (pmOS owns kernel/bootloader)

| Repo | What | Why skip |
|---|---|---|
| `u-boot-pinenote` | U-Boot for PineNote | pmOS device pkg territory |
| `rkbin` | Rockchip bootloader blobs | idem |
| `firmware` | PineNote binary blobs | idem |
| `kernel` | hrdl's tree with the `rockchip_ebc` e-ink driver | pmOS device pkg territory; userspace here assumes the ebc driver exists |
| `initrd` | dracut initramfs bits | pmOS has its own initramfs |
| `eink-kernel-magic` | 2-script copy of hrdl's `wbf_to_custom.py` (wavefile→custom waveform) | dev tool; revisit if we ever need custom waveforms |
| `quillstrap` | the installer/bootstrap tool itself | not needed post-install |
| `netboot-host` | netboot server bits | n/a |
| `alpine-chroot-install` | pin of the chroot helper, used by quillstrap | n/a |

### Quill boot/session machinery — skip for pmOS (v1)

- **`quill-init` (qinit)** — Slint (backend-linuxkms-noseat, software renderer)
  GUI init that *replaces* early boot: mounts, overlay rootfs, netboot,
  QR-code WiFi setup, then hands off to systemd (`init_wrapper` feature execs
  `/sbin/init`). GPL-3.0-only. Deeply tied to Quill's boot flow; conflicts with
  the pmOS initramfs + OpenRC story. Skip.
- **`qoms`** — session orchestrator: drives greetd through a custom
  `libquillcom` + postcard socket protocol, login/power threads. Deployed as a
  systemd unit to `/opt/qoms` (unit file in `qoms/other/qoms.service`),
  greetd's `default_session.command` points at `/opt/qoms/get_sock.sh`. Tied to
  their qinit sockets + systemd. Skip for v1 (pmOS users use their own
  greeter/autologin; greetd itself is in Alpine community if ever needed).
- **`core-settings`** — Slint KMS settings UI (+ `libcoresettings`); talks to
  qinit's socket via `libqinit` path dep. Useless without qinit. Skip for v1.
- **`Write`, `xournalpp` forks** — both are pins of stock upstream
  (xournalpp fork = untouched upstream 1.2.7; verified via git log). Alpine
  already ships `xournalpp` + `koreader`. No Quill patches to carry.
- **`eww`, `lisgd`, `squeekboard` forks** — pins of upstream + tiny fixes
  (squeekboard fork = upstream + "Make squeekboard only listen to force
  events"). Alpine ships all three in community/testing; carry the one-commit
  squeekboard patch only if the stock one misbehaves.

### The userspace value — package these

| Component | What it does | Build inputs / notes |
|---|---|---|
| **`pinenote-service`** (PorQ fork of phantomas's sr.ht project, Cargo version 1.0.1, GPL-3.0-or-later) | D-Bus daemon `org.pinenote.PineNoteCtl` wrapping the `rockchip_ebc` driver: refresh modes, render hints (Y4/Y2/Y1, threshold/dither, redraw), dither mode, global refresh, suspend/off-screen image, EBC ioctls. Compositor bridges: `sway` (sway-ipc), generic D-Bus `HintMgr1`, and Quill's `quill-niri` (niri-ipc) bridge. Upstream ships a systemd *user* unit + D-Bus activation file in `packaging/resources/` | `zbus`, `nix`, `tokio`, `nalgebra`, `image`. Default features `["bridges", "quill-niri"]`; `quill-niri` pulls path deps `niri-ipc` (their niri fork), `quill-data-provider-lib`, `qoms_lib`, `inotify`. `--no-default-features --features bridges` = zero path deps. Needs OpenRC service and/or D-Bus activation (user session), or spawn from compositor config |
| **`quill-data-provider`** | Aggregates battery, backlight (cool/warm), BT, network, volume, media player, dunst, gestures, settings-menu, virtual-keyboard state and the **e-ink window-hint state** (window geometry via niri-ipc + inotify) and serves it over a socket in eww's `deflisten` var protocol. Repo contains 5 crates: `quill-data-provider` (daemon), `eww-data-requester` (eww-side client), `eink-window-settings` (egui/eframe tool), `quill-data-provider-lib`, `enums` | `niri-ipc` path dep (`../../niri/niri-ipc/`); egui stack for the settings tool. Spawned by niri `spawn-sh-at-startup` |
| **`eww-niri-toolbar`** | Small Rust bin: niri workspaces/windows + freedesktop icons → JSON for an eww taskbar widget (example yuck in its README) | `niri-ipc` path dep (`../niri/niri-ipc/`), trivial |
| **`eww-config`** | The actual Quill shell: bar, quick-settings/control center, power menu, calendar, screen settings (yuck + scss, data only). Consumes quill-data-provider + eww-niri-toolbar; controls squeekboard | noarch; install to `/usr/share/eww/quill` (+ skel symlink or docs) |
| **`Cosmic-Wanderer`** | Slint app launcher: desktop-entry scan, fuzzy match, notifications, niri integration, config file. Ships `cosmic-wanderer-opener` helper subcrate | slint with **backend-qt** → needs Qt6 dev headers at build time; `niri-ipc` path dep (`../niri/niri-ipc/`) |
| **`orbit`** (fork of `LifeOfATitan/orbit` + 1 fix commit) | GTK4 + layer-shell WiFi/Bluetooth/VPN manager over NetworkManager + BlueZ, high-contrast UI, theme sync | `gtk4`, `gtk4-layer-shell`, `zbus`, `reqwest`. pmOS uses NetworkManager on most devices → fits |
| **`procedural-wallpapers-rs`** | CLI generator for (e-ink friendly) procedural wallpapers; `procedural_wallpapers` + `wallpapers` lib workspace | `image`, `rand`, `clap`; trivial |
| **`rootfs-configs`** | niri `config.kdl` for the PineNote (spawns `quill-data-provider`, `cosmic-wanderer`, `swaybg`, `eww bar`, `orbit daemon`, `squeekboard` at startup), udev rules (`92-rockchip-ebc.rules`, backlight perms, CPU perms), fontconfig, greetd config, skel configs | cherry-pick into a noarch `quill-configs-pinenote`; the systemd bits stay out |
| `niri` fork | ≈ upstream main post-26.04 + tiny PorQ commits ("Add styluslabs Write to cursorless apps list", smithay bumps) | Alpine already ships niri (community). The fork is needed **only as a source tarball for `niri-ipc`** |
| `slint`, `smithay`, `usvg` forks | build-dep pins of upstream libs | irrelevant: vendored automatically by cargo |

## Structural findings (packaging strategy)

1. **Distributed monorepo via path deps.** The crates depend on each other with
   `path =` entries that resolve inside quillstrap's checkout layout:
   `os/low/{pinenote_service,qoms}`, `os/gui/{niri,quill_data_provider,
   Cosmic-Wanderer,eww_niri_toolbar,eww}`, `common/libquillcom`,
   `init/{quill_init,core_settings}`. Aports that need those deps must fetch
   the sibling repos as extra `source=` tarballs and **reconstruct the layout
   in `prepare()`** (same idiom as charon's gitlink submodules), after which
   `cargo fetch --locked` + `cargo auditable build --frozen` work normally —
   every repo ships a `Cargo.lock`.
2. **`niri-ipc` from the fork ≠ crates.io 26.4.0.** The fork's `niri-ipc`
   carries unreleased-upstream API (`WindowGeometries` response +
   `WindowGeometry` struct, `MaxBpc` action/enum) that quill-data-provider's
   window→hint logic uses (verified by diffing against the published crate).
   So the path deps can't be sed'd to crates.io; fetch the PorQ niri fork
   tarball as a build-time source. Runtime niri stays Alpine's stock package.
3. **No tags/releases anywhere** (checked all repos) → pin exact commit SHAs,
   `source=` via `https://codeload.github.com/PorQ-Pine/<repo>/tar.gz/<sha>`,
   pkgver from the crate's Cargo.toml version + `_gitYYYYMMDD` style suffix.
4. **systemd assumptions are light** in the Tier-1 daemons (they're designed
   to be spawned by niri config / systemd user units); for pmOS write OpenRC
   initds and/or D-Bus user-session activation files, or just document the
   `spawn-at-startup` lines for the user's niri config.
5. **musl/glibc**: developed on Fedora/glibc, but Tier 1/2 deps are
   musl-friendly (pure Rust + zbus/dbus, gtk4-rs, slint software renderer).
   Watch out for slint `backend-qt` (Cosmic-Wanderer) needing Qt6 at build.
6. **Pinned commits at research time** (default branch `main`, HEAD):
   - `pinenote-service` d19573beb2 2026-06-25
   - `quill-data-provider` a43b3b921c 2026-07-01
   - `libquillcom` c8fbf1de1f 2026-01-03
   - `qoms` 55b13d62f2 2026-06-28
   - `Cosmic-Wanderer` 5a8a3563a5 2026-04-19
   - `eww-niri-toolbar` 51e01ee454 2026-02-05
   - `orbit` 165e6bb3e6 2026-08-27
   - `procedural-wallpapers-rs` (not pinned yet)
   - `squeekboard` c50af7b92b 2025-12-06 (= upstream + force-events commit)
   - `niri` (fork) 0e30f33bc5 2026-06-08
   - `xwayland-satellite` 9dbfca5b3a 2026-06-29 (Alpine ships stock anyway)

## Alpine/pmOS availability (checked on pkgs.alpinelinux.org, edge)

Already packaged → do NOT duplicate (only patch/fork if a real fix is needed):

- community: `niri`, `squeekboard`, `greetd`, `lisgd`, `xwayland-satellite`,
  `koreader`, `alacritty`, `dunst`, `xournalpp`
- testing: `eww`

Not in Alpine (our opportunity): `pinenote-service`, `quill-data-provider`,
`eww-niri-toolbar`, `eww-config`, `cosmic-wanderer`, `orbit`,
`procedural-wallpapers-rs`, `quill-configs-pinenote`.

## Packaging plan

### Tier 1 — the pmOS-worthy core (EBC control + shell data layer)

1. **`pinenote-service`** — start **bridges-only**
   (`--no-default-features --features bridges`): zero path deps, cleanest
   first aport (sway bridge + generic D-Bus HintMgr1 bridge). OpenRC service
   (`pinenote-service` user daemon) + install upstream's D-Bus activation
   file. Once quill-data-provider is in, bump to default features to enable
   the `quill-niri` bridge (fetch niri + quill-data-provider + qoms tarballs,
   reconstruct layout, path deps resolve).
2. **`quill-data-provider`** — daemon + `eww-data-requester` (+ subpackage),
   `eink-window-settings` (+ subpackage, egui). Needs the niri fork tarball
   for `niri-ipc`. Started by niri config or an OpenRC user service.
3. **`eww-niri-toolbar`** — trivial cargo aport, needs `niri-ipc` tarball.
4. **`eww-config`** — noarch, data-only (yuck/scss/svg); install to
   `/usr/share/eww/quill/`, document `eww --config … open bar` or ship a
   wrapper. Depends on: `eww`, `quill-data-provider-eww-requester`,
   `eww-niri-toolbar` (soft).

### Tier 2 — apps

5. **`cosmic-wanderer`** (+ `cosmic-wanderer-opener` subpkg) — slint
   backend-qt → `makedepends` Qt6 headers; `niri-ipc` tarball needed.
6. **`orbit`** — gtk4-rs + layer-shell; straightforward.
7. **`procedural-wallpapers-rs`** — trivial.

### Tier 3 — garnish

8. squeekboard "force events" patch vs Alpine's squeekboard (only if needed).
9. **`quill-configs-pinenote`** (noarch) — cherry-pick from `rootfs-configs`:
   udev rules (`92-rockchip-ebc.rules`, `90-backlight-perms`, `91-cpu-perms`),
   PineNote niri `config.kdl` sample, fontconfig `local.conf`. No systemd
   bits, no greetd config (pmOS session story differs).
10. Optional metapackage `quill-os-pinenote` pulling the whole stack.

### Explicitly deferred / skipped

- `qoms`, `quill-init`, `core-settings` (session/boot machinery; revisit
  `core-settings` only if we ever adopt qinit).
- kernel/u-boot/firmware/rkbin/initrd/netboot/quillstrap (board level).
- `Write`/`xournalpp`/`koreader`/`eww`/`lisgd`/`xwayland-satellite` forks
  (stock Alpine packages suffice; forks carry no must-have patches).
- `slint`/`smithay`/`usvg` forks (cargo-vendored automatically).

## Open questions / decisions to make as we go

- pinenote-service: bridges-only first vs jumping straight to default
  features (full niri bridge) — leaning bridges-only for the first CI run.
- Where the daemons live in a pmOS session: OpenRC user services vs D-Bus
  activation vs purely compositor `spawn-at-startup`. The Quill way is the
  latter; OpenRC purists may want initds.
- `eink-window-settings` uses egui/eframe — heavy for what it does; keep as
  subpackage, possibly `!check` + only build if egui behaves on musl.
- eww itself is only in Alpine *testing* — fine for a DIY repo, but our
  `eww-config` consumers need it built into this repo or pulled from testing.

---

## Execution log (2026-09-02)

### Aports created (incubating/)

| aport | version | notes |
|---|---|---|
| `pinenote-service` | 1.0.1_git20260625 | DEFAULT features (quill-niri bridge), full os/{low,gui} layout reconstruction (pinenote-service + qoms + niri + quill-data-provider tarballs); unlocked `cargo fetch` because the committed lock predates the quill-data-provider-lib egui 0.33→0.34 bump; D-Bus activation file installed with `SystemdService=` stripped |
| `quill-data-provider` | 0.1.0_git20260701 | 3 of the repo's 5 crates: daemon (main), `eww-data-requester` + `eink-window-settings` subpackages; niri symlink reconstruction; `--locked` fetch works (own lock is internally consistent) |
| `eww-niri-toolbar` | 0.2.0_git20260205 | bin `eww-niri-taskbar`; niri symlink; unlocked fetch (lock pins niri-ipc 25.11.0 + serde_json 1.0.145 < fork's floors) |
| `eww` | 0.6.0_git20260111 | PorQ fork of elkowar/eww master (2026-01-11, post-0.6.0 + "Fix icon DPI"); wayland-only feature set; master lock is current so no Alpine-style lock-bump patch needed |
| `cosmic-wanderer` | 0.1.0_git20260419 | slint backend-qt (qt6-qtbase-dev/qmake6, clang), `cosmic-wanderer-opener` subpackage (standalone nested crate); one-line Cargo.lock sed niri-ipc 25.11.0→26.4.0 keeps `--locked` (its serde_json 1.0.149/serde 1.0.228 already satisfy the fork) |
| `orbit` | 2.4.6_git20260827 | gtk4-rs + layer-shell; aws-lc-sys via cmake/perl (dirge-style makedepends) |
| `procedural-wallpapers-rs` | 0.1.0_git20251218 | bin `procedural_wallpapers`; trivial |
| `eww-config` | 20260625 (noarch) | data-only to /usr/share/eww/quill + `quill-eww` wrapper (`eww --force-wayland --config …`) |
| `quill-configs-pinenote` | 20260813 (noarch) | udev rules (backlight/cpu/rockchip-ebc perms), fontconfig conf.avail snippet, sample niri config.kdl, orbit configs, wallpaper, README.alpine |
| `quill-os-pinenote` | 1 (noarch) | metapackage pulling the stack + niri/swaybg/squeekboard/dunst/adwaita-fonts/font-noto/papirus-icon-theme |

### Decisions taken vs the plan's open questions

- **pinenote-service features**: went straight to default features (quill-niri
  bridge) instead of bridges-only — the bare `bridges` feature compile_error!s
  without a concrete bridge, and building sway-only still requires fetching the
  path-dep tarballs anyway (the lock contains their entries), so there was
  nothing to save. Runtime bridge = niri, matching the stack we ship.
- **Cargo.lock drift**: the pinned repos were never --locked-consistent with
  each other (quillstrap builds unlocked). Aports follow the split: repos whose
  own lock matches the pinned siblings keep `--locked` fetch + `--frozen`
  build; pinenote-service and eww-niri-toolbar do an unlocked `cargo fetch` in
  prepare() (lock refreshed on disk) and then build `--frozen`. cosmic-wanderer
  takes a one-line lock sed (niri-ipc version; path dep → no checksum involved).
- **Session daemons**: no OpenRC initds (they are session/user processes);
  pinenote-service gets D-Bus session activation, quill-data-provider is
  compositor-spawned (documented in quill-configs-pinenote's README + sample
  niri config).
- **eww source**: PorQ fork master commit instead of v0.6.0+Alpine's
  update-cargo-lock.patch — the fork is what Quill actually runs (icon DPI
  fix) and its lock is current.
- **Physical `..` gotcha**: the builddir for pinenote-service is MOVED into
  the reconstructed os/ layout (not symlinked) — path deps walk `../..` and
  `..` resolves physically, so a symlinked crate dir would escape the layout.
  The niri/qoms/quill_data_provider links are fine as *targets*.

### CI iteration history

- (pending)
