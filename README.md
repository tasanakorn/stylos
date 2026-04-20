# Stylos

**Stylos is the primary Rust library repository for the Stylos interconnect.**

Stylos provides reusable crates for identity, config loading, transport setup, and `zenoh::Session` construction. This repo should be treated as the source-of-truth home for Stylos itself.

## What Stylos is

Stylos is a zenoh-backed interconnect foundation intended for cross-process and cross-host communication. It is designed to support:

- pub/sub on hierarchical key expressions
- query/queryable request-reply patterns
- shared identity conventions across consumers
- reusable session/bootstrap code for Rust applications

Today this repository contains the Rust library workspace for those core pieces.

## Workspace crates

| Crate | Purpose |
| --- | --- |
| `stylos-common` | Shared constants, error types, and result aliases |
| `stylos-identity` | Realm/role/instance identity model and key composition |
| `stylos-config` | JSON5 config schema and loader |
| `stylos-transport` | Transport helpers such as endpoint building and port walking |
| `stylos-session` | `zenoh::Session` setup from Stylos config |

## Documentation

Primary Stylos docs live in this repository.

| Doc | What it covers |
| --- | --- |
| [`docs/README.md`](docs/README.md) | Stylos doc index |
| [`docs/architecture.md`](docs/architecture.md) | Process model, crate split, config construction, data flow |
| [`docs/addressing.md`](docs/addressing.md) | `stylos/<realm>/<role>/<instance>` key grammar |
| [`docs/discovery.md`](docs/discovery.md) | Multicast scouting, UDP/TCP listeners, failure modes |
| [`docs/poc.md`](docs/poc.md) | POC scenarios and smoke-test notes |
| [`docs/cross-lang.md`](docs/cross-lang.md) | Cross-language status notes |
| [`docs/origin.md`](docs/origin.md) | Origin and repository positioning |

## Status

Stylos is currently a **library workspace**, not an installable CLI application.

That means:

- `cargo build` works at the workspace root
- crates can be consumed as dependencies
- `cargo install --git ...` does not work yet because there is no binary target in this repo

## Workspace layout

```text
crates/
  stylos-common/
  stylos-identity/
  stylos-config/
  stylos-transport/
  stylos-session/
docs/
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
    println!(
        "realm={} role={} instance={}",
        cfg.stylos.realm,
        cfg.stylos.role,
        cfg.stylos.instance
    );
    Ok(())
}
```

## Notes

- Config is JSON5-based
- `stylos-session` uses `zenoh = 1.9.0`
- default multicast discovery address is `224.0.0.224:31746`
- default data listeners are UDP + TCP on port `31747`

## Near-term gaps

Useful next additions for this repo:

- crate-level docs and examples
- tests for identity validation and config loading
- a small CLI binary if install-from-git is desired
