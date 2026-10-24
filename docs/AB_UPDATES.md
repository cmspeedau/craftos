# A/B Updates in CraftOS

Status: design (Phase 5+)
Related: [ARCHITECTURE.md](ARCHITECTURE.md) · [UPDATES.md](UPDATES.md) · [MPKG.md](MPKG.md) · [INSTALLER.md](INSTALLER.md)

## Positioning

CraftOS is **package-based first** (`mpkg` pulls OS updates like a traditional
Linux distro). A/B slots are an **optional hardening layer**, not the update
mechanism itself. The two compose rather than compete:

| Layer | Provides | Used by |
|---|---|---|
| `mpkg` transactions | atomic package apply, journal, rollback | every install/update |
| A/B slots | boot-time fallback if a new set **fails to boot** | server/edge/appliance tiers; optional on desktop |

`mpkg` rollback undoes a *transaction* from a running (booted) system. A/B
recovers from the case `mpkg` rollback cannot: **the new state never boots.**

## Why not pure A/B (and why not pure packages)

Pure A/B (OSTree/Android-style): every update is a full image → conflicts with
the package manager, app store, and SDK workflow; large downloads per update.

Pure packages: a bad `myos-base`/kernel update can brick the device with no
console access.

Hybrid: packages are the unit of change; A/B is the safety net for the small
critical subset (`myos-base`, kernel, glibc) that already requires reboot.

## Layout

Two complete rootfs slots plus a shared data partition. Slots exist **only on
devices that opt in**; the standard desktop layout (single root) is unchanged.

```
p1  ESP / boot        kernel A + B (versioned names), boot config, slot state
p2  rootfs A          complete OS set (packages)
p3  rootfs B          complete OS set (packages)
p4  /var + /home      packages cache, mpkg DB, user data (shared, not duplicated)
```

Note: `mpkg`'s database and cache live on the **shared** partition, so there is
exactly one source of truth for installed-package state. Slots are
instantiations of that state.

## mpkg ab mode

`mpkg` gains an `ab` mode (enabled by installer profile, not hard-coded):

1. **Stage, don't commit.** A `critical = true` update downloads and stages to
   a staging area (as in the normal offline-finish flow), but at commit time
   materializes the result into the **inactive slot** instead of the live one.
2. **Sync non-critical packages.** After staging, `mpkg ab sync` replays the
   active slot's current non-critical package state into the new slot, so both
   slots converge to the same desired state.
3. **Flip the boot target.** Update boot config to boot the new slot, set
   `try_new=1` in the slot-state file (on the ESP/shared partition), reboot.
4. **Confirm or fall back.**
   - Success: after the user's session (or network) is up, `myos-updated`
     marks the slot `confirmed`. New slot becomes active; old slot retained as
     fallback.
   - Failure: boot counter (maintained by an early-boot unit) exceeds the
     threshold before confirmation → bootloader/early unit flips back to the
     old slot. Nothing user-visible is lost; `/var` was never touched.

State file (on shared/boot partition, not inside either rootfs):

```toml
# /boot/slot-state.toml
active = "A"
try_new = "B"
attempts = 0          # incremented on each failed boot attempt, reset on success
max_attempts = 3
confirmed = false
```

The slot logic itself is intentionally tiny — an early-boot `systemd` unit plus
bootloader config — so it can be relied upon when the main OS is broken.

## Consistency: packages vs slots

The failure mode of hybrid systems is slot state drifting from the mpkg DB.
Rules to prevent it:

- The **inactive slot is written only by** `mpkg ab stage/sync`. No other writer.
- Every mpkg transaction records the active slot ID in the journal; a
  transaction journal from the other slot triggers a full `mpkg ab sync` on boot.
- Non-critical package updates **apply to the live slot directly** (live, as
  normal) and set `slots_dirty = true`; the dirty flag is cleared by the next
  sync during the next critical update.
- `mpkg rollback` in ab mode re-syncs and re-stages rather than mutating the
  active slot in place.

## Failure matrix

| Failure | Package-only system | A/B system |
|---|---|---|
| Bad app / desktop package | `mpkg rollback <tx>` live | same (live rollback) |
| Bad kernel that panics early | recovery console / reflash | **auto-fallback to old slot** |
| Bad glibc | recovery console / reflash | **auto-fallback** |
| Update interrupted (power loss) | transaction journal replays | slot never activated |
| Fallback slot also broken | — | boot A→B→A→B counter, then rescue prompt |

## Rollout

- **Phase 5 (installer):** add `--profile ab` to `myos-installer` creating the
  4-partition layout; default desktop profile stays single-root.
- **Phase 6:** `mpkg ab` subcommands (`stage`, `sync`, `confirm`, `status`),
  boot-counter unit, `myos-updated` confirmation hook.
- Target devices: any appliance/server image. Desktop keeps single-root unless
  a user opts in.

## Milestone

QEMU demo: install with `--profile ab`, apply a critical update, watch it boot
into slot B, then corrupt slot B's kernel deliberately → device boots back into
slot A automatically, `mpkg ab status` shows the fallback reason.
