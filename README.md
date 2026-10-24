# CraftOS

A Linux distribution built on Buildroot with a Rust userland, its own package manager (`mpkg`), a native Wayland desktop, and an SDK for first- and third-party apps.

## Vision

CraftOS treats the OS as a set of packages managed by one tool: everything from the kernel to the app store is an `.mpk` package, updated through signed channels, transactionally and with rollback. First-party apps are Rust and target Wayland natively; third-party apps run through a packaging compatibility layer.

## Architecture

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for the full architecture: layers, data flow, runtime layout, and trade-offs.

```
Applications (first-party .mpk / third-party wrapped)
        │
Desktop Layer (compositor, shell, app store)
        │
System Services (mpkg, myos-updated, installer, firstboot)
        │
OS Core (kernel, glibc, systemd) — packaged via Buildroot
        │
Build & Distribution (myos-build, .mpk repos, channels, CI)
```

## Documentation

| Doc | Contents |
|---|---|
| [Architecture](docs/ARCHITECTURE.md) | Layers, components, data flow, design decisions |
| [Roadmap](docs/ROADMAP.md) | Phases 0–5, exit criteria, dependency map |
| [mpkg](docs/MPKG.md) | Package format, transactions, rollback, channels |
| [SDK](docs/SDK.md) | Toolchain export, `mpkg-sdk` workflow, scaffolding |
| [Desktop](docs/DESKTOP.md) | Wayland stack, app contract, XWayland policy |
| [Updates](docs/UPDATES.md) | Package-based updates, atomicity, channels, CI gating |
| [Installer](docs/INSTALLER.md) | Package-seeded install, disk layout, first boot |
| [Contributing](CONTRIBUTING.md) | Repo layout, conventions |

## Status

Early phase: architecture and documentation are defined; `mpkg` core work is starting (Phase 1). See the [roadmap](docs/ROADMAP.md) for what's next.

## License

TBD — see [CONTRIBUTING.md](CONTRIBUTING.md).
