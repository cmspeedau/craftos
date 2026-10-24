## 8. A/B slots

A/B is an **optional hardening layer**, not the primary update mechanism.
CraftOS remains package-based. See [AB_UPDATES.md](AB_UPDATES.md) for the full
design; the interaction with mpkg:

- `mpkg` gains an `ab` mode (installer profile enables it): critical updates
  stage into the inactive slot, boot-flip with a confirmation counter, and
  auto-fallback to the old slot if the new one fails to boot.
- Non-critical packages always apply live, as normal.
- The mpkg database lives on a shared partition — one source of truth; slots
  are instantiations of it, and any divergence triggers a sync.
