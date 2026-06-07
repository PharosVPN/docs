# Threat model

What PharosVPN protects, what it doesn't, and what each party can see or do if
compromised. This distills the trust model in [DESIGN.md](./DESIGN.md) §3–§5 into
a reviewer-facing document and is **honest about current gaps** — see the
"Residual risks" and "Does NOT protect against" sections.

> **Pre-alpha.** Nothing here has been independently audited. See
> [STATUS.md](./STATUS.md). Do not rely on PharosVPN for production privacy or
> security yet.

## Parties

| Party | Trust | Role |
|---|---|---|
| **Operator** | trusted (it's you) | runs the controller; holds the Fleet CA key |
| **coxswain** (controller) | **trusted root** | provisions the fleet, signs identities, seals profiles. No inbound ports; assumes it will be attacked. Ephemeral — can be turned off after configuring. |
| **node** (data plane) | **untrusted public infra** | a rented VPS that terminates client tunnels and egresses. Acts only on cryptographically validated control-plane instructions. |
| **relay** (control plane) | **untrusted public infra** | forwards the control plane (and optionally hides the controller's origin). Sees metadata, never profile contents. |
| **caravel** (client) | trusted by its user | holds the device key + the account passphrase; decrypts profiles locally. |
| **network / observer** | adversary | can watch traffic at any public point. |

The design intent: the **controller is the only trusted server**; nodes and
relays are disposable, untrusted, and individually replaceable. Profiles are
**end-to-end sealed** so the controller stores only ciphertext.

## What is encrypted end-to-end

- **Profiles.** Sealed to the device's X25519 key (the user's private key is
  passphrase-wrapped). coxswain only ever stores/serves **ciphertext**; a
  controller compromise yields ciphertext, not profiles. (DESIGN §8–9.)
- **Data plane.** Client↔node traffic is AmneziaWG (obfuscated WireGuard) or
  XRay (VLESS + REALITY) — encrypted end to end on the wire.
- **Control plane.** mTLS (gRPC) between coxswain and the fleet, plus an optional
  onion wrapper across relays.

## Compromise containment (what each compromise yields)

- **A compromised node** gets: its own private key, the live traffic transiting
  it (as the **exit**, it sees client destinations — like any VPN exit), and the
  CA **certificate** (not the key). It **cannot**: control the fleet, reach or
  identify the controller, or impersonate another node (identities are
  controller-assigned). In a cascade, a single mid/exit hop sees only its
  segment, never the originating client.
- **A compromised relay** sees: control-plane **metadata** (that some fleet's
  traffic transits it). It does **not** see profile contents (ciphertext) or the
  data plane. With onion routing, a single relay learns only its adjacent hops.
- **A compromised controller** gets: the **Fleet CA key → full fleet
  compromise** going forward. It still **cannot read past profiles** (e2e-sealed;
  it only ever held ciphertext). This is the crown jewel — keep the controller
  private, and note it is **ephemeral by design** (configure, then power it off;
  the data plane keeps running autonomously).
- **A compromised client/device** exposes that device's profiles + passphrase.
  Endpoint security is the user's responsibility.

## What metadata each party sees

- **coxswain:** the full fleet topology, which users/devices exist, provisioning
  state. It is **not in the data path** and does not see user traffic.
- **nodes:** the tunnels they terminate; the exit sees destination IPs/ports
  (not the user's identity, if the deployment is unlinkable).
- **relays:** that control-plane (and, if used as egress, data-plane ingress)
  traffic passes through — endpoints + volume + timing, never contents.

## Unlinkability posture and residual risks

**Goal:** no node ever traces to the controller; no cross-node correlation; the
controller's origin is hidden behind relays. Tiers (DESIGN §3): direct →
nested-TLS egress → onion routing (relay-operator-resistant).

**Residual risks — known and honestly unsolved:**

1. **Guard exposure (unsolved).** The first relay the controller dials (the
   guard) still sees the controller's IP. Onion routing hides the controller
   from *downstream* relays and from nodes, but not from the guard. The fix
   (dial the guard via Tor / a disposable egress) is **not yet implemented**.
2. **Fleet CA links servers.** Every node/relay certificate chains to one Fleet
   CA, so an observer who collects certs can tell they belong to one fleet.
   Per-server CAs (deferred) would break this linkage.
3. **Traffic correlation / timing.** A global passive adversary correlating
   ingress/egress timing and volume is **out of scope** (true of all VPNs).
   Cascades raise the bar but do not defeat a global adversary.
4. **Exit visibility.** The exit node sees your destinations (not who you are, in
   an unlinkable deployment). Don't send plaintext secrets over any VPN.
5. **Enrollment SAN handling (code hardening).** `pki.SignNodeCSR` currently
   *includes the SANs requested in the CSR* (in addition to the address coxswain
   pins itself). In the current SSH-driven enrollment the controller generates
   the CSR, so SANs are not attacker-chosen — but the signer should **assign**
   SANs (as `SignRelayCSR` already does) rather than copy them, before any flow
   accepts a CSR from an untrusted party. Tracked hardening item.

## What PharosVPN does NOT protect against

- A **global passive adversary** doing traffic correlation.
- A **compromised controller** (it is the root of trust).
- A **malicious exit node** observing your plaintext destinations.
- **Endpoint compromise** (your own device).
- The **guard relay** learning the controller's origin (until controller-origin
  hiding ships — risk #1 above).
- Anything implied by "pre-alpha and unaudited."

## Reporting

Security issues: see [SECURITY.md](https://github.com/PharosVPN/.github/blob/main/SECURITY.md).
Please report privately and allow time to fix before public disclosure.
