# CraftOS Architecture

This document describes the overall architecture of CraftOS: its layers, major components, data flow, and the design decisions that bind them together. It complements the per-component docs ([mpkg](MPKG.md), [SDK](SDK.md), [Desktop](DESKTOP.md), [Updates](UPDATES.md), [Installer](INSTALLER.md)) which describe each piece in depth.

## 1. Architectural Overview

CraftOS is a Linux distribution built on Buildroot, with a Rust userland, its own package manager (`mpkg`), a native Wayland desktop, and an SDK for building third-party apps.

```
┌─────────────────────────────────────────────────────────────┐
│                        Applications                        │
│   First-party (Rust, .mpk)   │   Third-party (flatpak/x)   │
├─────────────────────────────────────────────────────────────┤
│                     Desktop Layer (Rust)                    │
│   Compositor (wlroots)  │  Shell / Panel  │  App Store GUI  │
├─────────────────────────────────────────────────────────────┤
│                    System Services (Rust)                   │
│   myos-updated │ mpkg (lib) │ myos-installer │ myos-updater │
├─────────────────────────────────────────────────────────────┤
│                      OS Core (Buildroot)                    │
│   Linux kernel │ glibc/musl │ systemd │ udev │ toolchain    │
├─────────────────────────────────────────────────────────────┤
│                      Build & Distribution                   │
│   myos-build (Buildroot) │ .mpk repo │ update channels │ CI │
└─────────────────────────────────────────────────────────────┘
```

The stack is intentionally layered: each layer depends only on the one below it. The OS core is upstream Buildroot plus a thin overlay; everything above it is first-party Rust.

## 2. Core Design Principles

1. **One package manager, one source of truth.** Everything installed — OS core packages, desktop, apps — is represented as an `.mpk` package tracked in mpkg's database. No parallel system (apt overlays, image-only components) is allowed.
2. **Rust-first userland.** All first-party userspace (mpkg, SDK, desktop, installer, update daemon) is Rust, sharing `*-core` library crates that are pure and host-testable.
3. **Package-based, not image-locked.** The system updates by pulling packages from channels, not by reflashing images. A base image exists only for install/seed.
4. **Transactions, not in-place edits.** Destructive operations (install, upgrade, rollback) are staged, verified, then applied as a unit. Critical packages (kernel, libc) apply at next boot.
5. **Same binary everywhere.** The installer, updater, and store all link the same `mpkg-core`; behavior is identical whether run interactively, scripted, or from CI.
6. **Native first, compatible second.** First-party apps target Wayland directly; X11/flatpak support is a compatibility layer, never a dependency of the core desktop.

## 3. Layer Detail

### 3.1 Build & Distribution Layer

**myos-build** — a Buildroot-based build system:

- `buildroot/` — pinned upstream Buildroot + `board/craftos/` overlay (kernel config, rootfs skeleton, defconfig)
- `package/` — Buildroot package recipes for first-party Rust components (mpkg, compositor, etc.)
- `mpk/` — post-build script that converts Buildroot per-package outputs into `.mpk` files
- CI publishes the resulting package set to the **nightly** channel on every green run; promotion to `testing` → `stable` is manual after a QEMU boot smoke test.

**Repository format** — a channel is a static directory: `index.json` (signed) + `.mpk` blobs. Any static file host can serve it.

### 3.2 OS Core Layer

- **Kernel**: Linux, config maintained in `myos-build/board/craftos/`.
- **libc**: glibc initially (musl as a build variant later).
- **Init**: systemd — required for the offline-finish update path, unit management, and login session handling.
- **Foundation**: udev, dbus, wayland libraries, Mesa — all packaged as `.mpk` by the build.

These packages are marked `critical = true` in their manifests: they stage and apply at boot, never over a running system.

### 3.3 System Services Layer

| Component | Role | Key crate split |
|---|---|---|
| `mpkg` / `mpkgd` | Install, upgrade, remove, rollback; CLI + library API | `mpkg-core` (pure logic), `mpkg-cli` |
| `myos-updated` | Update daemon: timer-driven channel check, notify, apply, reboot flagging | links `mpkg-core` |
| `myos-installer` | Disk partitioning, package-seeded install, user setup | `installer-core` (targets `&Path`) |
| `myos-firstboot` | One-shot onboarding wizard (locale, user, hostname) | Wayland app |

