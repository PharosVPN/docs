# Project status

> **PharosVPN is pre-alpha.** It is suitable for experimentation, self-hosting
> tinkering, and code review — **not** production privacy or security use.
> Nothing here has been independently audited.

Legend: **✅ implemented + live-proven** · **🟡 implemented, lightly tested** ·
**🧪 experimental / best-effort** · **📋 planned**

The controller (`coxswain`) is the most mature component — a feature-rich
control plane (currently `v0.4.x`). The clients are `v0.1.0`. "Mature" here is
relative to the rest of the project; the pre-alpha disclaimer above still holds
for everything, and none of it has had a third-party audit.

## Controller — coxswain

### Fleet & data-plane control

| Capability | Status |
|---|---|
| Fleet provisioning (servers → nodes), control plane over outbound mTLS | ✅ |
| Per-user / per-device **profiles** (egress + entry-IP subset + protocol) | ✅ |
| Multi-hop **cascade** paths (entry → mid → exit), hop-aware MTU | ✅ (2- and 3-hop proven live) |
| Account **sync** — end-to-end-sealed profiles, controller stores only ciphertext | ✅ |
| Embedded + remote **relays**, SSH enrollment | ✅ |
| Control-plane **onion routing** across relays | 🟡 (implemented; multi-relay onion lightly tested) |
| Admin **web UI / management dashboard** | ✅ |

### Always-on, self-healing control plane

| Capability | Status |
|---|---|
| **Reconcile sweep** — interval check of every node, auto-heals config drift + stale data planes (rate-limited) | ✅ |
| **Zero-touch provisioning** — create/remove profile or device auto-pushes to affected nodes; unreachable nodes get it when they return | ✅ |
| **Restart resilience** — controller restart re-reconciles; a node restart auto-recovers, flagged `Unreachable` while gone | ✅ |
| **Drift-aware status** — `cox nodes status` surfaces applied-vs-intended revision + live handshake liveness | ✅ |
| **Graceful degradation** — controller briefly down ⇒ nodes keep serving existing tunnels from persisted config | ✅ |

*(Live-chaos-tested on the DO fleet: provision→auto-deliver, drift→heal,
kill-daemon→recover, kill-controller→re-reconcile, cascade integrity preserved.)*

### Enterprise management & security

| Capability | Status |
|---|---|
| **Token-authenticated management API** (alongside browser session login) | ✅ |
| **Scoped tokens** (`readonly` / `monitor` / `admin`), optional expiry, hashed at rest, revocable (`cox tokens …`) | ✅ |
| **Tamper-evident audit log** — every API/CLI action logged (who/what/where/result), hash-chained, `cox audit verify` | ✅ |
| **Hardened access** — `Secure` cookie over TLS; dashboard only over TLS or SSH-forwarded loopback | ✅ |

### Live monitoring & session history

| Capability | Status |
|---|---|
| Real-time monitoring over the WatchEvents **gRPC stream** (connect/disconnect with source IP + resolved device/user, handshake up/down) | ✅ |
| **Persisted session history** (`connection_events`), queryable via `GET /api/sessions`, shown in the dashboard | ✅ |
| **Live event stream** for consumers — SSE/WebSocket for dashboards + a first-class **gRPC SIEM stream** (monitor-scope token, off by default, TLS for prod) | ✅ |
| **Per-session byte totals** (rx/tx per session, survives endpoint roams) | 🧪 best-effort accounting |

### Smart analytics / anomaly detection

| Capability | Status |
|---|---|
| **Tier-1** — leaked-profile, impossible-travel, concurrent-sessions-across-nodes, new-geo | ✅ (leaked-profile proven live) |
| **Tier-2** — revoked-profile-active, auth-failure spike, dormant-then-active, off-hours, fleet-health (auto-resolving) | ✅ (v0.3.0) |
| **Data-volume / exfil** rule (upload > 1 GiB *and* 10× device median, ≥20-session baseline) | 🧪 experimental (v0.4.1, best-effort; depends on the per-session byte totals above) |
| Alerts carry severity + evidence; surfaced via `GET /api/alerts`, the live stream, `cox alerts`, the dashboard; ack/resolve | ✅ |
| **Tier 3–4** — geo-fence / concurrent-device policy, cert-expiry-in-use, deeper fleet-health | 📋 |

