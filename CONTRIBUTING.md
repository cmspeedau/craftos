# Contributing to CraftOS

## Repositories

| Repo | Purpose |
|---|---|
| `craftos` | This repo — top-level docs, design, coordination |
| `myos-build` | Buildroot configuration, image build, CI |
| `mpkg` | Package manager (`mpkg-core` + `mpkg` CLI), Rust |
| `mpkg-sdk` | SDK tooling, Rust |
| `craftos-shell` | Wayland compositor/shell, Rust |
| `craftos-store` | App store + update daemon, Rust |
| `craftos-installer` | OS installer, Rust |

## Conventions

- **Rust workspaces** for multi-crate repos; core logic separated from I/O so it runs host-side in unit tests (the `root: &Path` pattern)
- **Commits:** conventional commits (`feat:`, `fix:`, `docs:`, ...)
- **Design docs live here** in `docs/`; implementation repos link back to them
- Every feature proposal starts as an issue in this repo referencing the relevant doc

## Getting started (Phase 0)

1. Clone `myos-build`
2. `make craftos_defconfig && make` → boot QEMU image
3. See [docs/ROADMAP.md](docs/ROADMAP.md) for current phase

## License

TBD — GPL-compatible preferred (Buildroot ecosystem). To be decided before first external contribution.
