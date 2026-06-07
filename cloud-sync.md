# Cloud sync — the client UX contract

How a caravel client logs into an account, pulls its profiles from the
controller, and presents the cloud session. This is **platform-agnostic**: the
macOS app (`caravel-mac`) is the reference implementation; iOS and Android must
behave the same. The shared Go core (`caravel/go`) provides the wire/crypto
primitives; only the keystore and the UI are per-platform.

> Companion to [DESIGN.md](./DESIGN.md) §8–9 (account sync, the `.pharos`
> format). This doc is the **UX + state** contract; DESIGN is the protocol.

---

## 1. The two ways to get a profile

- **Import** — the user picks a `.pharos` file by hand (no controller, no
  account). File-imported profiles are fully local: editable, deletable, never
  touched by sync.
- **Sync (cloud)** — the user logs into their account and the client fetches its
  profiles from the controller. This doc is about sync.

A client always supports both, side by side.

## 2. Identity and on-disk files

A synced ("cloud") profile is three files in the profile store, sharing a base
name `<name>`:

| File | What it is | Secret? |
|---|---|---|
| `<name>.pharos` | The profile **bundle** — `profiles[]` + the controller location. Stored **decrypted** on-device (the controller only ever held ciphertext; decryption happened here on sync). | holds tunnel keys |
| `<name>.pharosid` | The **device identity** — the device's mTLS leaf + the relay endpoint + the fleet CA. Issued by `coxswain devices issue`. The login credential. | holds a private key |
| `<name>.synced` | A small JSON **marker** that flags the bundle as cloud-synced and records the last sync. | no |

The **account passphrase** is *not* a file — it lives in the platform keystore
(§4). An imported profile has only a `.pharos` and **no** `.synced` marker;
that absence is how a client tells cloud from imported.

### `.synced` marker schema

```json
{ "user": "you@example.com",
  "revision": 7,
  "relay": "157.245.255.196:443",
  "controller": "<fleet CA fingerprint>",
  "synced_at": "2026-06-06T23:17:35Z" }
```

## 3. The logged-in model

**Login = first successful sync.** The user provides a `.pharosid` (device file)
and the account passphrase; the client fetches + decrypts the bundle, stores it,
and **saves the passphrase in the keystore**. From then on the session is
"logged in": re-sync is one tap.

**Logout** clears the keystore entry **and removes every cloud-synced profile**
(every bundle with a `.synced` marker, plus its sidecars). Imported profiles are
untouched. After logout the client is back to "not logged in".

There is **one** logged-in account at a time — see §5.

## 4. Keystore

The account passphrase is the only thing persisted, in the OS secure store:

- **macOS / iOS** — Keychain (`kSecClassGenericPassword`, accessible after first
  unlock). Reference: `caravel-mac/app/Caravel/Keychain.swift`.
- **Android** — Keystore-backed `EncryptedSharedPreferences` (or equivalent).

One item is enough (one account). Never write the passphrase to a file or the
process table; the worker takes it on **stdin** (`sync --password-stdin`).

## 5. Sync is replace-all (one controller)

A sync **replaces the entire cloud set**: before storing the freshly-fetched
bundle, the client deletes *all* existing cloud-synced bundles (every `.synced`
one) and their sidecars. Consequences:

- Re-syncing never accumulates duplicates or leaves stale profiles.
- Switching to a different controller drops the previous controller's profiles.
- Imported (non-`.synced`) profiles are never deleted.

So the cloud set is always *exactly* the latest sync. Reference:
`purgeCloudProfiles` in `caravel-mac/cmd/caravel-mac/main.go`.

## 6. What's in a bundle

```jsonc
{ "fleet_id": "...", "user": "...", "revision": 7,
  "profiles": [ /* the named profiles — see the profiles contract */ ],
  "control": { "label": "Controller · New York", "city": "New York",
               "lat": 40.7128, "lon": -74.006 } }
```

- `profiles[]` — the device's named connection configs (each: an egress, an
  entry-IP subset, a protocol of `amneziawg` | `xray-reality` | `both`). Picking
  + connecting is the profiles contract, not this doc.
