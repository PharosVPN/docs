# PharosVPN ⇄ OPNsense integration (design)

Status: **design / feasibility**, not implementation. Grounded in the live dev VM
`opnsense-dev` (192.168.0.224, OPNsense 25.7 / FreeBSD 14.3-RELEASE-p1 amd64) —
every "verified on the VM" claim below was checked there read-only.

The goal: let an OPNsense firewall be either a **client** of a PharosVPN fleet
(import a `.pharos` profile, route the LAN through the tunnel) or a **server
component** (run `node` / `relay` / `coxswain` on the box). The crux for both is
the same: how do we run **AmneziaWG** on FreeBSD?

---

## 1. What the platform gives us (verified on the VM)

| Capability | Finding | How verified |
|---|---|---|
| OS / arch | OPNsense 25.7, FreeBSD 14.3-RELEASE-p1, amd64, pkg ABI `FreeBSD:14:amd64` | `opnsense-version`, `freebsd-version -ku`, `pkg config ABI` |
| Kernel WireGuard (`if_wg`) | **Present**: `/boot/kernel/if_wg.ko` (123 KB, dated Jul 2025) | `find /boot -name if_wg.ko` |
| WireGuard userspace tool (`wg`) | **Present in base**: `/usr/bin/wg` = `wireguard-tools v1.0.20210914` | `wg --version` |
| `wg-quick` | **Absent** | not on PATH; OPNsense doesn't use it (see below) |
| AmneziaWG (`awg` / `awg-quick`) | **Absent**, and **not in the OPNsense pkg repo** | `pkg rquery` for `amnezia` → nothing |
| AmneziaWG kernel port for FreeBSD | **Does not exist** (no FreeBSD `if_awg` / DKMS-equivalent) | repo search empty; upstream ships Linux DKMS + Go userspace only |
| `tun`/`tap` driver | **Built in**: `if_tuntap` resident, `net.link.tun.devfs_cloning = 1` | `kldstat -v`, `sysctl net.link.tun` |
| WireGuard GUI | **Now in OPNsense core** (not a separate plugin anymore) — full MVC under `/usr/local/opnsense/.../Wireguard`, configd actions, `wg-service-control.php` | `find … -ipath '*Wireguard*'` |
| Go compiler | **Not in the OPNsense repo** (lives in FreeBSD ports `lang/go`, whose repo is disabled here). We **cross-compile on the Mac**, ship a built binary. | `pkg rquery` for `go1` → nothing; `/usr/local/etc/pkg/repos` shows FreeBSD repo `enabled: no` |
| PHP (plugin framework) | PHP 8.3.23 NTS | `php -v` |
| rc.d services present | `configd`, `lighttpd` (the GUI), `openvpn`, `strongswan`, `unbound`, … | `ls /usr/local/etc/rc.d` |

Two facts dominate the whole design:

1. **There is no AmneziaWG kernel module for FreeBSD and none in the repo.** The
   kernel `if_wg` is *plain* WireGuard — it has no Jc/Jmin/Jmax/S1-4/H1-4/I1-5
   obfuscation knobs. So the OPNsense-core WireGuard stack **cannot** carry a
   PharosVPN data plane. We cannot reuse it for the tunnel itself.
2. **We can run AmneziaWG in userspace.** `amneziawg-go` ships a native FreeBSD
   tun backend (confirmed: `…/amneziawg-go@v0.2.18/tun/tun_freebsd.go`, using
   `if_tuntap` cloning + the `TUNSIF*`/`SIOC*IFINFO_IN6` ioctls), and `if_tuntap`
   is resident in this kernel. caravel's core (`caravel/go`) already depends on
   exactly this version and is platform-agnostic. **This is the linchpin: the
   same Go data plane caravel uses on macOS compiles and runs on FreeBSD.**

### The OPNsense plugin model (verified by reading the in-core WireGuard plugin)

OPNsense's modern WireGuard integration is the canonical template for `os-pharosvpn`.
Its on-disk shape:

