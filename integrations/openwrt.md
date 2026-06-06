<!--
SPDX-License-Identifier: Apache-2.0
Copyright (C) 2026 The PharosVPN Authors
-->

# PharosVPN on OpenWrt — Integration Design

**Status:** design / feasibility — not an implementation plan.
**Grounded in:** the live dev VM `root@192.168.0.217` (OpenWrt **24.10.0**,
`r28427-6df0e3d02a`, kernel **6.6.73**, target **x86/64**, `DISTRIB_ARCH=x86_64`),
inspected read-only on 2026-06-06.
**Reads against:** `docs/DESIGN.md` (decisions 14–18, §3), the `caravel` core
(`caravel/go/vp/engine.go`, `caravel/go/profile/pharos.go`), `caravel-mac`
(the reference platform shell), and the `node` agent (`internal/awg`,
`internal/netpolicy`).

---

## 0. TL;DR

| | Verdict |
|---|---|
| **Client mode** (router = caravel, whole-LAN VPN) | **Feasible now**, no new kernel module. Ship caravel-core as a **userspace AmneziaWG** Go binary over `kmod-tun`. This is the *same* engine and tun model caravel already uses on macOS/mobile. Recommended first target. |
| **Server mode — node** | **Feasible but heavier.** OpenWrt has **no AmneziaWG kernel module** in its feeds, and `node` today shells out to `awg`/`awg-quick` (kernel path) + **iptables**. Two ports needed: (a) run the data plane in userspace (amneziawg-go, no `awg-quick`), and (b) re-render netpolicy for **fw4/nftables**. |
| **Server mode — relay / coxswain** | **Feasible, low-risk.** Pure Go, no data plane, no kernel module. Mostly a packaging exercise (opkg + procd init). coxswain on a router is unusual (it holds the CA + SQLite) but nothing blocks it. |

The crux question — *"is there a kmod-amneziawg, and if not can we run amneziawg-go
userspace?"* — resolves to: **no kernel module in the stock feeds, but yes,
amneziawg-go userspace is fully feasible** and is already how every caravel
client works. That makes userspace the unifying data-plane strategy for OpenWrt.

---

## 1. What the dev VM actually has (verified)

### 1.1 Platform

```
DISTRIB_ID='OpenWrt'   DISTRIB_RELEASE='24.10.0'   DISTRIB_TARGET='x86/64'
DISTRIB_ARCH='x86_64'  Linux openwrt-dev 6.6.73 x86_64
```

- **CPU:** `Common KVM processor` (this is a VM; real routers are usually
  arm/aarch64/mips/mipsel — see §7 on cross-arch).
- **RAM:** 493 MB total, ~424 MB available. Comfortable for a Go binary.
- **Disk:** `/dev/root` **98.3 MB, 71.1 MB free** (26% used). `/tmp` is a 241 MB
  tmpfs. **This is the binding constraint** — on a 16/32 MB flash router there is
  almost no room (see §7). On this x86 image there is no separate `/overlay`
  partition; the rootfs is the writable store.
- **libc is musl:** `/lib/ld-musl-x86_64.so.1 -> libc.so` (513 KB). Any native
  C component must be musl-linked; a pure-Go static binary sidesteps this
  entirely (`CGO_ENABLED=0`).

### 1.2 Package feeds (stock, signed)

`/etc/opkg/distfeeds.conf` points only at the official 24.10.0 feeds:
`openwrt_core`, `_base`, `_kmods`, `_luci`, `_packages`, `_routing`,
`_telephony`. `option check_signature` is on and every `opkg update` line
reported `Signature check passed`. No custom feed is configured.

**AmneziaWG: absent.** `opkg find "*amnezia*"` and `"*awg*"` returned nothing;
grepping the decompressed package indices for `amnezia` found nothing.
There is **no `kmod-amneziawg` and no `amneziawg-tools`** in any stock feed.

**Stock WireGuard: present.** `kmod-wireguard 6.6.73-r1`, `wireguard-tools
1.0.20210914-r4` (the `wg`/`wg-quick` userspace + a netifd protocol helper),
`luci-proto-wireguard` (depends on `wireguard-tools`). Stock `wg` cannot speak
AmneziaWG's Jc/Jmin/Jmax/S1-4/H1-4 obfuscation — it has no knobs for those keys —
so it is **not** a substitute.

**Go toolchain: present** as `golang 1.23.12-r2` (+ `golang-src` for
cross-compilation). We will not build *on* the router; this only confirms the
ecosystem packages Go normally.