- `control` — the **control-plane endpoint** the client syncs through (the relay,
  geo-resolved by coxswain). Coordinates are embedded so an **offline** client
  can place the pin. **Optional**: absent when the controller has neither a geoip
  database nor a `control_location` config — then the client simply draws no
  controller pin. It is *not* a hidden-controller leak: it's the endpoint the
  client already talks to.

## 7. Controller status (informational)

The client surfaces three things about the session. **None of them block
connecting** — the controller is a control-plane, not a data-plane, dependency,
so the data plane keeps running even if the controller is briefly unavailable.

| Field | Source | Meaning |
|---|---|---|
| **reachable** | a short **TLS dial** to the relay (`.pharosid` `RelayAddr`) with the device leaf — no auth, no fetch | the control-plane endpoint answered just now |
| **last synced** | `synced_at` + `relay` from the `.synced` marker | when/where the bundle was last pulled |
| **location** | `control` from the bundle | where to draw the controller on the map |

Core helper: `sync.Reachable(ctx, bundle, timeout)` (`caravel/go/sync`).
Reference command: `caravel-mac controller-status <bundle>` →

```json
{ "reachable": true, "last_synced_at": "2026-06-06T23:17:35Z",
  "relay": "157.245.255.196:443",
  "controller": { "label": "Controller · New York", "lat": 40.7128, "lon": -74.006 } }
```

### Polling — mind the battery

Reachability is a network round-trip, so:

- **Mobile (iOS/Android):** check **on app foreground**, **before a sync**, and
  on a manual refresh. **Do not poll on a timer** while backgrounded or idle.
- **Desktop (macOS):** a gentle poll (≈30 s) while the window is open is fine.

## 8. Actions

- **Sync now** — re-fetch using the stored `.pharosid` + the keystore passphrase
  (`sync <…>.pharosid --password-stdin`, no email needed — the device leaf
  authenticates). With no stored passphrase, fall back to the login sheet. This
  is replace-all (§5).
- **Log out** — disconnect if a cloud profile is up, run the logout purge
  (`caravel-mac logout` → `purgeCloudProfiles`), and delete the keystore entry.
- **Connect / Disconnect** — never require the controller; they act on the local
  bundle only.

## 9. The map

Two planes, two line styles (DESIGN §3 convention):

- **Control plane — solid line**, *You → Controller*. The controller pin comes
  from the bundle's `control` coords; its dot reflects **reachable**. This is the
  line you sync over.
- **Data plane — dashed line**, *You → entry → [mid] → exit*, from the selected
  profile's nodes/path. Drawn (and "live") when connected.

Reference: `caravel-mac/app/Caravel/{TunnelController.swift,LandMap.swift}`
(`PinKind.controller`, `ArcStyle.controlPlane`).

## 10. Worker / core surface (what each client wires to)

| Need | Shared core (`caravel/go`) | Reference CLI (`caravel-mac`) |
|---|---|---|
| Fetch + decrypt the bundle | `sync.Fetch(ctx, bundle, email, pass)` | `sync … --password-stdin` |
| Reachability | `sync.Reachable(ctx, bundle, timeout)` | `controller-status <bundle>` |
| Parse the bundle (`profiles[]`, `control`) | `profile.Parse` → `Profile` | — |
| Replace-all on sync | — | `purgeCloudProfiles` |
| Logout purge | — | `logout` |
| Connect a named profile | — | `connect --profile <b> --name <p> [--protocol]` |

A native mobile client links `caravel/go` (via gomobile) and calls
`sync.Fetch` / `sync.Reachable` / `profile.Parse` directly; the file-store
conventions (§2, §5) it reimplements per the rules above.

---

## Reference implementation

macOS: [caravel-mac](https://github.com/PharosVPN/caravel-mac) —
`cmd/caravel-mac/{main,daemon,state}.go` (worker) and
`app/Caravel/{TunnelController,ContentView,Profiles,Keychain,LandMap}.swift`
(UI). Controller side: [coxswain](https://github.com/PharosVPN/coxswain)
`internal/profile` (`Control`) + `internal/provision` + `internal/config`
(`control_location`).
