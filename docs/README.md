# Stylos Documentation

Stylos documentation lives in this repository.

## Contents

| Doc | What |
| --- | --- |
| [architecture.md](architecture.md) | Process model, crate split, config construction, data flow |
| [addressing.md](addressing.md) | `stylos/<realm>/<role>/<instance>` key grammar and usage |
| [discovery.md](discovery.md) | Multicast scouting, data listeners, failure modes |
| [poc.md](poc.md) | POC scenarios and smoke-test notes |
| [cross-lang.md](cross-lang.md) | Rust/Go/Python/TS binding status and Go prereqs |
| [origin.md](origin.md) | Origin in Stele and the move to this primary repo |

## PRDs

| PRD | Status | What |
| --- | --- | --- |
| [prd-001-consolidate-stylos-workspace-into-one-primary-crate.md](prd/prd-001-consolidate-stylos-workspace-into-one-primary-crate.md) | Proposed | Prefer one primary `stylos` crate and allow at most one optional second session crate only if `zenoh` dependency isolation is still required. |
