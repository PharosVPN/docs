# PharosVPN — feature inventory (advertising source of truth)

This is the canonical list of capabilities for the website + docs to draw from.
**Advertise only what's marked SHIPPED.** Items marked COMING / PLANNED are not
yet real — do not put them in marketing copy until they move to SHIPPED (we
deliberately avoid over-claiming; see the false-advertising audit, 2026-06-07).

Status legend: **SHIPPED** (live + proven) · **PARTIAL** (works, lightly tested)
· **COMING** (in active build) · **PLANNED** (designed, not built).

---

## Always-on, self-healing control plane — SHIPPED
The controller stays up and continuously guarantees the fleet is correct.
- **Reconcile sweep** — the controller checks every node on an interval and
  **auto-heals drift** (a node serving stale config) and **stale data planes**
  (peers present but nothing handshaking), with per-node rate-limiting.
- **Zero-touch provisioning** — creating/removing a profile or device **pushes to
  the affected nodes automatically** (no manual step); if a node is unreachable
  the sweep delivers when it returns.
- **Restart resilience** — controller restart immediately re-reconciles the fleet;
  a node restart auto-recovers and is flagged `Unreachable` while it's gone.
- **Drift-aware status** — `cox nodes status` surfaces applied-vs-intended config
  revision and live handshake liveness; a silently-broken node shows red, not green.
- **Graceful degradation** — if the controller is briefly down, nodes keep serving
  existing tunnels from persisted config (resilience, not "turn it off").
- *(Battle-tested live: provision→auto-deliver, drift→heal, kill-daemon→recover,
  kill-controller→re-reconcile, cascade integrity preserved.)*

## Enterprise management & security — SHIPPED (Phase A)
- **Token-authenticated management API** — programmatic API secured by API tokens
  (alongside browser session login).
- **Scoped tokens with expiry** — least-privilege scopes (`readonly` / `monitor` /
  `admin`), optional expiry, **hashed at rest** (plaintext shown once), revocable.
  `cox tokens create|list|revoke`.
- **First-class, tamper-evident audit log** — **every** management action (API and
  CLI) is logged: who, what, to what, from where, success/failure. Rows are
  **hash-chained** so edits/deletions are detectable (`cox audit verify`).
  Queryable (`GET /api/audit`, `cox audit`); structured logs to journald;
  retention-bounded. *(Hardened via an adversarial security audit — see below.)*
- **Hardened access** — the session cookie is `Secure` over TLS; the dashboard is
  reached only over TLS or an SSH-forwarded loopback port; failed logins never log
  attacker-controlled input as the actor.

## Live monitoring — SHIPPED (Phase B)
- Real-time monitoring of every node over the WatchEvents **gRPC stream**: client
  **connect/disconnect** carrying the per-session **source IP** + resolved
  **device/user**, plus handshake up/down. *(Verified live: a client connect
  surfaced as a persisted, device-attributed session.)*
- **Persisted session history** — every connect/disconnect is stored
  (`connection_events`), queryable via `GET /api/sessions` (monitor scope).
- **Live event stream** for consumers — SSE/WebSocket for dashboards **and a
  first-class gRPC stream** for SIEM/enterprise ingestion (monitor-scope token,
  off-by-default, TLS for production). *(Verified live: a real connect streamed
  through gRPC to a token-authenticated consumer.)*

## Smart analytics / anomaly detection — SHIPPED (Phase C, Tier-1)
An in-process engine sweeps the session history (every 60s) and raises alerts.
**Live since Phase C** (the four Tier-1 rules):
- **Leaked-profile detection** — one profile from multiple source IPs in a window.
  *(Proven live: two devices sharing one key were flagged with full evidence.)*
- **Impossible travel** — geographically impossible IP changes (geo-velocity).
- **Concurrent sessions across nodes**.
- **New geo** — a device connecting from a country it's never used.

**Tier-2 (SHIPPED v0.3.0):**
- **Revoked-profile-active** (critical) — a revoked/disabled device still connecting.
- **Auth-failure spike** (warning) — brute-force / credential-stuffing (≥5 failed
  logins from one source IP in a window).
- **Dormant-then-active** (info) — a long-idle device suddenly reconnecting.
- **Off-hours access** (info) — a connect well outside the device's learned active
  hours (conservative, low-false-positive baseline).
- **Fleet-health** (warning/critical) — a node Unreachable/Error, **auto-resolved**
  when it recovers.

- Alerts carry severity + evidence, surfaced via `GET /api/alerts`, the live stream
  (SSE/WS + gRPC SIEM), the `cox alerts` CLI, and the dashboard; **Ack/Resolve**.
- **Postgres posture**: the engine **warns when run on SQLite** and recommends
  Postgres for production/enterprise scale.

**PLANNED (Tier 3–4):** geo-fence / concurrent-device policy, cert-expiry-in-use,
data-volume/exfil anomalies (needs a metrics-persistence pipeline), and deeper
fleet-health (handshake-success drop, cascade degradation, capacity).

## Management dashboard — SHIPPED
The self-hosted web dashboard surfaces the whole control plane: fleet/paths/
profiles plus **Live & sessions** (real-time connect/disconnect feed + history),
**Alerts** (severity, evidence, ack/resolve), **Audit log** (filterable), and
**API tokens** (create with scope/expiry, secret shown once, revoke). Served
locally on the controller (zero inbound).

## Data plane — SHIPPED (onion PARTIAL)
- **Dual-protocol** — **AmneziaWG** (obfuscated WireGuard, DPI-resistant) and
  **XRay / VLESS+REALITY**; chosen per profile, or **both**.
- **Multi-hop cascades** — entry → [mid] → exit routed server-side; the client
  dials only the entry. Proven live (2- and 3/4-hop).
- **Hop-aware MTU** — cascade profiles size their MTU to the path so large packets
  don't blackhole (verified live end-to-end).
- **Controller-hiding** — the controller dials OUT (zero inbound ports) and can be
  hidden behind **relays** with **onion routing** (PARTIAL — implemented,
  multi-relay onion lightly tested).
- **Endpoint rotation / unlinkability posture** — per-server keys, endpoint pool
  rotation, no node→controller trace.

## Clients — SHIPPED v0.1.0 (pre-alpha)
- **macOS** (signed + notarized DMG), **Linux** (AppImage x86_64 + aarch64),
  **Android** (APK), **iOS** (signs for device; TestFlight pending an App Group),
  **OpenWRT** (installable LuCI app `.ipk`), **OPNsense** (installable `os-pharosvpn`
  plugin with firewall/NAT/policy-route wiring). Shared Go core engine.

## Architecture & ops — SHIPPED
- **Single static binary** per component (`CGO_ENABLED=0`, pure-Go incl. SQLite) —
  no runtime deps.
- **Self-hostable** — your controller, your nodes, your keys; no PharosVPN servers
  in the path.
- **End-to-end sealed profiles** — the controller stores only ciphertext; profiles
  decrypt on the device.
- **Per-repo semantic versioning** + signed GitHub releases.
- **SQLite by default; optional pure-Go Postgres backend** for always-on/
  enterprise scale (pgx, selected by DSN — single static binary preserved,
  verified against live Postgres incl. the tamper-evident audit chain).

---

> Maintenance: update this file as features move between statuses. It is the
> source the website (`website/`) and public docs should advertise from — and the
> guardrail against advertising anything not yet SHIPPED.
