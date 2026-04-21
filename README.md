# Stylos

**Stylos is the primary Rust library repository for the Stylos interconnect.**

Stylos provides a single primary crate for identity, config loading, transport setup, and `zenoh::Session` construction. This repo should be treated as the source-of-truth home for Stylos itself.

## What Stylos is

Stylos is a zenoh-backed interconnect foundation intended for cross-process and cross-host communication. It is designed to support:

- pub/sub on hierarchical key expressions
- query/queryable request-reply patterns
- shared identity conventions across consumers
- reusable session/bootstrap code for Rust applications

Today this repository contains one primary Rust library crate for those core pieces.

## Primary crate

| Crate | Purpose |
| --- | --- |
| `stylos` | Identity model, JSON5 config loading, transport helpers, and `zenoh::Session` setup |

The crate keeps internal module boundaries so applications can extend or consume only the pieces they need:

- `stylos::identity`
- `stylos::config`
- `stylos::transport`
- `stylos::session`

## Documentation

Primary Stylos docs live in this repository.

| Doc | What it covers |
| --- | --- |
| [`docs/README.md`](docs/README.md) | Stylos doc index |
| [`docs/architecture.md`](docs/architecture.md) | Process model, module layout, config construction, data flow |
| [`docs/addressing.md`](docs/addressing.md) | `stylos/<realm>/<role>/<instance>` key grammar |
| [`docs/discovery.md`](docs/discovery.md) | Multicast scouting, UDP/TCP listeners, failure modes |
| [`docs/poc.md`](docs/poc.md) | POC scenarios and smoke-test notes |
| [`docs/cross-lang.md`](docs/cross-lang.md) | Cross-language status notes |
| [`docs/origin.md`](docs/origin.md) | Origin and repository positioning |

## Status

Stylos is currently a **library crate**, not an installable CLI application.

That means:

- `cargo build` works at the workspace root
- the crate can be consumed as a dependency
- `cargo install --git ...` does not work yet because there is no binary target in this repo

## Workspace layout

```text
crates/
  stylos/
docs/
```

## Build

```bash
cargo build
```

## Use as a git dependency

```bash
cargo add stylos --git https://github.com/tasanakorn/stylos.git
```

Then build normally:

```bash
cargo build
```

## Example

```rust
use stylos::StylosConfig;

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
- `stylos` uses `zenoh = 1.9.0`
- default multicast discovery address is `224.0.0.224:31746`
- default data listeners are UDP + TCP on port `31747`

## Near-term gaps

Useful next additions for this repo:

- crate-level docs and examples
- tests for identity validation and config loading
- a small CLI binary if install-from-git is desired
