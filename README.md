# CraftOS

A hobby operating system built on **Buildroot Linux**, with an **all-Rust userland, package manager, SDK, and desktop environment**.

> **Name note:** the desktop environment is called **Rusty** — an all-Rust DE inspired by *Satisfactory*'s industrial factory UI. See `docs/DESKTOP.md`.

## Vision

CraftOS is a clean-room OS experience on top of Buildroot: everything above the kernel is Rust, updates are transactional, and the desktop looks and feels like the control room of a Satisfactory factory — dark gunmetal panels, amber accents, and the machine-as-metaphor everywhere.

- **One package manager:** `mpkg` (Rust) — local `.mpk` packages, `pkg.toml` manifests, signed repos, transactions with rollback.
- **One SDK:** Rust toolchain + sysroot export, `mpkg-sdk new/build/package/deploy/run`.
- **One desktop:** Rusty — Rust shell on Wayland (iced/wgpu), apps use shared Rusty widget templates.
- **Package-based updates** with an **optional A/B slot mode** for appliance/edge installs.

## Architecture

See **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)** for the full layered architecture, data flows, and trade-offs.

## Documentation

| Doc | Contents |
|---|---|
| [ROADMAP.md](docs/ROADMAP.md) | Phases 0–5, milestones, exit criteria |
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | Layered architecture, data flows, design decisions |
| [MPKG.md](docs/MPKG.md) | Package manager: format, repos, transactions, SDK integration |
| [SDK.md](docs/SDK.md) | OS SDK: toolchain, scaffolding, app lifecycle |
| [DESKTOP.md](docs/DESKTOP.md) | **Rusty** DE: Satisfactory-inspired design language, crates, milestones |
| [UPDATES.md](docs/UPDATES.md) | Update flow, `myos-updated`, CI gating |
| [AB_UPDATES.md](docs/AB_UPDATES.md) | Optional A/B slot mode on top of the package model |
| [INSTALLER.md](docs/INSTALLER.md) | ISO, disk layout, package seeding, first boot |
| [CONTRIBUTING.md](CONTRIBUTING.md) | Repo layout, conventions, how to help |

## Status

Phase 0 — build foundation. `myos-build` scaffolding and `mpkg-core` crate are the first deliverables.

## License

TBD — see CONTRIBUTING.md.