- **MVC (Phalcon/Volt PHP)** under `/usr/local/opnsense/mvc/app/{models,controllers,views}/OPNsense/Wireguard/`:
  - `models/.../Server.xml` + `Server.php` — the config schema (field types like
    `Base64Field`, `PortField`, `NetworkField`, `ModelRelationField`), mounted at
    an XML path in `config.xml` (`<mount>//OPNsense/wireguard/server</mount>`).
  - `models/.../Menu/Menu.xml`, `ACL/ACL.xml` — GUI menu entry + permissions.
  - `controllers/.../Api/*Controller.php` + `forms/*.xml` + `views/*.volt` — the
    GUI pages and the REST API the GUI calls.
  - `Migrations/M1_0_0.php` — schema migrations.
- **configd actions**: `/usr/local/opnsense/service/conf/actions.d/actions_wireguard.conf`
  maps action names → scripts, e.g.
  `[start] command:/usr/local/opnsense/scripts/Wireguard/wg-service-control.php  parameters: start %s`.
  Code triggers these with `configctl wireguard start <id>`.
- **service templates**: `/usr/local/opnsense/service/templates/OPNsense/Wireguard/*`
  render config files from the model into the filesystem.
- **The plugin hook file** `/usr/local/etc/inc/plugins.inc.d/wireguard.inc` — a set
  of well-known functions OPNsense auto-discovers and calls during reloads
  (verified bodies):
  - `wireguard_services()` → registers a service for the Services widget, wiring its
    `start`/`restart`/`stop` to configd actions.
  - `wireguard_interfaces()` → registers a virtual interface *group* `wireguard`
    (so firewall rules can target the group).
  - `wireguard_devices()` + `wireguard_prepare($device)` → declares devices matching
    `^wg` and creates them: literally
    `ifconfig wg create name <if>; ifconfig <if> group wireguard`.
  - `wireguard_configure()` → returns reconfigure hooks
    (`'vpn' => ['wireguard_configure_do']`, `'newwanip' => ['wireguard_sync']`) so
    the tunnels reconcile when the VPN subsystem or the WAN IP changes.

The FreeBSD data-plane bring-up pattern, read from `wg-service-control.php`:
`ifconfig wg create name wgN` → `wg syncconf wgN <conf>` → `ifconfig wgN inet <addr> alias`
→ set MTU → put it in `group wireguard`. We mirror this shape, but with a
**userspace `pharos-awg` daemon + tun** in place of `ifconfig wg create` + kernel `if_wg`.

---

## 2. AmneziaWG data-plane strategy (the central decision)

**Run `amneziawg-go` userspace on FreeBSD. Do not chase a kernel port.**

Rationale:
- No FreeBSD AmneziaWG kernel module exists and none is in the repo (verified). A
  DKMS-style kernel port is a multi-month FreeBSD-driver project, out of scope.
- `amneziawg-go` already has a working FreeBSD tun backend (verified source) and
  `if_tuntap` is resident in this kernel (verified). It is the *same* engine caravel
  ships, so there is exactly one obfuscation implementation across the whole
  platform — no second codebase to keep in sync with the Jc/S/H/I parameter
  semantics.
- Userspace cost is a userspace-WireGuard cost (extra copies, no kernel crypto
  offload). For a homelab/SMB firewall on udp/443 this is fine; it is the same
  tradeoff every wireguard-go/amneziawg-go deployment already accepts. Throughput
  ceiling is an open question to benchmark (§9), not a blocker.

Concretely there are two consumers of this engine, and they already exist in the
tree:

- **Client mode** uses caravel's `vp` engine verbatim: `caravel/go/vp/engine.go`
  `vp.Up(cfg, tunDev, logLevel)` builds the `amneziawg-go` device over a tun the
  caller created with `tun.CreateTUN(...)`, feeds it a UAPI string carrying the
  obfuscation params (`jc`, `jmin`, `jmax`, `s1..s4`, `h1..h4`, `i1..i5`) plus the
  single server peer. The macOS shell (`caravel-mac`) is the reference for owning
  the tun + routing; on FreeBSD the `ifconfig`/`route` syntax is **the same BSD
  command surface** the macOS shell already uses (see §3).

- **Server mode** (`node`) currently shells out to `awg`/`awg-quick`
  (`node/internal/awg/runtime.go`) — Linux-only assumptions baked in (`ip route`,
  `awg-quick` is bash, Linux `rp_filter` sysctls). On FreeBSD we either (a) ship
  the AmneziaWG userspace **tools** built for FreeBSD, or (b) — preferred long-term
  — add a FreeBSD `Runtime` that drives `amneziawg-go` directly. See §4.

