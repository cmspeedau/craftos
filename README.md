# CraftOS

A custom Linux OS built on [Buildroot](https://buildroot.org), with a native Rust package manager (`mpkg`), an OS SDK (`mpkg-sdk`), a Wayland desktop, and an integrated app platform.

## Vision

CraftOS is a from-scratch, package-managed desktop Linux distribution:

- **Buildroot base** — deterministic, reproducible OS builds; every package output becomes an `.mpk` artifact
- **mpkg** — transactional, signed package manager written in Rust (the only way software — OS or apps — gets installed)
- **mpkg-sdk** — everything a third party needs to build, package, and deploy CraftOS apps
- **Wayland-native desktop** — branded compositor/shell on wlroots, launcher driven by `.desktop` files
- **App platform** — software store as the single front-end for apps *and* OS updates

## Architecture

```
┌─────────────────────────────────────────────┐
│              App Store (GUI)                │
├──────────────────────────┬──────────────────┤
│      Wayland Desktop     │  myos-updated    │
│  (wlroots shell, panel,  │  (update daemon) │
│   launcher, XWayland)    │                  │
├──────────────────────────┴──────────────────┤
│                    mpkg                     │  ← package transactions,
├─────────────────────────────────────────────┤     channels, rollback
│              Buildroot (glibc)              │
├─────────────────────────────────────────────┤
│               Linux kernel                  │
└─────────────────────────────────────────────┘
```

## Key components

| Component | Language | Repo/module | Status |
|---|---|---|---|
| OS build system | Make/Buildroot | `myos-build` | Phase 0 |
| Package manager (core) | Rust | `mpkg-core` | Phase 1 |
| Package manager (CLI) | Rust | `mpkg` | Phase 1 |
| SDK | Rust + scripts | `mpkg-sdk` | Phase 2 |
| Desktop (compositor/shell) | Rust (wlroots) | `craftos-shell` | Phase 3 |
| App store / updater | Rust | `craftos-store` | Phase 4 |
| Installer | Rust | `craftos-installer` | Phase 5 |

## Documentation

- [Roadmap](docs/ROADMAP.md) — phases, milestones, exit criteria
- [mpkg design](docs/MPKG.md) — packages, repos, transactions, channels, rollback
- [SDK design](docs/SDK.md) — toolchain, sysroot, app scaffolding, deploy
- [Desktop & GUI stack](docs/DESKTOP.md) — Wayland, shell, app compatibility
- [Updates & channels](docs/UPDATES.md) — package-based update flow, atomicity, channels
- [Installer design](docs/INSTALLER.md) — disk layout, install flow, first boot
- [Contributing](CONTRIBUTING.md)

## Status

🚧 Pre-Phase 0 — design and scaffolding. See the [roadmap](docs/ROADMAP.md).

## License

TBD (see [CONTRIBUTING.md](CONTRIBUTING.md)).
