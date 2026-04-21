# Stylos Architecture

Stylos is a zenoh-backed interconnect layer. In this repository it is primarily a reusable Rust library exposed as one primary crate with internal modules for identity, config, transport, and session setup.

## Process model

A Stylos participant is a single process that opens one `zenoh::Session`. Library consumers can embed Stylos directly instead of reimplementing session/bootstrap logic.

## Module layout

The public crate is `stylos`, with internal module layering that preserves the previous conceptual split without forcing several tiny packages:

```text
stylos
  ├── error
  ├── identity
  ├── config
  ├── transport
  └── session
```

- **error** — constants, `StylosError`, `Result<T>`
- **identity** — validated `Realm` / `Role` / `Instance` and root-key composition
- **config** — JSON5 config schema and loader
- **transport** — endpoint building and port walking
- **session** — `open_session(&cfg, &overrides)` and session info helpers

Applications can still extend Stylos by using the module-level APIs directly instead of only one top-level convenience entrypoint.

## Config construction

`stylos::session` builds a `zenoh::Config` programmatically from `StylosConfig`:

- mode
- listen endpoints
- connect endpoints
- multicast scouting
- gossip settings

This keeps most zenoh-specific config details isolated inside one module.

## Data flow

1. Load `StylosConfig`
2. Validate identity via `stylos::identity`
3. Choose listen port if endpoints are not explicitly pinned
4. Build zenoh config
5. Open `zenoh::Session`
