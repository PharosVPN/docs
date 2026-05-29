# PharosVPN — Global Build Conventions

Conventions every PharosVPN repo follows. Each subproject's own `BUILD.md` is the
specific brief; **this** file is the shared baseline. A subagent assigned a
subproject must read, in order: this file → `docs/DESIGN.md` → the subproject's
`BUILD.md`.

---

## 1. Before writing any code

1. Read `docs/DESIGN.md` end to end. It is the source of truth.
2. Read your subproject's `BUILD.md`.
3. If the design is silent or contradictory on something you need, **stop and
   raise it** — do not invent a contract. Update `docs/DESIGN.md` in the same PR
   that depends on the decision.

## 2. Languages & versions

- **`coxswain`, `buoy`, `beacon`** — Go (latest stable, currently 1.25.x).
  Module paths: `github.com/PharosVPN/<repo>`.
- **`coxswain` admin UI** — SvelteKit 2 + Svelte 5, TypeScript, Tailwind. Built to
  static assets, embedded in the Go binary via `//go:embed`.
- **`caravel`** — native: Kotlin (Android) + Swift (iOS). VPN tunnelling needs
  `VpnService` / `NetworkExtension`, which are platform-native. Cross-platform
  framework choice is decided in `caravel/BUILD.md`, not here.

## 3. Wire contracts

- gRPC over mTLS. Protobuf schemas are the contract; they live in `docs/proto/`.
- **Never hand-roll a message type** that crosses a process boundary. Add it to
  the proto, regenerate, use the generated type.
- Schemas are versioned. Add fields, never repurpose tags. Unknown fields are
  ignored, never rejected.

## 4. Reused code — rebrand obligation

`buoy` and `beacon` adapt the reverse-tunnel, transparent-proxy, and device-CA
machinery the operator wrote for an earlier private project. When adapting it:

- Strip **every** identifier from the origin project — package names, import
  paths, type names, metadata keys, comments, file names.
- The new repo must contain **zero trace** of the origin project.
- Re-license headers to AGPL-3.0 (see §6).

## 5. Repo layout (Go projects)

```
<repo>/
  cmd/<binary>/main.go
  internal/...            # all non-exported packages
  LICENSE                 # full AGPL-3.0 text
  README.md
  BUILD.md
  CONTRIBUTING.md         # DCO instructions
  go.mod
  .github/workflows/      # CI: build, vet, test, lint
```

## 6. License & commits

- License: **AGPL-3.0-or-later**, whole platform. Every source file carries the
  short AGPL header. Every repo ships the full `LICENSE` text.
- Commits use **Conventional Commits** (`feat:`, `fix:`, `docs:`, `perf:`, …).
- Every commit is **signed off** (`git commit -s`) — DCO, no CLA.
- Branch from `main`; do not commit straight to `main`. PRs only.

## 7. Quality bar

- `go vet` and `gofmt` clean. Lint with `golangci-lint`.
- Unit tests for logic; integration tests for anything crossing mTLS.
- No secrets in git. Ever. Not in test fixtures either.
- A subproject is "done" when it builds, tests pass in CI, its README is
  accurate, and it interoperates with `coxswain` against the shared protos.

## 8. Security posture (non-negotiable)

- `coxswain` opens **no inbound ports**. All connections are coxswain-initiated outbound.
- `buoy` and `beacon` accept only mTLS; certs must chain to the in-repo CA.
- `beacon` never sees plaintext profile bundles — only `account`-mode ciphertext.
- User private keys never exist in usable form on `coxswain` (see DESIGN §8).
