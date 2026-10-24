# CraftOS

A buildroot-based Linux OS with a native Rust package manager, software
store, and SDK. Everything user-facing — apps, tools, and OS components —
distributes through one system: **mpkg**.

## Components

| Repo | What | Status |
|---|---|---|
| `craftos` (this repo) | Vision, architecture, specs, coordination | active |
| `mpkg` | Rust package manager: install/update/rollback, channels, transactions | not started |
| `myos-build` | Buildroot fork: config, patches, package→`.mpk` packaging CI | not started |
| `myos-sdk` | Cross-compile toolchain + `mpkg-sdk` wrapper | not started |
| `myos-desktop` | wlroots compositor, shell, panel, store GUI | not started |
| `myos-updated` | Update-check/apply daemon with reboot-required flow | not started |
| `myos-installer` | Disk partitioning + package-seeded install (optional A/B profile) | not started |

## Quick architecture

```
+--------------------------------------------------+   
| Applications      (mpkg packages: GUI + CLI)     |   App layer
+--------------------------------------------------+   
| Desktop           (myos-desktop: wlroots, shell, |   GUI
|                    panel, app store)              |   
+--------------------------------------------------+
| System services   (mpkg-core, myos-updated,       |   Services
|                    session mgmt, network)         |   
+--------------------------------------------------+
| OS core           (kernel, glibc, systemd,       |   Base
|                    drivers) — from myos-build      |   
+--------------------------------------------------+
| Build & dist      (Buildroot CI → .mpk set →     |   Pipeline
|                    mpkg repo: stable/testing)      |   
+--------------------------------------------------+
```

Key decisions: one package manager for OS + apps; package-based updates
(linux-distro style, transactions + rollback); A/B slots as an optional
hardening profile for headless/edge targets; Rust for all first-party
userland; wlroots/Wayland desktop.

## Docs

- [Architecture](docs/ARCHITECTURE.md) — layers, principles, trade-offs
- [Roadmap](docs/ROADMAP.md) — phases 0–5 with exit criteria
- [mpkg spec](docs/MPKG.md) — package format, transactions, channels
- [SDK spec](docs/SDK.md) — toolchain, `mpkg-sdk` workflow
- [Desktop spec](docs/DESKTOP.md) — Wayland stack, app compatibility
- [Update system](docs/UPDATES.md) — package-based update flow, channels, CI gating
- [A/B updates](docs/AB_UPDATES.md) — optional slot-based fallback layer
- [Installer spec](docs/INSTALLER.md) — disk layout, install flow, A/B profile

## Status

Phase 0 — repo and documentation setup.
