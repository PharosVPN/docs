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
| **caravel-mac** (macOS, SwiftUI) | ✅ import + cloud sync + connect (AmneziaWG/XRay/both), map, controller status — **`v0.1.0` signed DMG released** |
| **caravel** core (Go, shared engine) | ✅ — exposes the same surface (sync, list, prepare, connect, controller-status) to every client; version embedded |
| **caravel-linux** (Wails + AppImage) | 🟡 built — **`v0.1.0` AppImage released for x86_64 + aarch64**; on-device TUN testing pending |
| **caravel-android** (Compose, VpnService) | 🟡 built — **`v0.1.0` debug APK released**; bundles the real Go engine (all 4 ABIs); on-device connect testing pending |
| **caravel-ios** (SwiftUI, NEPacketTunnel) | 🟡 **signed device build passes** (app + Packet Tunnel extension + engine framework, App Group registered, Team baked in); remaining = TestFlight upload / on-device run |
| **caravel-openwrt** | 🟡 client mode shipped — **installable LuCI app (`.ipk`)** + CLI + procd + uci + fw4 NAT/kill-switch. On the dev VM: UI loads end-to-end and `connect` brought **`awg0` up with a completed handshake** (single LAN node). Multi-hop fleet connect untested; server mode planned. |
| **caravel-opnsense** | 🟡 client mode shipped — **installable `os-pharosvpn` MVC plugin (FreeBSD `pkg`)** + `pharos-awg` daemon. On the dev VM: VPN → PharosVPN menu/API/configd verified, the daemon brings **`awg0` up serving the UAPI**, and enabling a client generates the gateway + outbound-NAT + policy-route + kill-switch into `pf` (no self-lockout, clean teardown). Real-fleet leak-test pending; server mode planned. |

## Tested platforms / flows

- **Live-proven** (on DigitalOcean droplets): single-hop egress, 2-hop and 3-hop
  cascade egress, XRay/REALITY egress, account sync + profile delivery, the
  macOS client connecting over both protocols.
- Controller + nodes run on Linux (amd64). The macOS client runs on Apple
  Silicon + Intel (universal).

## Releases

First public builds are out as **`v0.1.0`** (pre-alpha) — see each repo's
Releases page (linked from the [org profile](https://github.com/PharosVPN)):
macOS **signed + notarized** `.dmg`, Linux `.AppImage` (x86_64 + aarch64),
Android debug `.apk`. iOS builds signed for device (App Group registered);
TestFlight pending an App Store Connect record. Every repo carries a `VERSION`
file, tags releases `vX.Y.Z`, and bumps via an interactive
`scripts/bump-version.sh` (patch/minor/major).

## Not done yet (so you're not surprised)

- Play-signed Android, iOS TestFlight upload; checksums, SBOMs, dependency
  scanning, third-party audit. (The macOS DMG is Developer-ID signed + notarized.)
- The residual risks in the [threat model](./threat-model.md) (e.g. guard-relay
  origin exposure).
- On-device connect testing for the mobile + Linux clients; controller
  backup/restore; Postgres backend.
- Any claim of production-grade privacy or security.

See the [threat model](./threat-model.md) for what is and isn't protected, and
[SECURITY.md](https://github.com/PharosVPN/.github/blob/main/SECURITY.md) to
report issues.
