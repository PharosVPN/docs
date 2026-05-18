# PharosVPN — Platform Design

**Status:** draft v2 · 2026-05-17
**Supersedes:** `amnezia-travelvpn/NEW-PROJECT-DESIGN.md` (the v1 design notes).

This is the single source of truth for the PharosVPN platform. Every subproject
README and `BUILD.md` defers to this document. When code and this document
disagree, the document is wrong — fix it in the same PR.

---

## 0. What PharosVPN is

A self-hostable, open-source, dual-protocol VPN fleet platform. One codebase
serves two postures from the same binaries:

- **Personal** — "I want my own VPN," one operator, a handful of nodes.
- **Enterprise** — a team managing many users across many regions.

Defaults differ; the engine is identical.

The data plane is **AmneziaWG** (obfuscated WireGuard) and **XRay (VLESS +
REALITY)**, both terminating end-user tunnels on UDP/TCP 443. The platform is
the control plane, account system, and clients around that data plane.

---

## 1. Goals

- **Self-hostable in under 30 minutes.** Clone, follow the README, get a working
  fleet. No lock-in beyond the chosen cloud provider.
- **Defense-in-depth control plane.** The controller issues credentials and
  rotates server config — the highest-value target. It assumes it *will* be
  attacked: no inbound ports, no public DNS, no public IP.
- **Dumb nodes.** A compromised VPN node must not yield control of the fleet.
  Nodes act only on cryptographically validated instructions.
- **Operate with the controller offline.** If the controller is down, every node
  keeps serving existing tunnels indefinitely. Control plane, not data-plane
  dependency. The same applies to clients: a client connects from cached
  profiles when the account service is unreachable.
- **The controller never holds usable user secrets.** User profiles are
  end-to-end encrypted; a controller compromise yields ciphertext, not profiles.

---

## 2. The three node roles + clients

```
                  ┌──────────────────────────────────┐
                  │  helm  — CONTROLLER               │
                  │  (private network, behind NAT)    │
                  │  vpn-mgr daemon + admin Web UI     │
                  │  + SQLite state + CA + embedded    │
                  │    beacon relay (toggleable)       │
                  └───────┬──────────────────┬─────────┘
        mTLS, helm-initiated outbound        │ reverse tunnel
        gRPC/HTTP2 to each node              │ (helm dials OUT to a
                  │                          │  remote beacon)
        ┌─────────┼─────────┐                ▼
        ▼         ▼         ▼          ┌──────────────┐
   ┌────────┐┌────────┐┌────────┐      │ beacon RELAY │  (public)
   │  buoy  ││  buoy  ││  buoy  │ ...  │ mTLS ingress │
   │ NODE A ││ NODE B ││ NODE N │      │ for clients  │
   │ public ││ public ││ public │      └──────┬───────┘
   │ AWG udp ││  ...   ││  ...   │             │ mTLS
   │ XRay tcp││        ││        │             ▼
   └────┬───┘└────┬───┘└────┬───┘      ┌──────────────┐
        │ udp/tcp 443       │          │   caravel    │
        ▼        end users  ▼          │ mobile client│
     end-user tunnels                  └──────────────┘
```

| Role | Repo | Network posture | Job |
|---|---|---|---|
| **Controller** | `helm` | Private, behind NAT. Zero inbound ports. | Source of truth, admin UI, issues certs/profiles, drives the fleet. |
| **VPN node** | `buoy` | Public IP. Listens udp/tcp 443 + mTLS control port. | Runs the data plane. Dumb agent — applies only validated config. |
| **Relay** | `beacon` | Public. The only public ingress for *clients*. | mTLS-terminating proxy. Lets clients reach a NAT'd controller. Always embedded in `helm`; optionally deployed remote. |
| **Mobile client** | `caravel` | End-user device. | Runs the actual VPN tunnel + acquires profiles from multiple sources. |

**Key inversion:** the controller *dials out* to everything. Buoys are already
public (they must be, to terminate tunnels), so `helm` initiates outbound mTLS
to each buoy. `helm` also dials *out* to a remote `beacon` (reverse tunnel), so
the controller needs zero inbound ports anywhere.

**`beacon` embedded vs remote.** A `beacon` relay always runs in-process inside
`helm` (toggleable off in the admin UI). When the controller sits behind NAT and
must serve clients, deploy a **remote** `beacon` on a public host (its own VM, or
co-located on a `buoy`); `helm` dials out to it over a persistent reverse
tunnel. Embedded and remote are transport differences only — identical trust.

