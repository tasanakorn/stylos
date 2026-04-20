# Stylos Architecture

Stylos is a zenoh-backed interconnect layer. In this repository it is primarily a reusable Rust library workspace for identity, config, transport, and session setup.

## Process model

A Stylos participant is a single process that opens one `zenoh::Session`. Library consumers can embed Stylos directly instead of reimplementing session/bootstrap logic.

## Crate split

Five libraries with a strict dependency DAG:

```text
stylos-session -> stylos-transport -> stylos-common
               -> stylos-identity  -> stylos-common
               -> stylos-config    -> stylos-common
```

- **stylos-common** — constants, `StylosError`, `Result<T>`
- **stylos-identity** — validated `Realm` / `Role` / `Instance` and root-key composition
- **stylos-config** — JSON5 config schema and loader
- **stylos-transport** — endpoint building and port walking
- **stylos-session** — `open_session(&cfg, &overrides)` and session info helpers

## Config construction

`stylos-session` builds a `zenoh::Config` programmatically from `StylosConfig`:

- mode
- listen endpoints
- connect endpoints
- multicast scouting
- gossip settings

This keeps most zenoh-specific config details isolated inside one crate.

## Data flow

1. Load `StylosConfig`
2. Validate identity via `stylos-identity`
3. Choose listen port if endpoints are not explicitly pinned
4. Build zenoh config
5. Open `zenoh::Session`