**TUN: available, not loaded.** `/dev/net/tun` does not exist yet, but
`kmod-tun 6.6.73-r1` ("Kernel support for the TUN/TAP tunneling device") is in
the feed. `kmod-tun` is the foundation of userspace AmneziaWG.

### 1.3 Network / firewall model (the live config)

`/etc/config/network` — a `br-lan` bridge over `eth0`, `lan` as `proto dhcp`
(this VM is a LAN client, not a edge router; a real router would have a `wan`).
`/etc/config/firewall` — standard `lan`/`wan` zones, `wan` has `option masq 1`
and `option mtu_fix 1`, a `lan→wan` forwarding.

- **Firewall backend is fw4 / nftables**, not iptables. `/sbin/fw4` exists
  (`fw3 → fw4` symlink), `/usr/sbin/nft` is present, `nft list tables` shows
  `table inet fw4`. **`iptables` is not installed.** This directly conflicts
  with `node`'s netpolicy, which emits `iptables`/`iptables -t nat`/`-t mangle`
  commands (`node/internal/netpolicy/netpolicy.go`). See §5.3.
- **`ip` is BusyBox** by default (`ip rule`/`ip route` subset only). Full
  iproute2 is available as `ip-full 6.11.0-r1` (228 KB, deps `libnl-tiny1
  libbpf1 libmnl0`) — needed for the cascade's `ip rule`/`ip route ... table`
  policy routing.
- NAT kernel bits present: `kmod-nf-conntrack`, `kmod-nf-nat`, `kmod-nft-nat`.
  `firewall4` + `nftables-json` installed.

### 1.4 LuCI (the web UI)

LuCI is the **modern client-side** generation: `luci-base 25.x`, `luci-light`,
`luci-mod-network/-status/-system`, `rpcd-mod-luci`, `ucode` + `rpcd-mod-ucode`.
The app model is:

- **Client JS views** under `/www/luci-static/resources/view/<app>/*.js`
  (CBI-style `form`/`view` classes, fetched by the browser).
- **Menu** entries in `/usr/share/luci/menu.d/luci-app-*.json`.
- **ACLs** in `/usr/share/rpcd/acl.d/luci-app-*.json` (what ubus/uci paths the
  view may read/write).
- **Backend** = `rpcd` ucode/ubus methods (or shelling to our CLI via a small
  ubus object); state lives in **uci** (`/etc/config/<app>`).

There is also `luci-app-package-manager` installed — opkg has a GUI front-end,
so a published `.ipk` is installable from the browser.

---

## 2. AmneziaWG data-plane strategy (the central decision)

AmneziaWG is the data plane for every PharosVPN component (DESIGN §3). The wire
protocol is WireGuard with obfuscation params (`Jc/Jmin/Jmax`, `S1–S4`,
`H1–H4`, `I1–I5`). There are two ways to run it; only one exists on OpenWrt
today.

### 2.1 Kernel module (`kmod-amneziawg` + `amneziawg-tools`) — NOT available

This is the Linux server path. On the `node`'s Ubuntu droplets we install it from
`ppa:amnezia/ppa` (`node/deploy/cloud-init.sh`: `apt-get install amneziawg
amneziawg-tools` + `modprobe amneziawg`), then drive it with `awg` / `awg-quick`
(`node/internal/awg/runtime.go`). It is fast (in-kernel crypto) and gives us
`awg show … dump` for live per-peer stats.

**On OpenWrt 24.10 there is no such package.** Building one would mean either:
(a) an out-of-tree **DKMS-style kernel module** built against the exact OpenWrt
kernel (`6.6.73-1-a21259e4f338051d27a6443a3a7f7f1f` is even in the kmods feed
path — kmods are pinned to the kernel ABI hash), shipped as a `kmod-amneziawg`
`.ipk` per target/kernel; or (b) waiting for/contributing such a package
upstream. This is a real, ongoing maintenance burden: a kmod must be rebuilt for
**every target arch × every kernel bump**, and OpenWrt's kmod feed is regenerated
per release. We should **not** make the integration depend on it.

### 2.2 Userspace `amneziawg-go` over `kmod-tun` — RECOMMENDED