`mpkg-core` is the most important crate in the system. It owns:

- **Package database**: `/var/lib/mpkg/db` — installed manifest per package, files owned, versions.
- **Transaction journal**: `/var/lib/mpkg/txns` — ordered record of every applied transaction (what, when, from which channel, prior versions), enabling `mpkg history` and `mpkg rollback`.
- **Staging engine**: unpack to a staging directory, verify signatures and file conflicts, then atomically move into `/usr`.

### 3.4 Desktop Layer

- **Compositor**: wlroots-based, Rust (`wlr-rs` bindings). Owns output management, workspaces, window decoration policy.
- **Shell/Panel**: Wayland layer-shell client — launcher, taskbar, notifications, settings.
- **App Store / Update Center**: one GUI that fronts `mpkg-core` for browsing, installing apps, and applying OS updates. Shows "reboot required" when a critical transaction is staged.
- **App contract**: `.desktop` files + icons shipped in packages; GUI launcher enumerates via the standard menu spec. XWayland is available for legacy apps but no first-party component depends on it.

### 3.5 Application Layer

- **First-party apps** are Rust, built with the SDK, shipped as `.mpk` in the same channels as the OS.
- **Third-party apps**: prebuilt flatpaks/AppImages wrapped into `.mpk` bundles so the store, update system, and rollback treat them uniformly.
- **SDK**: exports the exact toolchain + sysroot from a `myos-build` output, so apps link against the same libraries the target image runs. `mpkg-sdk new/build/package/deploy/run` covers scaffold → QEMU round-trip.

## 4. Data Flow

### Install (Phase 5)

```
Boot installer media → myos-installer
  → partition disk (ESP + root)
  → seed rootfs from base image
  → mpkg upgrade (pull latest from stable channel)   ← system is current at install time
  → write fstab (UUID), bootloader
  → reboot → myos-firstboot wizard
```

### Update

```
myos-updated (timer) → check stable channel index
  → if updates: notify user
  → mpkg upgrade (via mpkg-core transaction):
       download → verify signatures → stage → apply
  → all files in /usr? → live apply, done
  → critical pkgs (kernel/glibc) staged? → write offline-finish systemd unit
  → reboot: unit applies staged critical pkgs before services start
```

### App development

```
mpkg-sdk new  → scaffold (Cargo + pkg.toml)
mpkg-sdk build → compile against exported sysroot
mpkg-sdk package → .mpk + .desktop entry
mpkg-sdk deploy/run → install to QEMU target, launch
```

## 5. Key Files & Paths (runtime layout)

| Path | Owner | Purpose |
|---|---|---|
| `/usr` | mpkg | All managed program files |
| `/etc` | admin | Machine-local config (mpkg never overwrites modified configs) |
| `/var/lib/mpkg/db` | mpkg-core | Installed package database |
| `/var/lib/mpkg/txns` | mpkg-core | Transaction journal (rollback source) |
| `/var/lib/mpkg/cache` | mpkg | Downloaded `.mpk` blobs |
| `/var/lib/myos/firstboot-done` | installer | First-boot wizard flag |
| `/usr/share/wayland-sessions/` | desktop | Session entries for the compositor |

## 6. Trade-offs & Rejected Alternatives

| Decision | Chose | Rejected | Why |
|---|---|---|---|
| Update model | Package-based (Debian/Arch-style) | Atomic image / A/B slots | One system for apps and OS; deltas; matches "update from package manager like Linux". A/B slots remain an installer option, not a requirement. |
| Init | systemd | runit/s6 | Offline-finish updates, session/seat management, mature Wayland integration. |
| Package format | Own `.mpk` (tar+zstd + signed index) | Adopt dpkg/rpm | Full control over transaction semantics; format is deliberately simple. |
| First-party language | Rust | C/C++/Go | Memory safety in system-critical tooling; shared `*-core` crate pattern. |
| Compositor | wlroots-based | HM/Sway fork | Native, fully first-party desktop identity; wlroots handles the hardest wlroots-of-the-work. |

## 7. Component Status

| Component | Status |
|---|---|
| Architecture & docs | **This document — defined** |
| mpkg (core transactions, channels) | Phase 1 — in progress |
| SDK | Phase 2 — planned |
| Desktop stack | Phase 3 — planned |
| Update daemon & store integration | Phase 4 — planned |
| Installer | Phase 5 — planned |