---

## 3. Component responsibilities

### helm (controller)

- **Source of truth.** SQLite holding fleet inventory, profiles, users, devices,
  peers, admins, sessions, the CA, audit log, metrics samples. See §10.
- **Admin Web UI** — SvelteKit SPA embedded in the binary, served on localhost.
- **Outbound control loop** — holds a long-lived mTLS/gRPC connection to each
  `buoy`; pushes config, pushes/revokes peers, and *receives a live event
  stream* (see §7).
- **Node onboarding over SSH** — installs and updates the `buoy` agent on
  operator-provided VMs over SSH; all node *control* is gRPC. See §5. helm
  does not call cloud-provider APIs — the operator creates the VM.
- **Issues** node certs, the controller's own client cert, relay certs, and
  per-user/device certs. Holds the CA. See §4.
- **Embedded `beacon`** and the reverse-tunnel dialer for remote `beacon`s.
- **Account & sync service** — authenticates users/admins, serves E2E-encrypted
  profile bundles (see §8). This surface is reached only via a `beacon`.

### buoy (VPN node agent)

- **Stateless except for what `helm` gave it.** All config is written to disk
  only after `helm` pushes it over mTLS.
- **Data plane:** `awg-quick@awg0` on UDP 443, `xray.service` on TCP 443.
- **Control port** (mTLS-only, gRPC). Operations: status, metrics, push config,
  add/remove peer (live, no restart), handshake stats, restart service, and a
  **server-stream of live events** back to `helm`.
- **SSH is install-only.** `helm` reaches a node over SSH solely to install and
  update the `buoy` agent; every operational instruction is gRPC.
- **Cold-start resilient.** Comes up from disk every boot using the last config
  `helm` pushed. Controller offline ⇒ existing peers keep working.

### beacon (relay)

- **Stateless public proxy.** Terminates client mTLS, forwards gRPC streams to
  `helm`. Holds no database; all lookups delegated to `helm`.
- Strips spoofable client metadata; injects exactly one trusted value: the
  verified **device fingerprint**.
- Two transports to `helm`: **embedded** (in-process, in-memory pipe) or
  **remote reverse tunnel** (`helm` dials out, multiplexed substreams).
- Carries only **ciphertext** profile bundles — see §8 — so a compromised remote
  `beacon` host cannot read user profiles.

### caravel (mobile client)

- **Two decoupled layers:** a VPN engine (establishes tunnels, multi-node /
  multi-protocol) and a set of pluggable **profile sources** (see §8).
- **Posture-aware:** *personal* (account login, QR, file import; admin section
  if the logged-in account is an admin) vs *managed* (MDM config present —
  account login and admin hidden, profiles locked). One app, one store listing.

---

## 4. Trust model & PKI

A single in-repo **root CA**, generated on `helm`'s first run, stored in `helm`'s
SQLite, never copied off the controller. Two intermediates under it:

- **Fleet CA** — issues `buoy` node certs, the controller's client cert, and
  `beacon` relay certs.
- **Device CA** — issues per-user/per-device leaf certs for `caravel` and the
  admin browser.

| Certificate | Issued by | Held by | Validity |
|---|---|---|---|
| Root CA | self-signed | `helm` only | 10 years |
| Fleet / Device intermediates | Root CA | `helm` only | 5 years |
| Controller client cert | Fleet CA | `helm` | 1 year, auto-rotated |
| Node server cert | Fleet CA | each `buoy` | 1 year, auto-rotated by push |
| Relay cert | Fleet CA | each `beacon` | 1 year, auto-rotated |
| Device leaf | Device CA | each `caravel` / browser | 1 year |
| helm SSH key | `helm` (self) | `helm` | long-lived, for agent deploy |

**Compromise containment:**
- Compromised `buoy` → attacker gets that node's key + the CA *cert* (not key).
  Cannot impersonate `helm` or other nodes. Operator revokes the node cert.
- Compromised remote `beacon` → attacker can see *traffic metadata* but profile
  bundles are E2E-encrypted ciphertext (§8). Cannot mint certs.
- Compromised `helm` → attacker gets the CA key. Fleet fully compromised — but
  user **profiles remain encrypted** (the controller never holds users' private
  keys in usable form, §8). Hence `helm`'s "no inbound ports, behind NAT" posture.