`amnezia-vpn/amneziawg-go` is the userspace AmneziaWG implementation in pure Go.
caravel **already uses it** for every client: `caravel/go/vp/engine.go` builds a
`device.NewDevice(tunDev, conn.NewDefaultBind(), …)`, feeds it a UAPI string
(private key + `jc/jmin/jmax/s1-s4/h1-h4/i1-i5` + one peer), and the platform
shell supplies the tun device. On macOS that tun is a `utun` from
`tun.CreateTUN("utun", mtu)` (`caravel-mac/.../main.go`); on Linux/OpenWrt
`tun.CreateTUN("awg0", mtu)` creates a `/dev/net/tun` interface of our choosing.

**Why this is the right call for OpenWrt:**

- **Zero new kernel module.** Only `kmod-tun` (already in-feed), `CGO_ENABLED=0`,
  one static Go binary. Portable across every OpenWrt target arch by
  recompiling `GOARCH`/`GOMIPS` — no per-kernel rebuild.
- **One engine, all platforms.** It is byte-for-byte the same obfuscation
  handling caravel ships on macOS/iOS/Android. The obfuscation set the node
  advertises (`node/internal/awg`, surfaced in profiles via
  `profile.Obfuscation`) is consumed identically.
- **Reuse.** caravel's `vp.Up` / `profile.Parse` / `profile.Tunnel` are already
  the contract. An OpenWrt client is a *new platform shell* around the existing
  core — not new crypto.

**Costs / what to prove:**

- **Throughput.** Userspace WireGuard does crypto + a kernel↔userspace copy per
  packet; on a weak router CPU it caps lower than in-kernel. On the x86 KVM VM
  this is a non-issue; on an ARM/MIPS home router it must be benchmarked
  (see §8). Mitigations: GSO/UDP-batching (amneziawg-go has it on Linux), MTU
  tuning. This is the single biggest open question for both modes on real
  hardware.
- **Stats.** No `awg show dump`. Per-peer stats come from the device UAPI
  (`IpcGet`, already used by `vp.Tunnel.Stats` for rx/tx). For *server mode*
  (many peers) we read/parse the full UAPI peer dump instead of `awg show`.

**Decision:** the OpenWrt data plane is **userspace amneziawg-go over kmod-tun**
for *every* mode (client, node, relay, coxswain). The kernel-module path stays a
future optimization, gated on someone maintaining a `kmod-amneziawg` feed.

---

## 3. Client mode — router as a caravel client (whole-LAN VPN)

The router imports a `.pharos` profile, brings up an AmneziaWG tunnel, and routes
the **whole LAN** out through it. For a multi-hop profile the router dials only
the **entry node** (`profile.Profile.EntryNodeID`); the controller routes the
rest server-side — the router never sees the cascade (DESIGN decision 18).

### 3.1 Component: `caravel-owrt`

A new platform shell, mirroring `caravel-mac` but emitting OpenWrt-native network
ops. Reuses, unchanged:

- `profile.Parse` / `profile.Store` — import + decrypt `.pharos`
  (`none`/`password`/`account` modes already implemented).
- `profile.Node.Tunnel()` — resolves endpoint pool → dialable `Tunnel`
  (random IP+port per connect, decision 17).
- `vp.Up(cfg, tunDev, logLevel)` — the AmneziaWG userspace engine.
- `sync.Client` — `.pharosid` account-sync fetch (account-mode profiles).

What's new (the platform-specific part `caravel-mac` does with `ifconfig`/`route`):

1. **Create the tun:** `tun.CreateTUN("awg0", mtu)` (Linux path of amneziawg-go).
2. **Address it:** `ip addr add <Tunnel.Address>/32 dev awg0; ip link set awg0
   up` (full `ip` from `ip-full`, or via netifd — see §3.2).
3. **Pin the endpoint:** add a host route to the dialed server IP via the real
   WAN gateway, so the encrypted UDP to the node does not recurse into the
   tunnel. caravel-mac does exactly this (`route add -host <ip> <gw>`); the
   OpenWrt form is `ip route add <serverIP>/32 via <wan-gw> dev <wan>`.
   (Endpoint is random per connect from the pool — re-pin on rotation.)
4. **Capture the default route:** the split-default trick caravel-mac uses
   (`0.0.0.0/1` + `128.0.0.0/1` via the tun) works identically and avoids
   clobbering the original default. Alternatively a dedicated routing table +
   `ip rule`.
5. **NAT the LAN into the tunnel:** masquerade LAN-sourced traffic out `awg0`
   so return packets come back to the router. On fw4 this is one nft rule
   (see §3.3) — *not* the iptables form node uses.

### 3.2 Two integration depths

