# mpkg-sdk — OS SDK Design

The SDK lets any developer build, package, and deploy CraftOS applications without knowing anything about Buildroot.

## Contents

The SDK is exported from a Buildroot build and consists of:

- **Cross toolchain** (glibc-targeted)
- **Target sysroot** (`sysroot/`): headers + libs for linking against system libraries
- **`env.sh`** — sets `CC`, `CARGO_TARGET_*` linker config, `PKG_CONFIG_SYSROOT_DIR`, etc.
- **`mpkg-sdk`** — the SDK CLI (Rust)
- Rust targets: prebuilt crates for `wayland-client`, `wlroots` bindings, etc.

## CLI

```
mpkg-sdk new <name>       # scaffold an app project (Cargo workspace + pkg.toml)
mpkg-sdk build            # cargo build against the sysroot
mpkg-sdk package          # produce <name>.mpk
mpkg-sdk deploy           # scp .mpk to device + remote `mpkg install`
mpkg-sdk run              # build + deploy + launch (dev loop)
```

## Scaffolding (`new`)

```
my-app/
├── Cargo.toml
├── pkg.toml
├── packaging/
│   └── my-app.desktop
└── src/main.rs        # minimal Wayland client starter
```

`pkg.toml` is the same manifest mpkg consumes — the SDK and package manager share one contract.

## Build flow

1. Source `env.sh`
2. `cargo build --target x86_64-craftos-linux-gnu --release`
3. Linker resolves against the sysroot, so binaries match the OS's glibc
4. `package` assembles `data/usr/bin`, `data/usr/share/applications`, `pkg.toml` into an `.mpk`

## Dev loop

Typical workflow against a QEMU device (or real hardware over SSH):

```
mpkg-sdk run   # build → deploy → launch on target in one command
```

Device target is configured in `.mpkg-sdk.toml` (host, port, ssh user).

## Distribution of the SDK

- SDK tarball published per release, alongside the OS image
- A Buildroot `BR2_SDK`-style export in `myos-build` regenerates it deterministically
