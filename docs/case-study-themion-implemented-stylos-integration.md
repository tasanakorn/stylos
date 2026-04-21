# Case Study: What Themion Actually Implemented on Stylos

This document describes the concrete Stylos integration that Themion implemented.

It is not a generic Stylos bring-up note. It is a case study of how **Themion** uses Stylos in practice, based on the implemented Themion code and docs.

## Sources used

Implementation and project docs reviewed for this case study:

- Themion repository: `crates/themion-cli/src/stylos.rs`
- Themion repository: `docs/architecture.md`
- Themion repository: `docs/prd/prd-019-basic-stylos-support-in-themion-cli.md`
- Themion repository: `docs/prd/prd-021-single-process-multi-agent-runtime-and-stylos-reporting.md`
- Themion repository: `docs/prd/prd-022-stylos-queryables-for-agent-presence-availability-and-task-requests.md`

## Why this case study matters

The Stylos crate in this repository provides transport/session foundations. Themion is a real external consumer of that foundation.

Themion shows a concrete implemented pattern for:

- choosing a Stylos identity for an application process
- opening one Stylos session per process
- publishing multi-agent process status over one session
- exposing discovery and direct-query surfaces over Stylos
- routing remote requests into an existing local runtime instead of inventing a second execution model
- using Stylos for presence, discovery, and lightweight delegation rather than for core model execution

That makes Themion a useful case study of Stylos as an embedded app-level mesh/reporting layer.

## High-level architecture of the Themion integration

Themion keeps Stylos in the CLI/runtime layer, not in its core agent loop.

Per Themion’s `docs/architecture.md` and `crates/themion-cli/src/stylos.rs`:

- `themion-core` owns agent logic, workflows, history, model calls, and tool definitions
- `themion-cli` owns process startup, TUI/runtime wiring, and Stylos transport integration
- one Themion process opens one Stylos session
- that one process may report multiple in-process agents through a shared process snapshot

This is an important design choice: Stylos is used as the network-visible coordination and observability layer around Themion, not as a replacement for Themion’s local runtime.

## Actual process identity used by Themion

Themion derives two related identifiers in `crates/themion-cli/src/stylos.rs`:

- `identity_instance = <hostname>`
- `key_instance = <hostname>:<pid>`

It constructs the underlying Stylos session config with:

- `realm = settings.realm()`
- `role = "themion"`
- `instance = identity_instance`

So the underlying Stylos identity uses a hostname-shaped instance value.

For network-visible Themion keys, it separately uses a transport-safe direct-instance identifier:

```text
<hostname>:<pid>
```

That key instance is used in published status keys and direct per-instance query keys so multiple local Themion processes on the same host can be distinguished cleanly.

## Session startup pattern

In `start_inner(...)` in Themion’s `crates/themion-cli/src/stylos.rs`, Themion builds a Stylos session config like this:

- Stylos identity:
  - realm from Themion settings
  - role `themion`
  - instance from sanitized hostname
- Zenoh section:
  - mode from Themion settings
  - connect endpoints from Themion config
  - listen endpoints left empty so Stylos can choose defaults
  - scouting left as `None`

Then it calls:

- `stylos::open_session(&cfg, &overrides).await`

This means Themion intentionally reuses the Stylos crate’s built-in session-opening behavior rather than reimplementing Zenoh configuration itself.

## One Stylos session per Themion process

Themion’s implemented model is:

- one process
- one Stylos session
- many possible in-process agents reported through that one session

This is described in Themion’s architecture docs and reflected directly in the status payload types in `stylos.rs`.

The key design outcome is that Themion does **not** create one Stylos session per agent. Instead, the process is the network node, and agents are process-local entities described inside payloads.

## Status publication model

Themion publishes periodic status from a background task.

In Themion’s `crates/themion-cli/src/stylos.rs`, it constructs a status key of the form:

```text
stylos/<realm>/themion/<hostname>:<pid>/status
```

Every 5 seconds it publishes a CBOR-encoded `ThemionStatusPayload` containing:

- `version`
- `instance`
- `realm`
- `mode`
- `startup_project_dir`
- `agents`

This is the implemented process-level status model.

## Multi-agent reporting over one process session

Themion’s status payload contains an `agents` array rather than flattening one agent into top-level fields.

Each agent snapshot includes implemented fields such as:

- `agent_id`
- `label`
- `roles`
- `session_id`
- `workflow`
- `activity_status`
- `activity_status_changed_at_ms`
- `project_dir`
- `project_dir_is_git_repo`
- `git_remotes`
- `provider`
- `model`
- `active_profile`
- `rate_limits`

