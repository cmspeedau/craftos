# mpkg — Package Manager Design

`mpkg` is CraftOS's native package manager, written in Rust. It is the *only* mechanism by which software — system or third-party — is installed.

## Package format

### `pkg.toml`

```toml
[package]
name = "craftos-hello"
version = "0.1.0"
architecture = "x86_64"
description = "Hello world app for CraftOS"
type = "app"            # app | gui | service | system | prebuilt
critical = false        # true → staged + reboot required (kernel, glibc)

[dependencies]
libwayland = ">=1.20"

[build]                 # for source packages
system = "cargo"        # cargo | make | cmake | custom

[install]
desktop = "craftos-hello.desktop"  # gui packages
```

### `.mpk` archive

- `tar` + `zstd`
- Layout: `pkg.toml`, `data/` (rootfs files), `scripts/` (pre/post install hooks)
- Signed (ed25519) — signature carried in the repo index

## Core principles

1. **`root: &Path` everywhere.** All core operations take a target root path. Unit tests run against a fake rootfs on the host — no VM required.
2. **Transactional.** Downloads complete → verify → unpack to staging → atomic swap → record transaction. Nothing is half-installed.
3. **Deterministic index.** Repo is an HTTP directory of `.mpk` files plus a signed JSON index.

## CLI

```
mpkg install <pkg>...   mpkg remove <pkg>...   mpkg upgrade
mpkg list [--installed] mpkg verify <pkg>      mpkg search <term>
mpkg update             mpkg history            mpkg rollback <tx>
mpkg repo add|remove|list
```

## Transactions

A transaction is: resolve deps → download all → verify signatures → stage to `/var/lib/mpkg/staging/<tx>/` → swap into root in one pass → append to `/var/lib/mpkg/history/<tx>.json`.

**Critical packages** (`critical = true`: kernel, glibc, init) are staged across the transaction and finished by a systemd unit on the next boot (offline-finish pattern). Everything else applies live.

## Rollback

- History is append-only JSON per transaction (installed/removed/upgraded, versions, timestamps)
- `mpkg rollback <tx>` re-installs prior versions from the repo (or a local cache)

## Prebuilt packages

Third-party upstream tarballs are wrapped as `.mpk` with `type = "prebuilt"`:
- Bundled libraries are quarantined into `lib/private/`
- A wrapper script sets `LD_LIBRARY_PATH` before exec

## Update channels

| Channel | Source | Audience |
|---|---|---|
| `stable` | Promoted from testing after soak | Users |
| `testing` | CI builds from `myos-build` main | Testers |
| `nightly` | Every green CI run | Developers |

Channels are index paths/tags; `mpkg` subscribes to one at a time.

## Testing strategy

- `mpkg-core`: 100% unit-tested against fake rootfs on host
- Integration: boot QEMU → `mpkg install/upgrade/remove` against a local test repo
- CI smoke test: boot older image → `mpkg upgrade` → verify boot (this is what gates channel promotion)