**Post-quantum hardening.** Every AmneziaWG peer is issued a unique 256-bit
**preshared key**, mixed into the WireGuard handshake. WireGuard's ECDH
(Curve25519) is quantum-vulnerable; the symmetric PSK is not — so recorded
tunnel traffic stays confidential against a future *harvest-now-decrypt-later*
attacker. `helm` generates the PSK per peer, ships it inside the E2E profile
bundle (§8), and pushes it to `buoy`. This is a pragmatic interim measure, not
a full post-quantum handshake.

---

## 5. Bootstrap & enrollment

**Node enrollment** — `helm nodes add <ssh-host>`:
1. The operator creates a VM on any provider and adds `helm`'s SSH **public
   key** (printed by `helm ssh-key`) to its `authorized_keys`. `helm` has its
   own SSH keypair, generated on first run and stored in SQLite.
2. `helm` connects out over SSH, pins the host key on first use (TOFU), and
   installs the `buoy` agent — either by uploading a bundled binary or running
   a one-line download.
3. `buoy` generates its own keypair **on the node** and emits a CSR. `helm`
   pulls the CSR back over SSH, signs it with the Fleet CA, and pushes the
   certificate plus the CA back. The node's private key never leaves the node
   and `helm` never holds it.
4. `helm` starts the `buoy` service. From here every instruction is gRPC.

SSH is a *deployment* channel only — install and update of the agent. There is
no enrollment-mode listener and no one-time bootstrap token; the trusted SSH
channel replaces both.

**Relay enrollment** — same SSH pattern for a remote `beacon`.

**User / device enrollment** — a user is given an **enrollment ticket** (QR or
deep link, see §9). `caravel` scans it, contacts `beacon`→`helm`, the device
generates a keypair, `helm` issues a Device-CA leaf, and the device is bound to
the user account. The enrollment ticket is the only moment of weakness: short
TTL, one-use, scoped.

---

## 6. Wire protocol

**gRPC over mTLS (HTTP/2, TLS 1.3).** Decided — not plain JSON. The live event
streaming in §7 needs server-streaming, which is native to gRPC and awkward
otherwise. Both Go ends make codegen cheap; `caravel` consumes generated
clients too.

- Protobuf schemas are the contract. They live in **`docs/proto/`** (may graduate
  to a dedicated `proto` repo). No subproject hand-rolls message types.
- Schemas are versioned; unknown fields are ignored, never rejected.

---

## 7. Real-time & multi-admin

The admin UI must feel **live** — a client connecting to a node appears
immediately, not on a 30-second poll.

- **`buoy` → `helm`:** `helm` holds its outbound mTLS connection open and the
  buoy **streams events** (handshake up/down, peer connect/disconnect, errors)
  over a gRPC server-stream. Polling remains only as a fallback heartbeat.
- **`helm` → browser:** every open admin page holds a WebSocket. `helm` pushes
  state changes to all of them — open the dashboard on three machines, all three
  update together.

**Optimistic concurrency.** Every mutable record carries a `version` integer.
A mutation must send the `version` the admin loaded. If `helm`'s current version
is higher, it rejects with **HTTP 409 Conflict** — "changed by someone else,
reload." Admin A editing a stale copy of a user is refused because Admin B
already bumped it. Live WebSocket replication makes conflicts rare (A's screen
usually updates before A saves); the version check is the hard safety net.

---

## 8. Accounts, profiles & sync

### Accounts and roles

`users` are authentication principals (unlike the old codebase, where only
admins logged in). On login the role decides the surface: `user` → own profiles
only; `admin` → also the admin console (web = full; `caravel` = a small
glance-and-quick-actions subset).

### Profile sources (the unified-client model)

`caravel` has a VPN engine that only ever reads a **local profile store**.
Profiles enter that store from interchangeable **sources** — "synced vs
unsynced" is just which sources are enabled, not two apps:

| Source | Audience | Mechanism |
|---|---|---|
| Account sync | Personal | login → `beacon`→`helm` → pull, E2E-decrypt on device |
| QR scan | Anyone | scan an enrollment ticket or a self-contained profile QR |
| File import | Anyone | open a `.pharos` file (Mail/Files/AirDrop/portal) |
| MDM managed config | Enterprise | MDM pushes profiles + policy into managed config |
| Deep link | Portal-driven | `pharosvpn://import?...` |

The account/sync service + `beacon` are an **optional platform component**: an
enterprise doing only MDM/QR runs no `beacon` and no account service.

### End-to-end profile encryption