This is the concrete shape of Themion’s Stylos reporting model: Stylos sees one Themion process with a structured list of local agents.

## Default single-agent boot behavior

Even though the reporting model supports multiple agents, the docs and code indicate the first shipped step still boots with a main interactive agent.

Themion’s documented initial roles are:

- `main`
- `interactive`

So the current case is effectively:

- one Stylos session for the process
- one main local agent in normal use
- but a payload model already shaped for more than one agent

This is a practical case study in how an application can start single-agent while adopting a multi-agent-capable network shape early.

## Discovery queryables implemented by Themion

Themion registers mesh-wide discovery queryables under:

```text
stylos/<realm>/themion/query/agents/alive
stylos/<realm>/themion/query/agents/free
stylos/<realm>/themion/query/agents/git
```

These are implemented in the queryable task inside Themion’s `crates/themion-cli/src/stylos.rs`.

### `alive`

`alive` returns one reply per responding instance with:

- instance identity
- session id
- current local agent list

This allows a caller to discover which Themion instances and agents are currently present on the mesh.

### `free`

`free` uses the current exported activity state and filters agents to those whose `activity_status` is:

- `idle`
- `nap`

That means Themion’s practical definition of “free for work” is not a separate scheduler flag. It is derived from the reported local activity state.

### `git`

`git` discovery returns agents whose:

- `project_dir_is_git_repo == true`

It can also filter by a requested remote/repo identity.

The reply includes:

- `git_remotes`
- normalized `git_repo_keys`

This makes repo-aware agent discovery possible without requiring exact raw-remote string equality.

## Direct per-instance query surface

Themion also implements direct per-instance queryables under:

```text
stylos/<realm>/themion/instances/<hostname>:<pid>/query/status
stylos/<realm>/themion/instances/<hostname>:<pid>/query/talk
stylos/<realm>/themion/instances/<hostname>:<pid>/query/tasks/request
stylos/<realm>/themion/instances/<hostname>:<pid>/query/tasks/status
stylos/<realm>/themion/instances/<hostname>:<pid>/query/tasks/result
```

These are not discovery queries. They are directed at one chosen Themion process instance.

## How `status` works in Themion

The direct `status` query returns the current process snapshot and supports optional filtering by:

- `agent_id`
- `role`

The implementation builds a filtered view of the current local process snapshot and returns:

- `found`
- `instance`
- `session_id`
- `startup_project_dir`
- `agents`
- optional `error`

So Themion’s direct status query is a structured runtime inspection API over Stylos.

## How `talk` works in Themion

Themion’s `talk` query is a remote prompt injection path into the existing local agent runtime.

The implemented behavior is:

1. parse the request containing `agent_id`, `message`, and optional `request_id`
2. find the target local agent in the current snapshot
3. reject if the agent does not exist
4. reject if the agent is not currently `idle` or `nap`
5. if accepted, enqueue the message through a local prompt bridge
6. return an acknowledgement, not the final answer

This is a very important actual design detail: Themion does **not** implement a second remote execution engine for Stylos requests. It feeds accepted remote talk requests into the same local runtime path used for normal agent execution.

## How task submission works in Themion

Themion’s `tasks/request` query supports structured remote delegation.

The request payload may include:

- `task`
- `preferred_agent_id`
- `required_roles`
- `require_git_repo`
- `request_id`

The implemented behavior is:

1. collect candidate local agents
2. keep only agents whose activity status is `idle` or `nap`
3. optionally filter by preferred agent id
4. optionally filter by required roles
5. optionally require that the agent’s project dir is a git repo
6. sort candidates by `agent_id`
7. choose the first deterministic match
8. allocate a `task_id`
9. record that task in the in-memory task registry as `queued`
10. enqueue the task through the same local prompt bridge used for remote delivery
11. return an acceptance/rejection payload

This shows a concrete and restrained use of Stylos: remote peers can request work, but the actual execution stays entirely local to the selected Themion agent.

## Task lifecycle tracking in Themion

Themion keeps task lifecycle state in memory, in-process.

The `TaskRegistry` in Themion’s `crates/themion-cli/src/stylos.rs` stores entries with:

- `task_id`
- `state`
- `agent_id`
- optional `result`
- optional `reason`
- update timestamp

Implemented lifecycle operations include:

- `insert_queued`
- `set_running`
- `set_completed`
- `set_failed`
- `get`
- `wait_for_terminal`

