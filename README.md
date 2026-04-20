# Stylos

**Stylos is the extracted Rust library foundation for the Stele ecosystem.**

This repository contains the reusable crates that were split out from the `https://github.com/tasanakorn/stele` repo so they can evolve independently and be consumed as normal Rust dependencies.

Current crates:

| Crate | Purpose |
| --- | --- |
| `stylos-common` | Shared constants, error types, and result aliases |
| `stylos-identity` | Realm/role/instance identity model and key composition |
| `stylos-config` | JSON5 config schema and loader |
| `stylos-transport` | Transport helpers such as endpoint building and port walking |
| `stylos-session` | `zenoh::Session` setup from Stylos config |

## Status

Stylos is currently a **library workspace**, not an installable CLI application.

That means:

- `cargo build` works at the workspace root
- crates can be used as git dependencies
- `cargo install --git https://github.com/tasanakorn/stylos.git` does **not** work yet because there is no binary target

## Origin

Stylos is based on code extracted from `https://github.com/tasanakorn/stele`.

The goal is to separate the low-level identity/config/session/transport pieces into their own focused Rust workspace while Stele continues to use them upstream.

## Workspace layout

```text
crates/
  stylos-common/
  stylos-identity/
  stylos-config/
  stylos-transport/
  stylos-session/
```

## Build

```bash
cargo build
```

## Use as a git dependency

Example using `stylos-config` directly from GitHub:

```bash
cargo add stylos-config --git https://github.com/tasanakorn/stylos.git
```

Then build normally:

```bash
cargo build
```

## Example

```rust
use stylos_config::StylosConfig;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let cfg = StylosConfig::load_default()?;
    println!("realm={} role={} instance={}", cfg.stylos.realm, cfg.stylos.role, cfg.stylos.instance);
    Ok(())
}
```

## Notes

- Config is JSON5-based
- `stylos-session` uses `zenoh = 1.9.0`
- default multicast discovery address is `224.0.0.224:31746`

## Future work

Possible next steps:

- add crate-level docs and examples
- add tests for identity validation and config loading
- add a small CLI binary if install-from-git is desired
