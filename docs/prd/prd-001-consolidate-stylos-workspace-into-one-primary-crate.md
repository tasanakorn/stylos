# PRD-001: Consolidate Stylos Workspace into One Primary Crate with Optional Session Split

- **Status:** Proposed
- **Version:** v0.2.0
- **Scope:** workspace layout, crate packaging, docs
- **Author:** Tasanakorn (design) + Themion (PRD authoring)
- **Date:** 2026-04-21

## Goals

- Reduce the current Stylos workspace from five tiny crates to a simpler package layout that better matches the actual size and maturity of the codebase.
- Prefer one primary `stylos` crate that exposes identity, config, transport helpers, shared errors, and session-opening behavior behind one coherent API.
- Preserve a clear fallback recommendation for a two-crate layout only if keeping `zenoh`-backed session setup isolated remains materially valuable for downstream consumers.
- Improve usability for downstream Rust users so they can depend on Stylos without deciding among several very small crates.
- Keep the resulting architecture easy to document, version, and evolve from this repository as Stylos's primary home.

## Non-goals

- No redesign of Stylos key grammar, identity validation rules, discovery defaults, or session semantics in the same change.
- No requirement to add a CLI binary or installable application target as part of the crate consolidation.
- No requirement to expand cross-language support in this PRD.
- No requirement to add new runtime features beyond packaging and API-surface consolidation.
- No requirement to preserve every current crate as a long-term public package if that preservation conflicts with the simplification goal.

## Background & Motivation

### Current state

Current documentation describes Stylos as a Rust library workspace split into five crates:

- `stylos-common`
- `stylos-identity`
- `stylos-config`
- `stylos-transport`
- `stylos-session`

`docs/architecture.md` documents this split as a strict dependency DAG, and the root `README.md` presents those crates as the current reusable units.

However, the code currently inside those crates is still quite small:

- `stylos-common` contains shared constants, a single error enum, and `Result<T>`
- `stylos-identity` contains validated identity newtypes and root-key composition
- `stylos-config` contains the JSON5 config schema, defaults, and loader
- `stylos-transport` contains endpoint formatting and port walking
- `stylos-session` contains `zenoh::Config` construction and session opening

This split is architecturally tidy, but it is finer-grained than the present implementation size appears to justify.

### Why simplify now

The repository is documented as the primary home for Stylos. That makes first-use ergonomics and long-term maintainability important.

For a tiny foundational layer, a five-crate workspace creates overhead in several places:

- users must choose between several small packages before they know the project well
- maintainers must carry multiple manifests, path dependencies, and crate boundaries
- documentation must explain the split even when most users likely want the whole package
- future refactors across identity, config, and transport cross crate boundaries that are not providing strong independent value today

The current design has one clearly meaningful boundary: `zenoh` session setup versus everything else. The other crate boundaries are much weaker at the current size.

### Why prefer one crate first

The most proportionate design for the current repository is one primary crate.

A single `stylos` crate would:

- match the small size of the codebase
- make onboarding easier
- give downstream users one obvious dependency name
- reduce packaging and release complexity
- keep internal boundaries as modules instead of public crate seams

A two-crate split remains defensible only if there is still strong value in letting some consumers use identity/config/transport code without taking a `zenoh` dependency.

**Alternative considered:** keep the current five-crate layout and rely on documentation to hide the complexity. Rejected: the complexity is structural, not just explanatory, and the current crates are too small to justify that level of packaging.

## Design

### Make `stylos` the default and preferred package shape

The preferred end state is one primary crate named `stylos` that contains modules for:

- common errors and constants
- identity types and validation
- config schema and loading
- transport helpers
- session construction and logging helpers

The internal code may still be organized as modules such as `identity`, `config`, `transport`, and `session`, but those should become module boundaries inside the crate rather than separate packages.

Normative intent:

- new users should be able to add one dependency and get the normal Stylos API surface
- public examples and docs should prefer `stylos::...`
- internal layering should remain visible in code structure without forcing separate package boundaries
- constants and shared errors should live in the main crate root or a small internal module rather than a standalone `stylos-common` package

Recommended module shape:

```text
stylos/
  src/
    lib.rs
    error.rs
    identity.rs
    config.rs
    transport.rs
    session.rs
```

**Alternative considered:** keep multiple crates but add a façade crate named `stylos` that re-exports them. Rejected: that would improve onboarding somewhat, but it would leave most of the packaging and maintenance overhead in place.

### Keep an optional two-crate fallback only for the `zenoh` boundary

If maintainers still want one strong dependency boundary, the only currently justified split is:

- `stylos`
  - identity
  - config
  - transport helpers
  - shared constants and errors
- `stylos-session` or `stylos-zenoh`
  - `zenoh::Config` construction
  - session opening
  - session info helpers

This fallback is recommended only when avoiding a `zenoh` dependency for lightweight consumers remains an explicit project requirement.

Normative guidance:

- do not preserve separate crates for `common`, `identity`, `config`, or `transport` in the simplified design
- if a second crate remains, it should exist solely to isolate `zenoh`-specific session behavior
- docs should describe the two-crate option as an implementation compromise, not as the preferred conceptual model, unless the dependency-isolation requirement is reaffirmed clearly

**Alternative considered:** merge into two non-`zenoh`-based groupings such as `stylos-base` and `stylos-runtime`. Rejected: that would still force users to learn project-specific packaging distinctions without a correspondingly strong architectural need.

