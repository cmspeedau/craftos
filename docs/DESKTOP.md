# CraftOS Desktop — "Rusty"

The CraftOS desktop environment is **Rusty**: an all-Rust desktop suite whose visual and interaction language is inspired by *Satisfactory* — the factory-building game. Clean industrial panels, factory-automation metaphors, and a system that feels like a well-run production line.

> **Status:** Phase 3 of the roadmap (`docs/ROADMAP.md`). Nothing here is implemented yet — this is the design contract.

## 1. Why "all Rust"

| Concern | Choice | Rationale |
|---|---|---|
| Compositor | wlroots (C) — or smithay (Rust) as a stretch goal | wlroots is proven; smithay is pure Rust but less battle-tested. Start on wlroots, keep the shell UI 100% Rust so a smithay swap later is possible. |
| UI toolkit | iced (or egui for panels) | iced is retained-mode, GPU-accelerated, declarative — good fit for a styled DE shell. |
| Rendering | wgpu / Vulkan | one GPU story across shell, panels, and future native apps. |
| IPC / services | Tokio + a typed IPC protocol (serde) | Rust services talk Rust; system services use the same stack. |

Every component in the DE is a Rust crate in one workspace (`myos-desktop`), shipped as `critical = false` / `desktop` packages via `mpkg`.

## 2. The Satisfactory design language

Satisfactory's UI is an **industrial sci-fi factory interface**: dark brushed-metal panels, amber/orange accent lighting, stencil/technical typography, corner brackets, status lights, and a pervasive sense that the computer is a **factory you operate**.

Rusty translates this into concrete, implementable rules:

### Visual
- **Palette:** near-black gunmetal surfaces (`#14181c`, `#1d232a`), steel borders, one dominant **amber accent** (`#ff9d00`) for active/attention states, plus cyan (`#38c8d8`) and green (`#4ad14a`) for informational/OK states. No gradients for decoration — gradients only for metal/sheen materials.
- **Typography:** a technical/stencil-flavored display face for headings, a clean grotesque (e.g. Inter) for body text. Uppercase-with-letter-spacing for labels.
- **Panels & framing:** every surface is a "machinery panel" — beveled edges, corner brackets/bolts at frame corners, 1px hazard-stripe separators where panels meet.
- **Status everywhere:** small blinking LED dots and readouts (like machine indicators) on the dock, taskbar, and notifications — network, updates, battery, build status.
- **Motion:** mechanical, not bouncy — hard eases, slight over-travel like heavy machinery, UI sounds optional (soft servo clicks).

### Interaction metaphors
- The desktop **is a factory floor**; apps are **machines**; notifications are **production alerts**; the taskbar is the **control room**.
- **Overview mode** renders windows as a production line layout — a zoomed-out map you can rearrange, not just a grid of thumbnails.
- **Notifications** arrive as "alerts" with severity tiers matching factory alert colors (info / warning / critical), dockable to an alert tray.
- **The updater** is the flagship metaphor: an in-progress update shows as a conveyor of packages moving through stages (download → verify → stage → activate), directly visualizing `mpkg`'s transaction journal. Failed packages land in a "scrapper" bin with one-click retry.

Keep it a *working* DE first: all of this is theming + shell behavior layered on boring-reliable Wayland plumbing. No metaphor should cost functionality.

## 3. Components (crates)

All under the `myos-desktop` workspace:

| Crate | Role |
|---|---|
| `rusty-compositor` | wlroots-based compositor; or the smithay backend behind a trait so it's swappable |
| `rusty-shell` | panel/dock, launcher, overview mode (iced) |
| `rusty-notifications` | alert daemon + tray UI, severity tiers |
| `rusty-settings` | settings app; reads/writes `/etc` and user config via typed backends |
| `rusty-updates-ui` | the "conveyor" front-end to `mpkg` / `myos-updated` |
| `rusty-theme` | palette, typography, panel widget library shared by all crates |
| `rusty-session` | session entry (`rusty-session`) + greeter integration |
| `rusty-xway` | XWayland integration for non-native apps |
| `craftos-sdk-templates` | ships the Rusty widget/palette templates so third-party apps match the DE |

## 4. App model

Native apps are **Rust + iced/wgpu** using `rusty-theme`, packaged as `mpkg` packages with `.desktop` entries (`docs/DESKTOP.md` contract below still applies) and distributed through the Software Center (`docs/MPKG.md`).

Compatibility tiers:
1. **Native Rusty apps** — full theming, IPC, and lifecycle integration.
2. **Generic Wayland/EGL apps** — work fine, get window decorations from Rusty.
3. **X11 apps** — via XWayland, flagged in the Software Center ("legacy machine").

## 5. IPC & services

One typed IPC bus (Tokio + serde, Unix socket) connects the shell crates and exposes a small surface to apps:

```
rusty.protocol: notifications (publish/subscribe), settings (get/set),
                session (lock/logout/power), updates (status/events),
                launcher (search providers)
```

System services (`myos-updated`, `mpkg-core`) sit behind this bus so the DE never talks to package internals directly.

## 6. Milestones (maps to Phase 3)

| M | Deliverable |
|---|---|
| M1 | Compositor + terminal + GTK fallback apps usable on QEMU/VM |
| M2 | `rusty-shell` panel + launcher; `rusty-theme` v1 palette/widgets |
| M3 | Notifications + settings app; IPC bus stable |
| M4 | Overview mode + updates "conveyor" UI |
| M5 | XWayland, greeter/session polish, installable `desktop` package set |

## 7. Open questions

- iced vs. egui split for shell panels vs. quick tooling UIs (lean iced everywhere unless a need appears).
- Compositor: commit to wlroots for Phase 3, but define the backend trait now so smithay is a Phase 5+ option.
- Sound design: optional, off by default.
- Font licensing for the display face — pick OFL-licensed stencil fonts only.