Records expire after a retention period of 30 minutes.

This means Themion implemented a lightweight, non-durable task observation layer on top of Stylos requests rather than a persistent distributed job queue.

## `tasks/status` and `tasks/result`

Themion separates task submission from status lookup and result waiting.

### `tasks/status`

Returns the current known lifecycle state for a task id without waiting.

### `tasks/result`

Supports:

- immediate return for terminal tasks
- immediate return of current non-terminal state when `wait_timeout_ms` is omitted or zero
- bounded waiting for terminal completion when `wait_timeout_ms` is positive

The wait timeout is clamped to 60,000 ms.

This is a concrete implemented request/result model that keeps submission separate from waiting.

## Git-aware agent discovery in practice

Themion implements git remote normalization helpers in `stylos.rs`.

Supported comparable normalization covers common forms for:

- `github.com`
- `gitlab.com`
- `bitbucket.org`

Examples described in code/tests/docs include normalization of forms such as:

- `git@github.com:example/themion.git`
- `https://github.com/example/themion`
- `ssh://git@gitlab.com/group/proj.git`
- `git@bitbucket.org:team/repo.git`
- direct comparable identities like `github.com/example/themion`

Themion uses this to derive `git_repo_keys` and to match requested repo identity against agents more usefully than exact raw URL comparison would allow.

## Self-exclusion behavior in discovery tools

Themion’s injected discovery tools default to excluding the local instance.

The important implementation detail is that self-exclusion is based on the decoded discovery reply payload’s `instance` field, not on parsing transport reply keys.

This is a concrete application-level decision built on top of Stylos query behavior to make agent-facing discovery safer and more useful.

## Injected Stylos tools in Themion

Themion exposes matching Stylos-oriented tool operations to agents, including:

- `stylos_query_agents_alive`
- `stylos_query_agents_free`
- `stylos_query_agents_git`
- `stylos_query_nodes`
- `stylos_query_status`
- `stylos_request_talk`
- `stylos_request_task`
- `stylos_query_task_status`
- `stylos_query_task_result`

The runtime bridge in Themion’s `crates/themion-cli/src/stylos.rs` translates these tool calls into direct Stylos queries.

For per-instance operations, Themion uses exact instance-addressed query keys and expects one logical reply. It treats:

- zero replies as timeout/offline/no responder
- one reply as normal success/error payload
- multiple replies as a protocol/configuration error

This is a strong case study of how an app can expose Stylos-backed functionality into an agent tool palette without collapsing transport code into the core agent loop.

## What Themion did not move onto Stylos

Themion did **not** move its core runtime onto Stylos.

Per the implementation/docs reviewed:

- model/provider calls remain local Themion runtime behavior
- tool execution remains local Themion runtime behavior
- workflow state ownership remains local to Themion agents
- persistent history remains SQLite-based
- Stylos is used as mesh visibility, query, and delegation plumbing around that runtime

This is one of the clearest lessons from the case study: Stylos is being used as a coordination/reporting layer, not as a replacement for the app’s internal execution model.

## Practical lessons from the Themion case

Themion demonstrates several useful patterns for applications embedding Stylos:

### 1. One process can be one Stylos node even if the app has multiple local agents

Themion avoids over-fragmenting the network model. The process is the node; the agents are payload-level entities.

### 2. Direct instance addressing can coexist with discovery queries

Themion uses:

- mesh-wide discovery keys for finding candidates
- exact per-instance keys for targeted inspection and requests

That is a clean split between discovery and RPC-like use.

### 3. Remote work requests should enter the normal local runtime path

Themion does not create a parallel Stylos execution path. It validates and enqueues remote requests into the existing local agent runtime.

### 4. Lightweight in-memory task tracking is enough for a first delegation layer

Themion tracks remote task lifecycle without pretending to be a durable distributed queue.

### 5. Repo-aware discovery becomes much more useful once remotes are normalized

Themion’s `git_repo_keys` support turns “who has this repo?” into a practical query.

## Summary

Themion is a concrete implemented consumer of Stylos that uses it for:

- process/session presence
- multi-agent status reporting
- mesh discovery
- direct process queries
- lightweight remote prompting
- lightweight remote task delegation

Its implementation shows a pragmatic integration model:

- keep core execution local
- use one Stylos session per process
- report multiple local agents through one process snapshot
- separate discovery from direct instance queries
- route accepted remote work into the normal local runtime

That makes Themion a strong real-world case study for Stylos as an application embedding layer rather than a standalone protocol abstraction.