### Preserve internal layering as modules rather than public crate seams

The existing crate split captures a sensible conceptual order:

1. identity and config
2. transport helpers
3. session opening

That ordering should be preserved internally even if the public package count shrinks.

Normative behavior:

- module dependencies inside one crate should remain roughly acyclic and readable
- `session` code should continue to own `zenoh::Config` construction details
- identity validation should remain separate from config loading logic at the module level
- transport helpers should remain small standalone functions or a small module rather than being inlined indiscriminately into session code

This keeps the code understandable without over-packaging it.

**Alternative considered:** flatten all code into one large `lib.rs`. Rejected: fewer crates should not mean losing internal structure.

### Provide a compatibility-minded transition for current crate consumers

Even though the repository is small, crate consolidation changes public package names and import paths.

Recommended migration posture:

- if the implementation goes to one crate, downstream code should migrate toward `stylos::...`
- if practical, temporary compatibility crates may re-export from the new main crate for one release window
- compatibility crates should be explicitly transitional and documented as such rather than treated as permanent architecture
- if temporary re-export crates are not worth the maintenance cost, the change should be documented clearly as a deliberate breaking simplification

This PRD does not require permanent backward compatibility, but it does require the migration story to be intentional and documented.

**Alternative considered:** preserve all old crate names indefinitely as thin wrappers. Rejected: that would weaken the simplification goal and keep package sprawl alive under a different form.

### Update docs to describe the simplified conceptual model first

The documentation currently explains Stylos in terms of five workspace crates. After consolidation, docs should instead explain Stylos in terms of one main library, with an optional note about a secondary session crate only if that split remains.

Normative doc expectations:

- `README.md` should present Stylos as one primary Rust library
- `docs/architecture.md` should describe module layering first, not a five-crate DAG
- `docs/origin.md` and `docs/cross-lang.md` should continue to position this repository as the primary Stylos home without overstating crate granularity
- examples should prefer the primary crate import path

**Alternative considered:** leave architecture docs crate-centric because the internal modules still map to the old conceptual pieces. Rejected: docs should describe the product as users are meant to consume it, not preserve obsolete package boundaries for their own sake.

## Changes by Component

| File | Change |
| ---- | ------ |
| `Cargo.toml` | Reduce workspace membership to one primary crate, or at most two crates if the `zenoh` split is retained. |
| `crates/stylos-common/` | Merge into the primary `stylos` crate as internal error/constants modules; remove as a standalone package afterward unless kept briefly as a transitional re-export crate. |
| `crates/stylos-identity/` | Merge into the primary `stylos` crate as an internal `identity` module; remove as a standalone package afterward unless kept briefly for compatibility. |
| `crates/stylos-config/` | Merge into the primary `stylos` crate as an internal `config` module; remove as a standalone package afterward unless kept briefly for compatibility. |
| `crates/stylos-transport/` | Merge into the primary `stylos` crate as an internal `transport` module; remove as a standalone package afterward unless kept briefly for compatibility. |
| `crates/stylos-session/` | Merge into the primary crate or remain as the sole optional second crate if `zenoh` dependency isolation is still required. |
| `README.md` | Reframe Stylos as one primary crate, with a short note about an optional session split only if it remains. |
| `docs/architecture.md` | Replace the five-crate DAG with a simplified module-layer or one-to-two-crate architecture description. |
| `docs/README.md` | Add this PRD to the documentation index. |

## Edge Cases

- a downstream user currently depends only on `stylos-identity` for validation → the migration should still provide a straightforward replacement import path in the new primary crate.
- a downstream user wants config and identity handling but does not want to pull `zenoh` → this is the main reason to choose the two-crate fallback instead of the preferred one-crate result.
- docs and examples are updated incompletely → users may see both old per-crate imports and new unified imports, so the implementation should update top-level docs in the same change.
- temporary compatibility crates are kept too long → the repository may drift back into effectively supporting the old architecture indefinitely.
- crate merging changes public error or type paths but not behavior → release notes and migration notes should call out import-path breakage explicitly even when runtime behavior stays the same.

## Migration

This change is primarily a packaging and API-path simplification.

Preferred migration path:

- move to one public dependency: `stylos`
- update imports from crate-specific paths to module paths under `stylos`
- update documentation and examples in the same release
- if a second crate remains for `zenoh`, document clearly when users should choose `stylos` alone versus `stylos-session`

Compatibility guidance:

- if temporary re-export crates are provided, mark them as transitional and plan their removal
- if no compatibility crates are provided, treat the change as an intentional breaking simplification in the next appropriate release

## Testing

- merge the current five crates into a single primary crate in a branch or focused implementation spike → verify: the public API can still load config, validate identity, build listen endpoints, and open a session through one crate.
- evaluate a variant that keeps only a second session crate for `zenoh` isolation → verify: consumers that do not need sessions can use the main crate without taking the `zenoh` dependency.
- run `cargo check` at the workspace root after consolidation → verify: the reduced workspace builds cleanly.
- update `README.md` examples to the preferred import style and run the narrowest compilation check for them → verify: the documented import path matches the implemented crate layout.
- review `docs/architecture.md` and `README.md` after the change → verify: they describe one primary crate, or one plus a clearly justified second crate, rather than the previous five-crate DAG.
- inspect the resulting workspace manifests after consolidation → verify: only one crate remains by default, or two at most when the `zenoh` isolation requirement is intentionally preserved.