**(a) Standalone CLI + procd service (faster to ship).** `caravel-owrt` is a
self-contained binary doing steps 1–5 itself. A procd init script
(`/etc/init.d/pharosvpn`) runs `caravel-owrt connect --profile <name>`; uci
`/etc/config/pharosvpn` holds the selected profile name + full/split-tunnel
toggle. This is the closest analog to `caravel-mac connect` and the lowest-risk
first deliverable.

**(b) A netifd `proto` integration (the "OpenWrt-native" path).** OpenWrt drives
interfaces through **netifd protocol helpers** in `/lib/netifd/proto/` (verified:
`dhcp.sh`, `dhcpv6.sh`, `ppp.sh`; `wireguard-tools` adds `wireguard.sh`, and
`luci-proto-wireguard` is the LuCI face of it). The native pattern is a
`/lib/netifd/proto/amneziawg.sh` that, on `proto_amneziawg_setup`, starts our
userspace device and registers the interface so it appears as a first-class
`config interface` with `proto 'amneziawg'`. This is more work but makes the VPN
a normal OpenWrt interface (assignable to a firewall zone, visible in
Status→Interfaces, brought up at boot by netifd). **Recommendation:** ship (a)
first, evolve to (b) once the engine is proven on-device.

Either way, persistence + boot is **procd** (the OpenWrt service manager) — an
`/etc/init.d/` script with `USE_PROCD=1`, `start_service`/`stop_service`,
`procd_set_param respawn`, and `procd_add_reload_trigger "pharosvpn"` so a uci
change reloads it. (Verified: the VM's `/etc/init.d/` already has `network`,
`firewall`, `dropbear`, etc. — same model.)

### 3.3 Firewall / NAT on fw4 (LAN → tunnel)

The clean OpenWrt way is **declarative uci**, not hand-rolled nft, so fw4
regenerates correctly on reload:

```
# /etc/config/network  — the VPN interface (proto static if option (a))
config interface 'pharos'
    option proto 'none'        # device managed by our service / proto helper
    option device 'awg0'

# /etc/config/firewall
config zone
    option name 'pharos'
    list network 'pharos'
    option masq '1'            # NAT LAN traffic into the tunnel
    option mtu_fix '1'         # clamp MSS — important for a ~1420 MTU tunnel
    option input 'REJECT'
    option output 'ACCEPT'
    option forward 'REJECT'

config forwarding
    option src 'lan'
    option dest 'pharos'       # route the LAN out through the VPN
```

This reuses the *exact* knobs the stock `wan` zone already uses (`masq 1`,
`mtu_fix 1` — verified in the VM's firewall config). **Kill-switch** (don't leak
to the real WAN if the tunnel drops): drop or remove the `lan→wan` forwarding
while connected, and add a fw4 rule rejecting `lan→wan` so traffic only egresses
`pharos`. **DNS:** point LAN clients at a resolver reachable inside the tunnel
(or run dnsmasq forwarding over `awg0`) to avoid DNS leaks.

### 3.4 Client-mode LuCI UX

`luci-app-pharos-client`:

- **Profiles** view: list imported profiles, **upload a `.pharos`** (browser →
  rpcd → our CLI `import`), set/clear password, run `sync` for account-mode.
- **Connection** view: pick profile, choose node/exit (if the profile carries a
  `Path`/multiple nodes), full- vs split-tunnel toggle, **Connect/Disconnect**,
  live status (up/down, endpoint, rx/tx from `vp.Tunnel.Stats`).
- A small **status indicator** on the LuCI overview.

Backend: a thin ubus/rpcd object (ucode) that calls our CLI and reads
`/etc/config/pharosvpn`. Profile blobs (which may be ciphertext) live under
`/etc/pharos/profiles/` (`0600`), referenced by name from uci — *not* inlined in
uci, which is world-readable.

---

## 4. Server mode — running a PharosVPN server component on the router

A home/office box acts as a fleet **node**, a **relay**, or even the
**coxswain** controller. All three are Go binaries; the differentiator is the
data plane.

### 4.1 relay and coxswain — straightforward

Neither runs an AmneziaWG data plane. They are pure Go (gRPC/mTLS + SQLite for
coxswain). Porting is **packaging only**:

- Cross-compile static (`CGO_ENABLED=0`) for the target `GOARCH`. **Verified:**
  coxswain already uses the **pure-Go** SQLite driver `modernc.org/sqlite
  v1.50.1` (`helm/go.mod`), so there is **no cgo blocker** — it cross-compiles
  to every OpenWrt target with `CGO_ENABLED=0` as-is.
