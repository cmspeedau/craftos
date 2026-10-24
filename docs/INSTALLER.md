# Installer Design

`craftos-installer` installs CraftOS to a target disk. Designed for Phase 5, specified now because it forces disk-layout and update-model decisions that are painful to retrofit.

## Model: package-seeding (hybrid)

Given CraftOS updates pull from mpkg, the installer uses a **package-based install with an image seed**:

1. Boot the installer environment (the CraftOS image itself, from USB/ISO — it dogfoods the Wayland desktop)
2. Partition and format the target disk
3. Unpack the base rootfs seed image
4. `mpkg upgrade --root /mnt/target` against the current `stable` channel → install lands on the *latest* packages, never a stale image

This avoids rebuilding Debian while avoiding stale-image installs.

## Components

| Component | Description |
|---|---|
| Live/installer environment | The OS image itself, bootable from USB/ISO |
| `craftos-installer` (GUI) | Wayland client app: Welcome → Disk → User → Install → Reboot |
| `installer-core` (Rust lib) | Pure backend logic, takes `target: &Path` — host-testable, same pattern as `mpkg-core` |
| First-boot wizard | Locale, timezone, user account, hostname; runs once, flagged via `/var/lib/myos/firstboot-done` |

## Disk layout

GPT, UEFI boot:

```
p1  ESP      512 MB   FAT32   bootloader + kernel
p2  root     rest     ext4    rootfs
```

### Optional: A/B slots

For atomic-image-update capability later, partition as:

```
p1  ESP (512M)   p2 root-A   p3 root-B (spare)
```

Costs ~2 GB of disk; makes future A/B atomic updates (write new image to inactive slot, flip bootloader entry, reboot) nearly free. **Not required** for the package-based update model — kept as an option for embedded/appliance targets.

## Install steps (installer-core)

1. Validate target (not the running root, not mounted elsewhere)
2. Partition: ESP + root (GPT), format
3. Unpack base rootfs seed to root partition
4. Write `/etc/fstab` by **UUID**
5. Install bootloader (**systemd-boot** or GRUB) + kernel to ESP
6. Configure: root password, locale defaults, network
7. `mpkg --root /mnt/target upgrade` to pull latest
8. Flag first-boot wizard
9. Reboot prompt

## Edge cases

- **Disk in use / is the live device:** hard-refuse with clear error
- **BIOS (non-UEFI) machines:** GRUB fallback, v0.2
- **Encrypted root:** LUKS option, post-v0.1
- **QEMU vs hardware:** same code path — `installer-core` targets a block device path either way

## Milestone

Write ISO/USB → boot target VM (then real PC) → GUI installer → reboot into the CraftOS desktop.
