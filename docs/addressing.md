# Stylos Addressing

## Identity

Every Stylos process uses an identity tuple:

- `realm`
- `role`
- `instance`

Each segment must match:

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

## Wildcards

Stylos uses raw zenoh key expressions under the `stylos/...` namespace.

| Expression | Matches |
| --- | --- |
| `stylos/dev/watcher/host-a-42` | one specific instance |
| `stylos/dev/watcher/*` | all watchers in `dev` |
| `stylos/dev/*/*/status` | status keys for all roles in `dev` |
| `stylos/**` | the whole Stylos namespace |