### Shipping the engine: `pharos-awg`

Define one small Go binary, **`pharos-awg`** (built from caravel's `vp` package),
that:
- creates a FreeBSD tun (`tun.CreateTUN`), names it `awgN`,
- runs the `amneziawg-go` device with a UAPI built from a config file,
- exposes the wireguard-go **UAPI socket** (`/var/run/wireguard/awgN.sock`) so
  `wg`-style tooling and our status scripts can read `rx_bytes`/`tx_bytes`/handshakes,
- on shutdown closes the device and removes its routes.

This is essentially `amneziawg-go`'s own `main` plus a thin config loader; caravel
already has all the hard parts (`vp.Up`, UAPI rendering, stats). `pharos-awg` is the
unit we package and that both modes build on.

---

## 3. Client mode — OPNsense as a caravel client

The firewall imports a `.pharos` profile and routes the LAN (or selected
sources) through the AmneziaWG tunnel. For a multi-hop profile the box dials only
the **entry** node; the controller routes the rest server-side (the profile's
`Path` view, `caravel/go/profile/pharos.go`).

### 3.1 Tunnel bring-up (data plane)

Reuse caravel's engine end to end. The flow mirrors `caravel-mac`'s `connect()` /
`configureNetwork()` (`caravel-mac/cmd/caravel-mac/main.go`), which is already
written against BSD `ifconfig`/`route` — macOS and FreeBSD share that surface, so
the port is nearly mechanical:

1. `dev := tun.CreateTUN("awg", mtu)` → kernel hands back `awg0` (or next free).
2. `vp.Up(cfg, dev, …)` brings the AmneziaWG device up against the resolved
   profile tunnel (`profile.Node.Tunnel()` → endpoint chosen at random from the
   pool per decision 17).
3. Address the interface: `ifconfig awg0 inet <addr> <addr> up` (point-to-point,
   exactly as the macOS shell does).
4. Routing for a full tunnel:
   - **Pin the endpoint**: `route add -host <server-ip> <physical-gateway>` so the
     encrypted WG packets to the node don't loop back into the tunnel.
   - **Override default**: add `0.0.0.0/1` and `128.0.0.0/1` via `awg0` (the
     classic split that beats the real default route without deleting it). The
     macOS shell does precisely this.
   - On teardown, delete each route we added (tracked in an undo list) and close
     the device.
5. Split tunnel: instead of the `/1` halves, add a route per `AllowedIPs` CIDR via
   `awg0`.

> FreeBSD note: the macOS code uses `route -n get default` to find the gateway —
> identical on FreeBSD. The point-to-point `ifconfig <if> inet A A up` form also
> works on FreeBSD tun. The one thing to verify on the VM is interface naming
> (`awg0` vs the cloned `tun0` name) and that `route -n get default` returns the
> WAN gateway in this single-NIC setup (§9).

### 3.2 LAN routing & firewall (the OPNsense-specific part)

Bringing the tunnel up is not enough — OPNsense's `pf` won't pass or NAT LAN
traffic into a foreign interface unless we register it. Model it after how the
core WireGuard plugin exposes its interfaces:

- Register `awg0` as an **assignable OPNsense interface** so it shows up under
  Interfaces and gets a `pf` identity, and add it to a virtual interface **group**
  `pharosvpn` (mirrors `wireguard_interfaces()` returning a `type: group`). Firewall
  rules then target the group, surviving interface renumbering.
- Create a **gateway** on `awg0` (the tunnel's far address / a sentinel) so
  policy-based routing and the dashboard health check work.
- **Outbound NAT**: add a hybrid/automatic outbound-NAT rule that masquerades
  `LAN net` out `awg0` (translation to the interface address). Without this, return
  traffic from the node has nowhere to come back to.
- **Policy routing (recommended default)**: a floating/LAN firewall rule
  `pass in on LAN from <sources> to any  → gateway PharosVPN_GW`. This routes the
  selected LAN sources through the tunnel while leaving the firewall's own control
  traffic alone — important, because **coxswain dials out and clients reach the
  fleet through relays**; we must not black-hole the box's own management path.
- **Kill-switch (optional)**: a block rule on WAN for the policy-routed sources so
  that if the tunnel drops, LAN traffic fails closed instead of leaking to the ISP.
- **DNS**: point the routed clients at a DNS server reachable through the tunnel
  (or Unbound bound to the tunnel) to avoid DNS leaks.

### 3.3 GUI / UX (the `os-pharosvpn` plugin, client side)

A new MVC module `OPNsense/PharosVPN` with a **VPN → PharosVPN** menu:

- **Profiles page**: "Import `.pharos`" (file upload; the parser is caravel's
  `profile.Parse`, exposed by the daemon — see §6). For password/account-mode
  profiles, prompt for the password (`EncPassword`) or device key bundle
  (`EncAccount`). Show resolved nodes / the egress path, expiry, fleet id.
- **Connection page**: pick a profile, full-vs-split tunnel toggle, the LAN source
  set to route, kill-switch toggle, connect/disconnect, live status (handshake age,
  RX/TX from the UAPI socket, the endpoint actually dialed).
- **Model** (`PharosVPN/Client.xml`): stores imported profile blobs (encrypted at
  rest, see §6), selected node/path, routing mode, source selectors. Field types
  reused from core (`Base64Field`, `NetworkField`, `BooleanField`).
- On save, the controller writes the daemon config and runs
  `configctl pharosvpn restart <id>`; the configd action invokes a
  `pharos-service-control.php` that starts/reloads `pharos-awg` and re-applies the
  interface/route/NAT setup (the analog of `wg-service-control.php`).
- A **diagnostics** page (like WireGuard's `wg_show.py`) that dumps the UAPI socket:
  peers, last handshake, throughput.

### 3.4 Persistence across reboots

OPNsense state lives in `config.xml`, not in hand-rolled rc scripts. The plugin
persists everything in its model; on boot the standard reconfigure hooks
(`pharosvpn_configure()` → on the `vpn` event) recreate the tun, restart
`pharos-awg`, and re-apply addresses/routes — exactly as `wireguard_configure_do`
does for WireGuard. The interface group + NAT + rules persist as normal `pf`
config. No `/etc/rc.conf` editing; configd + the `.inc` hooks own the lifecycle.

---

## 4. Server mode — OPNsense running a PharosVPN server component

Three sub-cases, in increasing order of awkwardness on a firewall.

### 4.1 `node` (a public VPN gateway) — the interesting one

`node` runs the client-facing AmneziaWG interface (udp/443) and cascades to other
nodes. Two FreeBSD gaps from the current Linux implementation
(`node/internal/awg/runtime.go`, `netpolicy/`):

1. **The data-plane runtime.** Today `ExecRuntime` shells out to `awg`/`awg-quick`
   and `ip route` (Linux). On FreeBSD there is no `awg-quick` and no `ip`. Options:
   - **(a) FreeBSD `Runtime` over `pharos-awg`** *(preferred)*: implement the
     `Runtime` interface (`Up/Down/SyncConf/AddPeer/RemovePeer/AddRoute/RemoveRoute/Show/Listening`)
     against the userspace device's UAPI socket + BSD `route`/`ifconfig`. This drops
     the bash `awg-quick` dependency and the Linux-only `ip route` calls. Peer
     add/remove map to UAPI `set` operations; `Show` reads the UAPI dump (the data
     `vp.Tunnel.Stats()` already parses); routes use `route add … -interface awgN`.
   - **(b) Port the AmneziaWG userspace tools to FreeBSD** and keep `ExecRuntime`:
     build `awg`/`awg-quick` for FreeBSD (awg-quick is bash, would need a BSD
     rewrite). More moving parts, less clean. Use only if (a) reveals a UAPI gap.
2. **The network policy (`netpolicy`).** `applier.go`/`netpolicy.go` install
   masquerade + isolation + cascade transit using **Linux fwmark + `ip rule` +
   `iptables`/`nftables`**. None of that exists on FreeBSD. A FreeBSD node needs the
   equivalent expressed in **`pf`** (NAT/`rdr`, `route-to` for the asymmetric
   cascade hops) and FreeBSD routing tables (`setfib` instead of `ip rule`
   tables). The cascade's asymmetric-routing problem (the `rp_filter=0` fix from
   the live tests) reappears as **`pf` antispoof / `net.inet.ip.fastforwarding`**
   tuning — different mechanism, same hazard. **This is the largest single piece
   of new work in the whole integration** and the main reason server-mode `node`
   on OPNsense is "advanced/experimental," not the headline feature.

Beyond those: `node` is enrolled by coxswain over SSH and driven over gRPC
(`node/BUILD.md`); none of that is Linux-specific. It cross-compiles to FreeBSD
(pure Go + gRPC). IP forwarding is `sysctl net.inet.ip.forwarding=1` (vs Linux
`ip_forward`). Packaged as a configd-managed rc.d service via the plugin (§5).

### 4.2 `relay` (public ingress, reverse-tunnels to coxswain)

Easiest server component to host. It's a Go binary that dials *out* to coxswain and
reverse-tunnels the client account-sync gRPC; it terminates AmneziaWG **only if it
also acts as an entry node** — a pure relay does not touch the data plane, so no
FreeBSD `pf`/AmneziaWG work is needed. Cross-compile, ship as an rc.d service +
config, expose a small GUI page for the coxswain dial target and enrollment
material. Recommended as the first server-mode target to ship.

### 4.3 `coxswain` (the private controller)

Technically runnable on OPNsense (pure Go + modernc SQLite, no cgo — confirmed by
`helm/go.mod`, which uses `modernc.org/sqlite`), but **architecturally a poor fit**:
coxswain holds the CA + fleet state and is meant to be *private, behind NAT, zero
inbound* (`helm/README.md`). A firewall is internet-edge and high-value; co-locating
the root of trust there contradicts the threat model. Support it as an *unsupported*
"all-in-one homelab" option (binary + rc.d + a thin status page), but steer users
to run coxswain on a separate private host. Not a priority.

---

## 5. Packaging — `os-pharosvpn` plugin + FreeBSD `pkg`

Layered, matching the WireGuard precedent:

1. **The engine/binaries**: cross-compile on the Mac —
   `GOOS=freebsd GOARCH=amd64 CGO_ENABLED=0 go build` — producing:
   - `pharos-awg` (the userspace AmneziaWG daemon, from caravel's `vp`),
   - `node`, `relay`, and optionally `cox`/coxswain.
   Wrap them in a base FreeBSD package `pharosvpn` (or per-component packages) with
   an rc.d script per long-lived service and a manifest declaring `if_tuntap` as a
   runtime expectation. Static (`CGO_ENABLED=0`) avoids FreeBSD libc ABI surprises.
2. **The GUI plugin** `os-pharosvpn`: the standard OPNsense plugin layout
   (the `helloworld`/`wireguard` template) — `src/opnsense/mvc/app/{models,controllers,views}/OPNsense/PharosVPN/`,
   `src/opnsense/service/conf/actions.d/actions_pharosvpn.conf`,
   `src/opnsense/service/templates/OPNsense/PharosVPN/`,
   `src/etc/inc/plugins.inc.d/pharosvpn.inc` (the `_services`/`_interfaces`/`_devices`/`_configure`
   hooks), and a `+MANIFEST`/`pkg-descr`. The plugin **depends on** the `pharosvpn`
   binary package.
3. **Distribution**: a self-hosted OPNsense plugin repo (add a `repos/*.conf`
   pointing at our pkg server) or, for early adopters, manual `pkg add` of the two
   packages. Long-term, submission to `opnsense/plugins` is possible but the
   userspace-AmneziaWG dependency + bundled Go binaries make a contrib-tier listing
   more realistic than core.

rc.d sketch for the daemon (configd starts/stops it; not enabled directly in
`rc.conf`, mirroring how WireGuard is configd-managed):

```sh
#!/bin/sh
# PROVIDE: pharos_awg
# REQUIRE: NETWORKING
. /etc/rc.subr
name=pharos_awg; rcvar=pharos_awg_enable
command=/usr/local/sbin/pharos-awg
command_args="--config /usr/local/etc/pharosvpn/awg0.conf"
load_rc_config $name
run_rc_command "$1"
```

---

## 6. Config & secrets handling

- **`.pharos` profile blobs** are imported through the GUI and stored in the
  plugin model (i.e. in `config.xml`). Account/password-mode payloads are already
  encrypted in the envelope (`EncAccount`/`EncPassword`,
  `caravel/go/profile/pharos.go`), so those stay sealed at rest. For `EncNone`
  profiles, the resolved private key is sensitive — keep the rendered daemon conf
  (the AmneziaWG private key + PSK) out of `config.xml`, written to
  `/usr/local/etc/pharosvpn/awgN.conf` at **0600** (the same posture
  `node/internal/awg/manager.go` uses: `confFileMode = 0o600`, atomic temp+rename).
- **Profile parsing/decryption runs in the privileged daemon, not PHP.** The GUI
  hands the blob (and password / device-key bundle) to `pharos-awg` over a local
  control socket; the daemon calls caravel's `profile.Parse`. This keeps the crypto
  in the audited Go implementation and avoids reimplementing Argon2id/XChaCha/sealed-box in PHP.
- **Secrets never on argv or in env or logs** — the existing runtime already
  enforces this (PSK piped on stdin, `redactOutput`/`redactPubkey` in
  `runtime.go`); the FreeBSD runtime must keep the same discipline (UAPI over a
  0600 unix socket, not command-line key material).
- **Node/relay enrollment material** (mTLS keys, the coxswain dial info) lives in
  the component's own config dir at 0600, written by the plugin from the model,
  same as a normal `node` deploy.

---

## 7. Security posture

- **Userspace AmneziaWG on an edge device**: the data plane runs as a daemon. Run
  it unprivileged where possible (it needs the tun fd + route changes — on FreeBSD
  that's root or `CAP`-style privilege; OPNsense daemons typically run as root under
  configd, which is the accepted norm here).
- **Client mode preserves the controller-hiding model**: the firewall only ever
  *dials out* to a node entry endpoint; it never learns coxswain's location
  (coxswain is never in the data path). The plugin must **not** route the box's own
  management/coxswain traffic into the tunnel — policy-route only the chosen LAN
  sources (§3.2), or a multi-hop profile's own control path could loop.
- **Server-mode `node` widens the attack surface** of an internet-edge box (it now
  terminates public udp/443 and forwards). Keep it opt-in and clearly labelled; the
  `pf` policy must enforce client isolation and the cascade transit rules correctly
  (the §4.1 work), or a misconfiguration leaks one client's traffic to another.
- **`coxswain` on the firewall is discouraged** — it co-locates the CA/root-of-trust
  with the most-exposed host, against the project's unlinkability/controller-hiding
  posture (see the project memory on unlinkability). Support, don't recommend.
- **Profiles at rest**: prefer account/password-mode profiles so the sealed payload
  protects the key even if `config.xml` leaks (config backups are a common exfil
  path on firewalls).
- **No new inbound for client mode**: zero listening ports added; the box only makes
  outbound udp to the node. Good fit for the project's egress posture.

---

## 8. Effort estimate (per mode)

Rough, assuming the caravel core and `node` are reused (not rewritten). "wk" =
focused engineering-weeks.

| Work item | Effort | Risk |
|---|---|---|
| `pharos-awg` daemon (wrap caravel `vp` + FreeBSD tun + UAPI socket + config loader) | 0.5–1 wk | Low — engine + FreeBSD tun both proven |
| FreeBSD cross-compile + base `pkg` (binaries, rc.d, manifest) | 0.5 wk | Low |
| **Client mode** plugin: MVC model + GUI (import/connect/status), configd actions, `.inc` hooks, interface/group/gateway registration, outbound NAT + policy-route + kill-switch, diagnostics | 3–4 wk | Medium — `pf`/interface plumbing + GUI are the bulk |
| Client-mode end-to-end on the VM (route LAN out a real fleet node, leak tests, reboot persistence) | 1 wk | Medium — DNS/route-leak edge cases |
| **Server mode — `relay`** (cross-compile, rc.d, small GUI page, enrollment) | 1–1.5 wk | Low — no data-plane/`pf` work |
| **Server mode — `node`**: FreeBSD `Runtime` (UAPI-backed), **`netpolicy` rewrite in `pf`/`setfib`** (NAT, isolation, asymmetric cascade route-to), forwarding sysctl, GUI | 4–6 wk | **High** — the `pf` cascade/policy port is genuinely hard; the asymmetric-routing hazard from live tests reappears in a new mechanism |
| **Server mode — `coxswain`** (binary + rc.d + thin status page; unsupported tier) | 0.5–1 wk | Low effort / High advisability risk |

Suggested order: **`pharos-awg` + pkg → client mode (the headline) → `relay` →
(experimental) `node` → (unsupported) `coxswain`**.

Client mode is the high-value, lower-risk deliverable and should ship first.

---

## 9. Open questions / what still needs proving

1. **Userspace throughput.** Benchmark `amneziawg-go` on the VM (and on a realistic
   appliance) at udp/443 full-tunnel. Userspace WG on a low-power firewall can cap
   well below line rate. Quantify before promising whole-network VPN performance.
2. **Tun naming & the OPNsense interface assignment.** `tun.CreateTUN("awg", …)`
   cloning behaviour on FreeBSD — does it yield `awg0` or a generic `tunN`? OPNsense
   interface assignment and the `^wg`-style device pattern need a stable, predictable
   name. Verify on the VM (the VM currently shows only `vtnet0 lo0 enc0 pfsync0
   pflog0` — no tun yet, since none has been created).
3. **`route -n get default` in this topology.** The dev VM is single-NIC LAN-DHCP
   (per `docs/DEV-VMS.md`); confirm the endpoint-pinning route logic finds the real
   upstream gateway on a normal WAN-having firewall.
4. **The `pf` cascade port (server `node`).** Prove that OPNsense `pf` `route-to` +
   `setfib` can reproduce the Linux fwmark/`ip rule` per-device cascade transit, and
   that the asymmetric cross-adapter traffic isn't dropped by `pf` antispoof (the
   FreeBSD analog of the `rp_filter=0` fix). This is the riskiest unknown.
5. **UAPI completeness for the server `Runtime`.** Confirm the `amneziawg-go` UAPI
   socket exposes everything `node`'s `Runtime` needs (per-peer dump fields,
   live peer add/remove, listen-port/obfuscation set) so we can retire `awg`/`awg-quick`.
6. **Plugin packaging/signing** for a self-hosted OPNsense repo (fingerprints, ABI
   pinning to `FreeBSD:14:amd64` / `25.7`) and the upgrade story when the ABI bumps
   to `26.1` (`CORE_NEXT` in the version blob).
7. **CARP/HA**: how the tunnel + daemon behave under an OPNsense CARP failover (the
   core WireGuard model has explicit `carp_depend_on` — we likely need the same).
8. **Profile decryption boundary**: confirm the GUI→daemon control-socket handoff
   is the right trust split, and that account-mode device keys can be supplied on a
   headless firewall (no interactive device-enrollment UX yet).

---

### Appendix — key source references

- Data-plane engine (reused as-is for client mode): `caravel/go/vp/engine.go`
- `.pharos` profile parse/resolve (multi-hop entry selection, endpoint pool): `caravel/go/profile/pharos.go`
- BSD tun + routing reference shell (port target for FreeBSD): `caravel-mac/cmd/caravel-mac/main.go` (`connect`, `configureNetwork`), `daemon.go`
- Server-node data-plane runtime to port to FreeBSD: `node/internal/awg/runtime.go`, `node/internal/awg/manager.go`, `node/internal/awg/conf.go`
- Linux network policy needing a `pf` rewrite: `node/internal/netpolicy/`
- Linux node base prep (FreeBSD equivalent needed): `node/deploy/cloud-init.sh`
- OPNsense plugin template (read live on the VM): `/usr/local/opnsense/mvc/app/{models,controllers,views}/OPNsense/Wireguard`, `/usr/local/opnsense/service/conf/actions.d/actions_wireguard.conf`, `/usr/local/opnsense/scripts/Wireguard/wg-service-control.php`, `/usr/local/etc/inc/plugins.inc.d/wireguard.inc`
- amneziawg-go FreeBSD tun backend (the feasibility linchpin): `…/amnezia-vpn/amneziawg-go@v0.2.18/tun/tun_freebsd.go`
- Dev VM bootstrap + gotchas (tcsh, configd-managed services): `docs/DEV-VMS.md`
