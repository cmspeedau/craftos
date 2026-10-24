# CraftOS Roadmap

Each phase's exit demo must be **usable**, not theoretical: booting image → installed package → drawn window → launched app → updated app → installed OS.

## Phase 0 — Foundation (Weeks 1–2)

**Goal:** a booting, hackable image.

- Buildroot image for QEMU x86_64 (`virtio` storage/gpu/net)
- Deterministic, reproducible builds
- Repo: `myos-build` (buildroot config fragments + post-build scripts), branch per image channel
- **glibc** chosen as libc (needed for GUI + prebuilt app compat)

**Exit:** image boots to shell in QEMU in one command.

## Phase 1 — mpkg core (Weeks 3–6)

**Goal:** native package management, testable on host.

- `mpkg-core` workspace: `pkg.toml`, `.mpk` archive (tar+zstd), fake-rootfs transactional install/remove
- `mpkg` CLI: `install`, `remove`, `list`, `verify` — always takes `root: &Path` so tests never need QEMU
- `mpkg upgrade` + transaction history + update channels
- Signed repo: HTTP index + ed25519 signatures

**Exit:** `mpkg install` works on the live image from your repo; 100% of core logic unit-tested on host.

## Phase 2 — SDK + first GUI milestone (Weeks 7–12)

**Goal:** build apps *for* the OS, and prove graphics.

- Wayland in Buildroot (`weston` or `labwc` first — ship now, customize later)
- SDK export from Buildroot: toolchain + sysroot + `env.sh` + `mpkg-sdk`
- `mpkg-sdk` commands: `new` (scaffold), `build` (cargo + sysroot), `deploy` (scp + remote `mpkg`)
- **Milestone demo:** boots → Wayland compositor → Rust `wayland-client` hello window on QEMU display

**Exit:** third parties could theoretically build and install a GUI app.

## Phase 3 — Desktop layer (Months 4–6)

**Goal:** an actual OS experience, not a shell.

- Custom branded shell on **wlroots** (Rust bindings): panel, launcher, window decorations
- Launcher reads `/usr/share/applications` — the `.desktop` contract with mpkg
- `pkg.toml` gains `type = "gui"`, desktop-file handling
- XWayland for legacy X11 prebuilts (optional, off by default)

**Exit:** QEMU boots into the CraftOS desktop, launches apps by icon.

## Phase 4 — App platform (Months 6–9)

**Goal:** the software front-end.

- App store UI = first flagship Wayland client (also a compositor stress test)
- `prebuilt` package type in mpkg: upstream tarballs wrapped as `.mpk`, bundled-libs/`LD_LIBRARY_PATH` wrapper generation
- App compat policy: **musl-static for services, system glibc for GUI, wrapper for third-party prebuilts**
- `myos-updated` daemon: check → notify → apply → flag reboot
- Delta updates, store "Updates" tab

**Exit:** install, launch, and update a GUI app entirely from the store.

## Phase 5 — Hardware + release (Months 9–12)

**Goal:** beyond the VM.

- Real x86 hardware bring-up (GPU: libdrm/mesa path); ARM64 as a second target
- Update channels: stable/nightly, transactional OS updates via mpkg
- `craftos-installer`: package-seeding install (base image + `mpkg upgrade` to latest), GUI installer, first-boot wizard
- v0.1 self-installing release

**Exit:** v0.1 installer ISO → boots to CraftOS desktop on real hardware.

## Dependency map

```
Phase 0 ──▶ Phase 1 ──▶ Phase 2 ──▶ Phase 3 ──▶ Phase 4 ──▶ Phase 5
 (image)   (mpkg)    (SDK+GUI)    (desktop)   (app store)  (hardware)
                ▲           │            │
                └── mpkg repo feeds SDK deploy & desktop launcher ──┘
```
