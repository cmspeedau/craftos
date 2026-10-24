# Updates — Package-Based Update Flow

CraftOS updates **pull from mpkg**, like traditional Linux distributions — not monolithic image flashes. The OS is defined as *the set of installed packages*.

## Model

Buildroot builds every package individually. Instead of shipping a monolithic rootfs, **each Buildroot output becomes an `.mpk`** in the system repo:

```
mpkg repo (channels)
  ├── myos-base        ← kernel, glibc, init (critical packages)
  ├── myos-desktop     ← compositor, shell, panel
  └── ~200 system pkgs ← all Buildroot output as .mpk
```

```
mpkg update   → refresh index
mpkg upgrade  → diff installed vs repo → download → transactional apply
              → reboot if kernel/glibc changed
```

## Why this model

- **Delta updates for free** — only changed packages download
- **No dual maintenance** — the app store and OS updates use the same tool, repo, and UI
- **Granular rollback** — `mpkg rollback` restores prior package versions
- **One front-end** — the store's "Updates" tab handles OS + apps

## Hard problems, and fixes

### 1. Atomicity
Naive `tar x` over a live rootfs can brick mid-upgrade.

**Fix:** transactional apply — download all → verify signatures → unpack to staging → swap into place in one pass → record in `/var/lib/mpkg/history`. `myos-base`/kernel upgrades finish via a systemd unit on next boot (offline-finish pattern).

### 2. libc/toolchain upgrades
Replacing glibc under a running system is dangerous.

**Fix:** glibc/kernel updates always route through the **reboot-required path**. Everything else applies live. In `pkg.toml`: `critical = true` → staged + reboot.

## Channels

Same index format, different path/tag:

| Channel | Source | Audience |
|---|---|---|
| `stable` | Promoted from testing after soak | Users |
| `testing` | CI builds from `myos-build` main | Testers |
| `nightly` | Every green CI run | Developers |

## CI pipeline

Every commit to `myos-build`:

1. Buildroot build
2. Emit `.mpk` set → publish to `testing` channel
3. **Smoke test:** boot QEMU → `mpkg upgrade` → verify boot

That smoke test is what makes channel promotion (`testing` → `stable`) trustworthy.

## Update daemon

`myos-updated` (Phase 4):

- Periodically checks the subscribed channel
- Notifies the user (desktop notification + store badge)
- Applies non-critical updates (on request or on schedule)
- Flags `critical = true` updates as **reboot required**

## Milestone demo

Boot an older image in QEMU → `mpkg upgrade` pulls a new shell/compositor from the repo → new version runs live, no reboot → bump glibc → verify the "reboot required" flow.