Each **user** has a long-lived keypair. `helm` holds only the **public** key.
`helm` generates a profile, encrypts it to the user's public key, stores
ciphertext, discards plaintext. Only the user's devices decrypt. Hybrid envelope:

- Random data key → profile encrypted with **XChaCha20-Poly1305** (AEAD).
- Data key wrapped to the user's public key.
- Bundle signed.

**Private-key storage — DECIDED:** *passphrase-wrapped blob held by `helm`.* The
user's private key is encrypted with a key derived (**Argon2id**) from the user's
passphrase; `helm` stores only that opaque blob and never the passphrase or a
usable private key. Any new device unwraps it with the passphrase. This gives
seamless multi-device + recovery; a `helm` compromise yields only a
passphrase-encrypted blob. (Device-to-device-only transfer was the alternative —
stronger, but no recovery and clunkier enrollment; rejected for v1.)

---

## 9. The `.pharos` file format

**One extension: `.pharos`.** Encryption is a property *inside* the file, not a
separate extension (no `.spharos`). The app reads the header and auto-routes.

A `.pharos` file is a JSON container with an always-readable header:

```jsonc
{
  "fmt": "pharos-profile",
  "v": 1,
  "enc": "none" | "password" | "account",
  // password mode: "kdf": {argon2id params + salt}, "cipher", "nonce"
  // account mode:  "recipient", "wrapped_key", "cipher", "nonce"
  "payload": <profile object>  |  "<base64 AEAD ciphertext>"
}
```

| `enc` | Meaning | Client handling |
|---|---|---|
| `none` | Plaintext | Load directly |
| `password` | Argon2id → XChaCha20-Poly1305 | Prompt for password, decrypt |
| `account` | Per-user hybrid envelope (§8) | Decrypt silently with device key |

The header is fed as **AAD** so `enc`/`v`/KDF params are authenticated (no
downgrade). Content-sniff on `fmt` so renamed files still import. Register MIME
`application/vnd.pharosvpn.profile`, an iOS UTI, an Android intent filter.

**Future-proof protocol model.** The profile carries nodes, each with a
**versioned, tagged list** of protocols:

```jsonc
{ "nodes": [ { "id": "...", "endpoints": ["..."], "protocols": [
    { "type": "amneziawg",    "v": 2, "params": { } },
    { "type": "xray-reality", "v": 1, "params": { } }
] } ] }
```

Clients keep a registry of handlers keyed by `type`. **Ignore-unknown, never
reject:** an old client skips a protocol/node it can't handle. Adding a protocol
= a new `type` string + a handler, zero format break. Plus metadata: `user`,
`device`, `fleet_id`, `issued_at`, `expires_at`, `revision`.

### QR codes

Compression does **not** solve QR size — profiles are mostly incompressible key
material. A reliably-scannable QR holds ~150–300 bytes; a multi-node profile is
kilobytes. Two QR kinds:

- **Enrollment ticket** (default): relay endpoint + one-time claim token + CA
  fingerprint to pin. ~150 bytes, fixed size regardless of profile size. The app
  scans it and downloads the full `.pharos` over the network.
- **Self-contained `.pharos` QR** (offline/air-gapped): single-node fits one QR
  via CBOR/binary → deflate → base45 → alphanumeric QR. Multi-node offline ⇒
  multi-part / structured-append / animated QR ("scan 1 of N").

A separate **Amnezia-compatible `.vpn`** export remains for users on the Amnezia
client. `.pharos` is *our* format and is not asked to do that job.

---

## 10. Persistence

Single SQLite database on `helm` (`state/app.db`), Goose migrations.

Tables: `ca` (the root + intermediate CAs, §4), `nodes`, `profiles`, `users`,
`devices`, `peers`, `admins`, `sessions`, `node_certs`, `device_certs`,
`bootstrap_tokens`, `audit_log`, `metrics_samples`, `relays`. **Every mutable
row carries `version INTEGER` and `updated_at`** for
§7 optimistic concurrency. YAML projections under `state/snapshots/` continue
for git-friendly diffs. `buoy` and `beacon` have no database.

---

## 11. Failure modes

| Failure | Behaviour |
|---|---|
| `helm` crashes | All buoys keep serving tunnels. No new peers until back. |
| `helm` ↔ buoy unreachable | Buoy keeps serving. `helm` marks `unreachable` after 3 missed polls, alerts, retries with backoff. |
| `buoy` crashes | Its tunnels drop. Clients fail over to other nodes in the profile. |
| `buoy` compromised | Attacker has that node's keys, not the CA key. Operator revokes the cert. |
| Remote `beacon` compromised | Traffic metadata exposed; profile bundles are ciphertext. No cert minting. |
| `helm` compromised | Worst case. CA key lost → rotate CA, mass re-enroll. User profiles stay encrypted. |
| Account service / `beacon` down | `caravel` connects from cached local profiles. |

