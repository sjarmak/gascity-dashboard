# Architecture diagram (LikeC4)

Architecture-as-code model of `gas-city-dashboard`, rendered with
[LikeC4](https://likec4.dev). The model is the source of truth across
[`spec.c4`](spec.c4) (element kinds, tags, deployment node kinds),
[`model.c4`](model.c4) (the system), and [`views.c4`](views.c4) (structure,
walkthrough, and risk views), with the deployment model in
[`deployment.c4`](deployment.c4). The narrative companions are the repo-root
[`README.md`](../README.md) and the engineer's-eye
[`specs/architecture/overview.md`](../specs/architecture/overview.md); the
binding visual register is [`DESIGN.md`](../DESIGN.md).

Every element `link`s to its source (`backend/src/…`, `frontend/src/…`,
`shared/src/…`) and, where one exists, to the relevant `specs/architecture/…`
document — so any box in the explorer is one click from the code and the
contract behind it.

## The architectural spine

The durable boundary
([`direct-supervisor-boundary.md`](../specs/architecture/direct-supervisor-boundary.md))
is that **GC-owned data is read by the browser directly from the gc
supervisor**, while the **dashboard service owns only local host
capabilities** — static hosting, runtime config, `git`/`gh`/build evidence,
host/dolt-noms health, client-error telemetry, and audit rows. The backend
binds `127.0.0.1` only and ships no auth; network exposure is the operator's
own contract ([`exposure.md`](../specs/architecture/exposure.md)). The `shared`
workspace is the wire-shape SSOT both halves import.

## Delivery state is tagged, not guessed

Every element carries a tag so **opt-in and planned work renders distinctly
from what is already built** (legend in `spec.c4`):

| Tag | Meaning | Render |
|---|---|---|
| `#built` | code path exists and is exercised (and tested) | solid |
| `#evolving` | built, but the contract / upstream surface is still moving | solid |
| `#planned` | designed; not yet implemented or OFF by default (gated module) | **dashed, dimmed** |
| `#research` | speculative track with no landed code | **dashed, indigo** |

Items below `#built` in the model: the generated supervisor client and run
projection / run-detail components are `#evolving` (they track the upstream
supervisor spec); the Maintainer (Triage) module (backend + frontend) is
`#planned` because it is opt-in and OFF by default (`MODULES_ENABLED=maintainer`).

## Views

**Structure** — the static map:

| View | Scope |
|---|---|
| `index` | system landscape — the dashboard in context of the supervisor, host CLIs, and GitHub |
| `dashboardSystem` | the system decomposed into its three workspaces (backend / frontend / shared) |
| `backendContainer` | backend (Node/Express) — config, security, transport proxy, per-city plane, exec wrapper, local routes |
| `frontendContainer` | frontend (React/Vite) — app shell, supervisor client, attention model, live hooks, domain workspaces |
| `sharedContainer` | the wire-shape SSOT — dashboard DTOs, the generated gc-supervisor client, run projection, fixtures |
| `planned` | the opt-in Maintainer (Triage) module, with the built dependencies it mounts into dimmed |
| `deployment` | where each piece runs — loopback-only backend, dev Vite vs prod static, supervisor source, browser over SSH |

**Walkthrough flows** (dynamic / numbered-step views) — the narrative spine for
a design-review walkthrough:

| View | Flow |
|---|---|
| `boot` | SPA bootstrap to first paint (load → /api/config → first GC read through the relay) |
| `liveUpdate` | the live-update spine over SSE (EventSource → coalesced refresh → attention → nav badges) |
| `operatorWrite` | an operator write (Origin + CSRF + X-GC-Request; mail sends always from the operator) |
| `runDiff` | a run git diff as local evidence (cwd confined to RUN_CWD_ALLOWED_ROOTS before `git -C`) |

**Risk lens:**

| View | Scope |
|---|---|
| `risks` | the `#risk`-flagged elements with each open question stated in-box (transport-proxy read-only is off by default, untrusted run-diff cwd, generated-client drift / no response validator, read-only mail impersonation) |

### Running the walkthrough

For a design review, present in this order: `index` → `dashboardSystem`
(orient on structure) → the four walkthrough flows in sequence (what actually
happens) → `deployment` (where it runs) → `risks` (what to probe) → `planned`
(the opt-in module). In `npx likec4 start`, the dynamic views animate
step-by-step and each view's notes panel carries the gotchas (the
loopback-only bind, the direct-supervisor boundary, the always-from-operator
mail rule).

## Viewing & regenerating

```bash
# Interactive, hot-reloading explorer (recommended)
npx likec4 start architecture

# Re-export the static PNGs (needs a one-time browser download:
#   npx playwright install chromium-headless-shell)
npx likec4 export png architecture -o architecture/exports

# Validate the model (strict — the source of truth for correctness)
npx likec4 validate architecture
```

### Viewing the interactive explorer over SSH (headless remote)

`likec4 start` serves a Vite dev server on `localhost:5173`. From a headless
remote, forward that port to your laptop and open it locally — three options,
easiest first:

1. **VS Code / Cursor Remote-SSH** — run `npx likec4 start architecture` in the
   integrated terminal; the editor auto-forwards 5173 and offers "Open in
   Browser". Nothing else to configure.
2. **SSH local port-forward** — on your laptop:
   ```bash
   ssh -N -L 5173:localhost:5173 user@remote   # leave running
   ```
   then on the remote `npx likec4 start architecture` and open
   <http://localhost:5173> locally. (Already in an SSH session? Add the tunnel
   without reconnecting: press `~C` then type `-L 5173:localhost:5173`.)
3. **Bind + reach directly** — `npx likec4 start architecture --listen 0.0.0.0`
   and browse to `http://<remote-ip>:5173` (only if that port is reachable /
   firewall-open; the tunnel in option 2 is safer).

The same SSH-forward shape is exactly how the dashboard itself is reached — the
backend binds `127.0.0.1` only, so the operator forwards `:8082` (prod) or
`:5174` (dev) to her laptop the same way.