- `relay` on-host layout is already specified (`helm/BUILD.md`):
  `/usr/local/bin/relay`, config `/etc/relay`, run as `relay run --config-dir
  /etc/relay`. On OpenWrt, swap the systemd unit for a procd init script;
  binary → `/usr/sbin/relay` (or keep `/usr/local/bin`), config dir unchanged.
- coxswain on a router is unusual — it holds the **CA private key + SQLite
  state** — and dials *out* with **zero inbound ports** (`helm/BUILD.md`
  non-negotiables). A router behind NAT is actually a *natural* home for
  "zero-inbound" coxswain, but flash wear (SQLite writes to a small overlay) and
  backup of the CA are real concerns. Treat coxswain-on-OpenWrt as a niche
  "personal controller appliance," not a headline use case.

### 4.2 node — the hard one

`node` is the only component with a real data plane, and today it assumes the
Ubuntu/kernel-module world. Two distinct ports are required.

**Port A — userspace data plane (replace `awg-quick`/`awg`).** `node`'s
`awg.Runtime` interface (`internal/awg/runtime.go`) is the *seam*: `Up`, `Down`,
`SyncConf`, `AddPeer`, `RemovePeer`, `AddRoute`, `RemoveRoute`, `Show`,
`Listening`. Today's `ExecRuntime` shells to `awg`/`awg-quick`. An
**`UserspaceRuntime`** would instead embed amneziawg-go:

- `Up` → `tun.CreateTUN("awg0", mtu)` + `device.NewDevice` + `IpcSet` (the
  rendered `[Interface]` private key, listen port, obfuscation).
- `AddPeer`/`RemovePeer`/`SyncConf` → `dev.IpcSet` UAPI deltas (peer pubkey,
  PSK, allowed-ips, optional endpoint for the inner-link cascade peer — the
  runtime already models a peer endpoint for node→node links).
- `Show` → parse `dev.IpcGet` (UAPI dump) into `LivePeer` instead of `awg show
  … dump`.
- `AddRoute`/`RemoveRoute` → `ip route` (same as today; needs `ip-full`).

This is a clean addition: the `Runtime` interface already isolates exactly this.
The conf-file source-of-truth model (`/etc/amnezia/amneziawg/awg0.conf`) becomes
internal state the userspace device is configured from.

**Port B — netpolicy on nftables/fw4 (replace iptables).** This is the sharper
mismatch. `node/internal/netpolicy/netpolicy.go` renders **iptables** for
forwarding/masquerade/isolation, and **`iptables -t mangle` + `ip rule` + `ip
route … table`** for the cascade *transit* rules (DESIGN decision 16/18). On the
VM **iptables is not installed; fw4/nftables is the backend.** Options:

1. **iptables-nft shim.** OpenWrt feeds carry `iptables` (legacy) and an
   `iptables-nft` compat layer; installing it lets the existing iptables strings
   run against the nft backend. Lowest-code-change, but it bolts a second
   firewall front-end onto fw4-managed nftables — fragile across fw4 reloads,
   and not idiomatic. Acceptable as a bring-up shortcut, not a shipping design.
2. **An nftables renderer for `netpolicy`.** Add an OpenWrt rule dialect:
   forwarding/masquerade/isolation as nft rules (or, better, as **uci firewall**
   sections fw4 owns), and the cascade `mangle MARK` as an nft `meta mark set` in
   a `mangle`/`prerouting` chain. The **fwmark policy routing itself** (`ip rule
   add fwmark … lookup …`, `ip route replace default dev <inner> table …`) is
   firewall-backend-agnostic and works as-is with `ip-full`. **This is the right
   long-term port** but it means the canonical rule set is no longer
   byte-identical between coxswain's preview and the OpenWrt node — DESIGN §3
   says coxswain previews and `node` applies the *identical* set. So this needs a
   contract decision (see Open Questions): either coxswain learns a per-node
   "firewall dialect," or OpenWrt nodes self-translate the canonical policy
   (the three bools + transit list arrive over `NetworkConfig`; the renderer is
   node-local already, so node-side translation is the smaller change).
3. **rp_filter still applies.** The cascade fix (`rp_filter=0` on `all` +
   `default`, proven live 2026-06) is a sysctl and works on OpenWrt unchanged —
   `node` even backstops it in `ExecRuntime.Up`.