---

## 12. Defaults: personal vs enterprise

Same binaries, two presets at `helm init`:

|  | `--personal` | `--enterprise` |
|---|---|---|
| Regions | 1, nearest | operator picks |
| Idle nodes | none | encouraged (pre-positioned, stopped) |
| Protocols | AmneziaWG default, XRay optional | both |
| `beacon` | embedded | embedded + remote relays |
| Account sync | on | optional (MDM-only deployments run none) |
| Admins | one (the operator) | core admin + UI-added others |
| Audit retention | 30 days | 1 year |
| Metrics retention | 7 days | 90 days |
| REALITY decoy site | `www.microsoft.com` | configurable, rotated |

---

## 13. License & contribution

- **AGPL-3.0-or-later** for the entire platform, every repo. Rationale: the user
  wants commercial users to *contribute back*, not to pay. AGPL's network
  copyleft forces anyone running a modified version as a service to publish
  their changes — forced contribution, not forced payment.
- **DCO** (Developer Certificate of Origin, `Signed-off-by`) for contributions.
  No CLA — there is no plan to relicense or dual-license.
- Every source file carries a short AGPL header. Each repo ships `LICENSE`
  (full AGPL-3.0 text) and `CONTRIBUTING.md` (DCO instructions).

---

## 14. Repo map

| Repo | What | Stack | Owner |
|---|---|---|---|
| `docs` | This document, `BUILD.md`, protobuf contracts | Markdown / proto | core |
| `helm` | Controller / management plane + admin UI | Go + SvelteKit | core (you + Claude) |
| `buoy` | VPN node agent | Go | subagent |
| `beacon` | Relay | Go | subagent |
| `caravel` | Mobile client | native (Kotlin / Swift) | subagent |
| `.github` | Org profile | Markdown | core |

`buoy` and `beacon` lift and **rebrand** the reverse-tunnel, transparent-proxy,
and device-CA machinery from the private `sultix` project (same owner). All
`sultix` / `mcproxy` / `mctunnel` / `x-sultix-*` identifiers must be stripped.

---

## 15. Decisions log

| # | Decision | Date |
|---|---|---|
| 1 | Name: PharosVPN. Org `github.com/PharosVPN`. | 2026-05-17 |
| 2 | License AGPL-3.0-or-later + DCO, no CLA. | 2026-05-17 |
| 3 | Wire protocol: gRPC over mTLS (not plain JSON). | 2026-05-17 |
| 4 | Three roles: `helm` / `buoy` / `beacon`; client `caravel`. | 2026-05-17 |
| 5 | `beacon` always embedded in `helm`, optionally remote. | 2026-05-17 |
| 6 | Live UI: buoy→helm event stream + helm→browser WebSocket. | 2026-05-17 |
| 7 | Optimistic concurrency: per-row `version`, 409 on stale write. | 2026-05-17 |
| 8 | Per-user E2E profile encryption; hybrid envelope. | 2026-05-17 |
| 9 | Private key: passphrase-wrapped blob on `helm` (Argon2id). | 2026-05-17 |
| 10 | File format: single `.pharos` extension, `enc` in-header. | 2026-05-17 |
| 11 | Protocols: versioned tagged list, ignore-unknown. | 2026-05-17 |
| 12 | QR: enrollment ticket default; self-contained QR for offline. | 2026-05-17 |
| 13 | Reuse + rebrand `sultix` relay/tunnel/device-CA code. | 2026-05-17 |
| 14 | Node/relay onboarding over SSH (agent install + update); no cloud-provider API. Node keys are generated on-node and signed via CSR; no bootstrap token. Supersedes the §3 `CloudProvider` interface. | 2026-05-18 |
| 15 | Per-peer 256-bit AmneziaWG preshared keys, for post-quantum (harvest-now-decrypt-later) hardening of the data plane. See §4. | 2026-05-19 |

### Still open

- `caravel`: native per-platform vs Kotlin Multiplatform — decide in `caravel/BUILD.md`.
- Whether protobuf contracts graduate from `docs/proto/` to a dedicated repo.
- Metrics export: built-in dashboard only, or also Prometheus push.
