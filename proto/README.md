# PharosVPN protobuf contracts

The canonical home for the platform's wire contracts (DESIGN §6). gRPC over
mTLS; these schemas are the contract — no subproject hand-rolls a message type
that crosses a process boundary.

## Layout

```
proto/pharos/buoy/v1/control.proto   buoy node control service (helm ↔ buoy)
```

## Rules

- Schemas are versioned by package path (`...v1`). Add fields, never repurpose
  tags. Unknown fields and enum values are ignored, never rejected (DESIGN §11).
- `helm` owns these contracts; `buoy`, `beacon`, and `caravel` consume them.
- Each consuming repo generates its own code (committed in-repo) from a copy of
  these schemas. `buf` managed mode sets the per-repo Go import path.
- May graduate to a dedicated `proto` repo later.