**Reachability caveat.** `node` is the *public* side of the control channel —
coxswain dials **in** to `node`'s gRPC on **TCP 8444** (`node/BUILD.md`). A home
router behind CGNAT/NAT usually has no inbound reachability, so a router-as-node
either needs a public IP + a `wan`-zone firewall allow for 8444 (and the
AmneziaWG UDP listener), or it must be reframed to use the **relay** reverse-
tunnel model instead of direct inbound. For most home boxes, **relay** (which
dials out) is the natural server role; **node** suits a box with a real public
IP.

### 4.3 Server-mode uci / init.d / firewall

- **Config:** `/etc/config/pharos-node` (uci) for non-secret settings
  (interface name, listen port range, control port, config-dir); secrets
  (`node.key`, PSKs, `awg-node.json`, certs) stay as `0600` files under
  `/etc/node/` exactly as on Ubuntu — **never in uci**.
- **Service:** `/etc/init.d/pharos-node` (procd), `ExecStart` equivalent
  `node run --config-dir /etc/node`, `respawn`, reload trigger on the uci file.
- **Onboarding:** unchanged contract — `node gen-csr` on the box, coxswain signs
  over SSH and pushes back `node.crt`/`ca.crt` (decision 14). OpenWrt runs
  **dropbear** sshd by default (verified `/etc/init.d/dropbear`), so the
  SSH-driven enroll works; coxswain's deploy just needs to target opkg/procd
  paths instead of apt/systemd.
- **Firewall:** open the control port + AmneziaWG UDP only on the appropriate
  zone via uci `config rule` (the VM already shows the pattern). Forwarding +
  masquerade for the data plane go through the netpolicy port (§4.2 Port B).

---

## 5. Packaging — luci-app + opkg via the OpenWrt SDK

### 5.1 The SDK / Makefile model

OpenWrt packages are built with the **OpenWrt SDK** for a given target (here
`x86/64`, 24.10.0) — a cross-toolchain + buildroot subset. A package is a
directory with a `Makefile` using `include $(TOPDIR)/rules.mk` and
`include $(INCLUDE_DIR)/package.mk`, declaring `Package/<name>` (SECTION,
CATEGORY, TITLE, **DEPENDS**), `Package/<name>/install` (what lands in the
`.ipk` and where), and build rules. `make package/<name>/compile` emits an
`.ipk`; a feed (a git repo of such Makefiles, added to `feeds.conf`) is how
third parties distribute. The user installs via `opkg install` or the
`luci-app-package-manager` GUI (installed on the VM).

For a Go binary the package **does not compile Go inside the SDK** by default —
the simplest, most portable approach is to **cross-compile the Go binary
out-of-band** (`GOOS=linux GOARCH=<arch> CGO_ENABLED=0 go build`) and have the
Makefile just `$(INSTALL_BIN)` the prebuilt binary into `/usr/sbin`. (OpenWrt
*does* have golang build infra in `lang/go` for source builds, but a prebuilt
static binary keeps our cross-arch matrix in our own CI and avoids SDK Go
version drift.)

### 5.2 Packages we ship

| Package | Contents | Depends |
|---|---|---|
| `pharos-caravel` | client binary + `/etc/init.d/pharosvpn` + uci defaults | `kmod-tun`, `ip-full` |
| `luci-app-pharos-client` | LuCI JS views + menu.d + acl.d + rpcd glue | `pharos-caravel`, `rpcd-mod-ucode` |
| `pharos-node` | node binary + init + uci | `kmod-tun`, `ip-full` (+ `iptables-nft` *iff* shim path) |
| `pharos-relay` | relay binary + init | (none beyond libc) |
| `pharos-coxswain` | coxswain binary + init | (pure-Go sqlite) |
| `luci-app-pharos-node` | node admin/status LuCI (optional) | `pharos-node` |

LuCI app layout (matches the verified VM conventions): JS under
`/www/luci-static/resources/view/pharos/`, `menu.d/luci-app-pharos-client.json`,
`acl.d/luci-app-pharos-client.json`, optional `/usr/libexec/rpcd/pharos`
(ucode). `luci-app-*` are `Architecture: all` (verified: `luci-proto-wireguard`
is `all`) — the JS is arch-independent; only the Go binaries are per-arch.

### 5.3 Binary size

