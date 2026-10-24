# Desktop & GUI Stack

CraftOS's graphical stack is Wayland-native, built with Rust.

## Stack

```
┌────────────────────────────┐
│  Apps (Rust wayland-client)│
├────────────────────────────┤
│  craftos-store, installer  │
├────────────────────────────┤
│  craftos-shell             │  ← panel, launcher, decorations
├────────────────────────────┤
│  wlroots (Rust bindings)   │
├────────────────────────────┤
│  libdrm / mesa / evdev     │
└────────────────────────────┘
```

## Phased approach

- **Phase 2 (proof):** ship `weston` or `labwc` from Buildroot. The goal is a working display, not identity.
- **Phase 3 (identity):** `craftos-shell` — a branded compositor/shell on wlroots:
  - Top panel (clock, status, notifications)
  - Launcher / app grid reading `/usr/share/applications`
  - Window decorations, alt-tab, workspaces
- **Phase 4 (integration):** the app store is itself the flagship Wayland client — if the store runs well, the compositor works.

## The `.desktop` contract

mpkg installs `type = "gui"` packages with desktop files into `/usr/share/applications`. The launcher and store both enumerate this directory. **A GUI app installed via mpkg automatically appears in the launcher.** No per-app registration, no daemon.

## XWayland

Optional, off by default, for legacy X11 prebuilt apps (`prebuilt` package type). CraftOS apps are Wayland-native; XWayland is a compatibility valve, not a dependency.

## App compatibility policy

| App kind | Mechanism |
|---|---|
| Native CraftOS apps | Built with `mpkg-sdk`, link system glibc |
| Services/daemons | musl static, no library surface |
| Third-party prebuilts | Bundled libs quarantined + `LD_LIBRARY_PATH` wrapper |
| Legacy X11 | XWayland (opt-in) |

## Rendering

- Apps: `wayland-client` + software rendering first (pixman), GPU acceleration (mesa) once hardware bring-up begins in Phase 5
- Compositor: wlroots handles DRM/KMS, input, output management
