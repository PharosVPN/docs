# Project status

> **PharosVPN is pre-alpha.** It is suitable for experimentation, self-hosting
> tinkering, and code review — **not** production privacy or security use.
> Nothing here has been independently audited.

Legend: **✅ implemented + live-proven** · **🟡 implemented, lightly tested** ·
**🧪 experimental / in progress** · **📋 planned**

## Controller — coxswain

| Capability | Status |
|---|---|
| Fleet provisioning (servers → nodes), control plane over outbound mTLS | ✅ |
| Per-user / per-device **profiles** (egress + entry-IP subset + protocol) | ✅ |
| Multi-hop **cascade** paths (entry → mid → exit) | ✅ (2- and 3-hop proven live) |
| Account **sync** — end-to-end-sealed profiles, controller stores only ciphertext | ✅ |
| Embedded + remote **relays**, SSH enrollment | ✅ |
| Control-plane **onion routing** across relays | 🟡 (implemented; live test partial) |
| Admin **web UI** | ✅ |
| Backup/restore of controller state + keys; Postgres backend | 📋 |

## Data plane — node

| Capability | Status |
|---|---|
| **AmneziaWG** (obfuscated WireGuard) tunnels | ✅ |
| **XRay (VLESS + REALITY)** tunnels | ✅ (proven on one node) |
| Cascade transit (inner links, fwmark routing) | ✅ |
| Node identity: locally generated key, controller-assigned cert | ✅ |

## Clients — caravel

| Client | Status |
|---|---|
| **caravel-mac** (macOS, SwiftUI) | ✅ import + cloud sync + connect (AmneziaWG/XRay/both), map, controller status |
| **caravel** core (Go, shared engine) | ✅ |
| **caravel-linux** (Wails + AppImage) | 🧪 in progress |
| **caravel-android** (Compose) | 🧪 in progress |
| **caravel-ios** (SwiftUI) | 🧪 in progress |
| **caravel-opnsense / caravel-openwrt** | 📋 design + scaffold |

## Tested platforms / flows

- **Live-proven** (on DigitalOcean droplets): single-hop egress, 2-hop and 3-hop
  cascade egress, XRay/REALITY egress, account sync + profile delivery, the
  macOS client connecting over both protocols.
- Controller + nodes run on Linux (amd64). The macOS client runs on Apple
  Silicon + Intel (universal).

## Not done yet (so you're not surprised)

- Signed releases, checksums, SBOMs, dependency scanning, third-party audit.
- The residual risks in the [threat model](./threat-model.md) (e.g. guard-relay
  origin exposure).
- Mobile + Linux clients (in progress), controller backup/restore, Postgres.
- Any claim of production-grade privacy or security.

See the [threat model](./threat-model.md) for what is and isn't protected, and
[SECURITY.md](https://github.com/PharosVPN/.github/blob/main/SECURITY.md) to
report issues.