### Persistence & ops

| Capability | Status |
|---|---|
| **Single static binary** (`CGO_ENABLED=0`, pure-Go incl. SQLite) | ✅ |
| SQLite by default; **optional pure-Go Postgres backend** (pgx, DSN-selected, single binary preserved) | ✅ (verified against live Postgres incl. the audit chain) |
| Backup/restore of controller state + keys | 📋 |

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
| **caravel-mac** (macOS, SwiftUI) | ✅ import + cloud sync + connect (AmneziaWG/XRay/both), map, controller status — **`v0.1.0` signed + notarized DMG released** |
| **caravel** core (Go, shared engine) | ✅ — exposes the same surface (sync, list, prepare, connect, controller-status) to every client; version embedded |
| **caravel-linux** (Wails + AppImage) | 🟡 built — **`v0.1.0` AppImage released for x86_64 + aarch64**; on-device TUN testing pending |
| **caravel-android** (Compose, VpnService) | 🟡 built — **`v0.1.0` debug APK released**; bundles the real Go engine (all 4 ABIs); on-device connect testing pending |
| **caravel-ios** (SwiftUI, NEPacketTunnel) | 🟡 **signed device build passes** (app + Packet Tunnel extension + engine framework, App Group registered, Team baked in); remaining = TestFlight upload / on-device run |
| **caravel-openwrt** | 🟡 client mode shipped — **installable LuCI app (`.ipk`)** + CLI + procd + uci + fw4 NAT/kill-switch. On the dev VM: UI loads end-to-end and `connect` brought **`awg0` up with a completed handshake** (single LAN node). Multi-hop fleet connect untested; server mode planned. |
| **caravel-opnsense** | 🟡 client mode shipped — **installable `os-pharosvpn` MVC plugin (FreeBSD `pkg`)** + `pharos-awg` daemon. On the dev VM: VPN → PharosVPN menu/API/configd verified, the daemon brings **`awg0` up serving the UAPI**, and enabling a client generates the gateway + outbound-NAT + policy-route + kill-switch into `pf` (no self-lockout, clean teardown). Real-fleet leak-test pending; server mode planned. |

## Tested platforms / flows

- **Live-proven** (on DigitalOcean droplets): single-hop egress, 2-hop and 3-hop
  cascade egress, XRay/REALITY egress, account sync + profile delivery, the
  macOS client connecting over both protocols, the always-on control plane
  (provision→auto-deliver, drift→heal, kill-daemon→recover,
  kill-controller→re-reconcile), a leaked profile flagged with full evidence, a
  real connect streamed through the gRPC SIEM stream to a token-authenticated
  consumer, and the Postgres backend (incl. the tamper-evident audit chain).
- Controller + nodes run on Linux (amd64). The macOS client runs on Apple
  Silicon + Intel (universal).

## Releases

Client builds are out as **`v0.1.0`** (pre-alpha); the controller is on the
`v0.4.x` line — see each repo's Releases page (linked from the
[org profile](https://github.com/PharosVPN)): macOS **signed + notarized**
`.dmg`, Linux `.AppImage` (x86_64 + aarch64), Android debug `.apk`. iOS builds
signed for device (App Group registered); TestFlight pending an App Store
Connect record. Every repo carries a `VERSION` file, tags releases `vX.Y.Z`,
and bumps via an interactive `scripts/bump-version.sh` (patch/minor/major).

## Not done yet (so you're not surprised)

- Play-signed Android, iOS TestFlight upload; checksums, SBOMs, dependency
  scanning, third-party audit. (The macOS DMG is Developer-ID signed + notarized.)
- The residual risks in the [threat model](./threat-model.md) (e.g. guard-relay
  origin exposure).
- On-device connect testing for the mobile + Linux clients; controller
  backup/restore.
- Per-session byte totals + the data-volume/exfil analytics rule are
  **experimental / best-effort** — treat their numbers as indicative, not
  authoritative.
- Tier 3–4 analytics (geo-fence / concurrent-device policy, cert-expiry-in-use,
  deeper fleet-health).
- Any claim of production-grade privacy or security.

See the [threat model](./threat-model.md) for what is and isn't protected, and
[SECURITY.md](https://github.com/PharosVPN/.github/blob/main/SECURITY.md) to
report issues.
</content>
</invoke>
