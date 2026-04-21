# Stylos Addressing

## Identity

Every Stylos process uses an identity tuple:

- `realm`
- `role`
- `instance`

For the canonical Stylos identity tuple, each segment must match:

```text
[a-z0-9][a-z0-9-]*
```

Validation lives in `stylos-identity`.

## Root key

The canonical root key is:

```text
stylos/<realm>/<role>/<instance>
```

Example:

```text
stylos/dev/watcher/host-a-42
```

## Application-level addressing under Stylos

In practice, applications may publish additional keys under the `stylos/...` namespace that extend beyond the canonical identity tuple.

A concrete implemented example is Themion, which:

- opens its Stylos session with a canonical validated identity tuple
- then uses an application-level per-process addressing component of the form `<hostname>:<pid>` in some published/queryable keys

Examples:

```text
stylos/<realm>/themion/<hostname>:<pid>/status
stylos/<realm>/themion/instances/<hostname>:<pid>/query/status
```

These should be understood as application-defined key shapes built on top of Stylos transport and naming conventions, not as a change to the canonical validated `realm` / `role` / `instance` identity grammar itself.

So, de facto:

- Stylos identity validation remains restricted for the actual identity tuple
- application subtrees under `stylos/...` may use broader path segment forms when an embedding application defines and consumes them consistently

## Wildcards

Stylos uses raw zenoh key expressions under the `stylos/...` namespace.

| Expression | Matches |
| --- | --- |
| `stylos/dev/watcher/host-a-42` | one specific instance |
| `stylos/dev/watcher/*` | all watchers in `dev` |
| `stylos/dev/*/*/status` | status keys for all roles in `dev` |
| `stylos/**` | the whole Stylos namespace |

Application-defined subtrees may also introduce deeper or non-canonical path segments beneath `stylos/<realm>/...` as needed by the embedding application protocol.

Application-defined segments such as `<hostname>:<pid>` are still a single key segment. As a result, zenoh single-segment wildcards such as `*` match them normally. For example:

```text
stylos/<realm>/themion/*/status
```

matches status keys published under application-defined instances like `<hostname>:<pid>`.

This wildcard behavior does not make `<hostname>:<pid>` part of the canonical Stylos identity grammar; it only describes how application-level key paths are matched.