The Go binaries (amneziawg-go + gRPC + profile crypto) realistically land in the
**8–20 MB** range stripped. On this VM (71 MB free) that's fine; on a 16 MB-flash
router it is fatal — these packages target **routers with ≥128 MB storage**
(x86, larger ARM boxes, or USB/SD-extended flash). State this in the package
description. `upx` compression can roughly halve the on-flash size at a small
startup cost.

---

## 6. Config & secrets handling

- **uci holds only non-secrets** (interface names, ports, toggles, profile
  *names*). uci files are world-readable; never put keys there.
- **Secrets are `0600` files**, matching the existing component contracts:
  client profiles `/etc/pharos/profiles/*.pharos` (may be ciphertext anyway —
  `password`/`account` modes); node `/etc/node/{node.key,awg-node.json,*.crt}`;
  relay `/etc/relay/*`. The node/relay private keys are generated **on the box**
  by `gen-csr` and never leave it (decision 14) — unchanged on OpenWrt.
- **`.pharos` decryption secrets** (password / device key) are never persisted
  by us; password is prompted/piped (caravel-mac uses `--password-stdin` so it
  never hits the process table — keep that on OpenWrt), account-mode uses the
  device key from the `.pharosid` flow.
- **Flash wear:** coxswain's SQLite and any frequent state writes should target a
  location that isn't a tiny flash overlay (USB/SD, or accept it only on x86).

---

## 7. Cross-arch reality

