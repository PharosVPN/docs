# PharosVPN · docs

The canonical design and build conventions for the [PharosVPN](https://github.com/PharosVPN)
platform. Every subproject defers to this repo.

- **[DESIGN.md](DESIGN.md)** — the platform architecture. Single source of truth.
- **[BUILD.md](BUILD.md)** — global build conventions all repos follow.
- `proto/` — shared gRPC/protobuf contracts *(added as the wire protocol lands)*.

If code and `DESIGN.md` disagree, the document is wrong — fix it in the same PR.

## The platform in one paragraph

PharosVPN is a self-hostable, open-source, dual-protocol (AmneziaWG + XRay/REALITY)
VPN fleet platform. A private controller (`coxswain`) drives a fleet of dumb public
VPN nodes (`buoy`) over outbound mTLS, exposes end-users through an optional
relay (`beacon`), and serves them a mobile client (`caravel`). One codebase, two
postures — personal and enterprise.

## Repos

| Repo | Role |
|---|---|
| [`coxswain`](https://github.com/PharosVPN/coxswain) | Controller / management plane + admin UI |
| [`buoy`](https://github.com/PharosVPN/buoy) | VPN node agent |
| [`beacon`](https://github.com/PharosVPN/beacon) | Relay |
| [`caravel`](https://github.com/PharosVPN/caravel) | Mobile client |

## License

AGPL-3.0-or-later. Contributions under the DCO (`git commit -s`). No CLA.
