# Stylos Discovery

Zenoh calls this mechanism **scouting**. In prose, Stylos docs use **discovery**.

## LAN multicast

Default multicast discovery settings:

- group: `224.0.0.224:31746`
- multicast enabled
- gossip enabled

This supports LAN peer discovery without a central broker.

## Data listeners

Stylos uses UDP + TCP listeners on the same chosen port.

Default port:

- `31747`

Advertised endpoints:

```text
udp/0.0.0.0:31747
tcp/0.0.0.0:31747
```

If the default port is busy, Stylos walks forward in a small capped range.

## Failure modes

| Scenario | Behavior |
| --- | --- |
| multicast blocked | peers do not discover each other automatically |
| UDP blocked | TCP can still be used if peers connect explicitly |
| port 31747 busy | walk forward to the next free port |