The dev VM is **x86_64**, the easy case. Real consumer routers are predominantly
**aarch64**, **arm**, **mipsel**/**mips** (the latter two often 32-bit,
big/little-endian, with `GOMIPS=softfloat`). Implications:

- The **Go binaries are per-target** (`GOARCH`/`GOARM`/`GOMIPS`); our CI
  produces a matrix of `.ipk`s. amneziawg-go is pure Go and cross-compiles to
  all of them.
- **Performance** of userspace WireGuard on a 32-bit MIPS SoC at ~600 MHz will
  be low (tens of Mbps), CPU-bound on crypto. aarch64 boxes (and x86) are fine.
  This bounds which routers client/node mode is *useful* on — document a tested
  hardware list once benchmarked.
- musl everywhere → keep `CGO_ENABLED=0`, no native deps.

---

## 8. Security posture

- **Client mode** turns the router into a single egress for the whole LAN —
  strong privacy win, but a **kill-switch is mandatory** (don't fall back to the
  clear WAN on tunnel drop; §3.3). Default the package to kill-switch-on.
- **DNS leak prevention** is part of the design, not an afterthought (§3.3).
- The PharosVPN unlinkability posture (DESIGN §3, no node→controller trace, per-
  server keys, endpoint rotation decision 17) is **inherited** by the OpenWrt
  client unchanged — `profile.dialEndpoint` already randomizes IP+port per
  connect; rotation just re-pins the endpoint route.
- **node/relay on a home IP** can deanonymize the operator's home as fleet
  infrastructure — counter to the "no node traces to the controller" posture for
  *public* nodes, but acceptable for an explicitly self-hosted box. Call this out
  in docs; don't auto-enroll a home node into a public fleet.
- LuCI exposure: the admin UI must be behind LuCI auth + ideally LAN-only;
  `luci-ssl` is installed on the VM, so serve it over HTTPS.
- opkg packages are **signed**; publish our feed signed and document the key, or
  users must `opkg --no-check-signature` (don't encourage that).

---

## 9. Effort estimate

Rough, assuming the Go core reuse holds and one engineer familiar with the
codebase. "Proof" = working on the x86 VM; multiply for the cross-arch matrix +
on-router perf hardening.

| Work item | Effort | Notes |
|---|---|---|
| **Client mode — standalone CLI + procd (option a)** | **S–M (1–2 wk)** | New `caravel-owrt` shell around existing `vp`/`profile`; tun + `ip` routing + endpoint pin + split-default; kill-switch; uci + procd. Highest reuse. |
| Client-mode fw4 NAT/kill-switch (uci) | S | Declarative uci; mirrors stock `wan` zone. |
| `luci-app-pharos-client` | M (1–2 wk) | JS views + rpcd ucode glue + ACL/menu; profile upload. |
| netifd `proto amneziawg.sh` (option b) | M | Optional polish; makes it a first-class interface. |
| opkg packaging + SDK Makefiles + signed feed | S–M | Per-binary `.ipk` + `luci-app` `all`; CI cross-arch matrix. |
| **relay packaging** (opkg/procd) | **S** | Pure Go; swap systemd→procd, apt→opkg paths in coxswain deploy. |
| **coxswain packaging** | S–M | Same; SQLite driver is already pure-Go (`modernc.org/sqlite`) — no CGO blocker. |
| **node — Port A: `UserspaceRuntime`** | **M (1–2 wk)** | New `awg.Runtime` impl on amneziawg-go UAPI; clean seam already exists. |
| **node — Port B: netpolicy on nft/fw4** | **M–L (2–4 wk)** | New rule dialect + cascade mark on nft + **a contract decision** with coxswain (byte-identical preview no longer holds). The genuinely hard part. |
| node reachability (inbound 8444 / relay reframe) | S–M | Mostly docs + firewall; or steer home boxes to the relay role. |
| **On-router performance hardening (all modes)** | **M, ongoing** | Benchmark userspace AWG on arm/mips; GSO/MTU tuning; hardware support list. |

**Recommended sequencing:** (1) client mode CLI+procd on x86 → (2)
`luci-app-pharos-client` → (3) relay packaging → (4) cross-arch CI + a real ARM
router → (5) node Ports A/B once client mode has proven the userspace engine on
hardware. coxswain-on-OpenWrt last (niche).

---

## 10. Open questions / what still needs proving

1. **On-router throughput of userspace amneziawg-go.** The decisive unknown.
   Benchmark on aarch64 + mipsel before promising node/full-tunnel client on
   small routers. (x86 VM is not representative.)
2. ~~coxswain's SQLite driver~~ — **resolved:** `modernc.org/sqlite` (pure-Go),
   no cgo, cross-compiles cleanly. Left here only to note it was checked.
3. **netpolicy contract under nftables.** DESIGN §3 mandates coxswain's preview
   and the node's applied rules be **identical**. On OpenWrt they cannot be
   (iptables vs nft). Decide: node-side translation of the canonical policy
   (smaller, keeps coxswain unaware), vs. a coxswain per-node firewall dialect.
   Lean node-side: `NetworkConfig` already sends *policy*, and the renderer is
   already node-local.
4. **iptables-nft shim vs native nft.** Is the compat shim stable enough across
   fw4 reloads to use even as a bring-up shortcut, or go straight to native nft?
5. **kmod-amneziawg, ever?** Is there appetite to maintain a per-target/per-
   kernel kernel-module feed for the throughput win, or is userspace the
   permanent answer? (Recommendation: userspace permanent; kmod only if a
   community feed appears.)
6. **netifd proto vs standalone service.** Confirm whether amneziawg-go can be
   driven cleanly from a `proto_*` shell helper (process lifecycle, fd handoff)
   or whether the standalone-service model is the only practical one.
7. **Real router validation.** Everything here is verified against an **x86 KVM
   VM**. The whole design must be re-validated on at least one real low-RAM,
   small-flash, non-x86 router (flash size, perf, `GOMIPS`, package size).
8. **Inbound reachability for node behind NAT.** Confirm the relay-reframe story
   so a home box can serve a useful server role without a public IP.

---

## Appendix A — VM facts cited (read-only inspection, 2026-06-06)

- `cat /etc/openwrt_release` → 24.10.0 / x86_64 / target x86/64.
- `uname -a` → `Linux 6.6.73 x86_64`; `df -h /` → 71.1 MB free of 98.3 MB.
- `/lib/ld-musl-x86_64.so.1 -> libc.so` (musl).
- `opkg find "*amnezia*"` / `"*awg*"` → **empty** (no AmneziaWG anywhere).
- `opkg list | grep wireguard` → `kmod-wireguard`, `wireguard-tools`,
  `luci-proto-wireguard` (stock WG only).
- `opkg list | grep golang` → `golang 1.23.12-r2` (+ `golang-src`).
- `kmod-tun 6.6.73-r1` in feed; `/dev/net/tun` not yet present (module unloaded).
- `/sbin/fw4` + `/usr/sbin/nft`, `nft list tables` → `table inet fw4`;
  **iptables not installed**; `ip-full 6.11.0-r1` available (BusyBox `ip` only
  by default).
- LuCI: `luci-base 25.x`, `rpcd-mod-luci`, `ucode` + `rpcd-mod-ucode`;
  `/usr/share/luci/menu.d/`, `/usr/share/rpcd/acl.d/`,
  `/www/luci-static/resources/`; `luci-app-package-manager` (GUI opkg) installed.
- `/etc/init.d/` has `network`, `firewall`, `dropbear` (procd model; dropbear
  sshd ⇒ decision-14 SSH enroll works).
