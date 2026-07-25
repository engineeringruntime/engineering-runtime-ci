# Engineering Runtime Samples — CI test harness & live demos

## Mission

This repo is the **out-of-band proving ground** for the Engineering Runtime
binary. It exists so that:

1. **CI extensively tests the runtime** the way real users and pipelines
   run it — install a release, bootstrap a Runtime Home, set env vars,
   authenticate, execute capabilities and commands, read the audit trail.
2. **Demos run here, not on a laptop.** Workflows, Actions logs, and
   `workflow_dispatch` inputs are the public demonstration surface. Prefer
   adding or running a workflow over "works on my machine" walkthroughs.
3. **Upgrade and backward-compat are cheap.** Wipe `runtime-home/`, point
   at another release (or a local build), bootstrap again, re-run the same
   workflows.

This repo has **no runtime source** (`no internal/`, no `main.go`). The
binary always comes from
[`kishore-gutta/engineering-runtime-releases`](https://github.com/kishore-gutta/engineering-runtime-releases)
in CI, or from `../engineering-runtime` when dry-running locally. Source
of truth for behavior remains `../engineering-runtime`; this repo only
exercises it. For AI-assisted engineering that must stay on the runtime
path, see `../engineering-runtime-ai-agent`.

When the workspace map (`../CLAUDE.md`) still describes this folder as an
empty scaffold, **this file wins** for what the samples repo actually is
today.

---

## What must be tested here

Every change that adds a workflow, capability demo, or CI step should move
coverage toward this checklist. Gaps are bugs in *this* repo's charter,
not optional nice-to-haves.

### A. Runtime install

- Download the latest (or pinned) linux-amd64 release archive
- Verify `SHA256SUMS.txt`
- Put `runtime` on `PATH`
- `runtime version` succeeds

Owned by: `.github/actions/setup-runtime/`.

### B. Bootstrap & Runtime Home

- `ENGINEERING_RUNTIME_HOME` → throwaway `runtime-home/` (gitignored)
- `runtime bootstrap` seeds from the binary: `config.yaml`,
  `policy-config.yaml`, `context.yaml`, `commands/`, `specs/`,
  `capabilities/`
- Idempotent re-bootstrap does not wipe user-owned edits (when we add
  tests that edit home files, assert that)
- Version bump / wipe-home / re-bootstrap is the upgrade path

### C. Environment variables

The full list lives in the runtime repo:
`../engineering-runtime/commands/runtime_env_variables.txt`
(also seeded into `runtime-home/commands/` after bootstrap).

**Every supported env var should eventually have CI coverage** — either a
dedicated workflow/job or an explicit step inside `runtime-ci.yaml`.
Minimum matrix this repo is responsible for:

| Variable | What CI should prove |
|---|---|
| `ENGINEERING_RUNTIME_HOME` | Home is relocatable; bootstrap + all commands use it |
| `RUNTIME_CONFIG_FILE` | Points at `runtime-home/config.yaml` (home-owned, not a second tree) |
| `RUNTIME_POLICY_FILE` | Points at `runtime-home/policy-config.yaml` |
| `RUNTIME_GITHUB_TOKEN` | Auth Engine login/status/logout + GitHub provider ops |
| `ENGINEERING_RUNTIME_CONSUMER` | Set to `ci` in workflows; appears on audit records |
| `CI` | Present in Actions; consumer fallback behaves |
| `RUNTIME_CAPABILITIES_DIR` | Point at the **company capability store** after bootstrap (GitHub clone, registry unpack dir, shared path — see §F). Never point it at this repo's `examples/` during bootstrap |
| `RUNTIME_AUTHENTICATION_*` | At least one AutomaticEnv override (e.g. base URL or log level) is applied and visible via `runtime config validate` |
| `KUBECONFIG` | Covered when/if a kubernetes-auth demo workflow is added |
| `USER` / `USERNAME` | Executor on audit records (usually OS-provided) |

Do **not** invent a parallel checked-in config/policy tree. Home-only +
explicit `RUNTIME_*_FILE` paths into that home is the contract.

### D. Core CLI surface

Smoke (`runtime-ci.yaml`) and specialized workflows should collectively
hit:

- `runtime bootstrap`
- `runtime config validate`
- `runtime context show` / seeded org context
- `runtime auth status` → `login` → `logout`
- `runtime github …` (curated ops + at least one `api` / `graphql` path
  via capabilities)
- `runtime capability validate` / `execute` (auth-free `files` + GitHub)
- `runtime audit tail`
- `runtime resolve …` where useful for zero-side-effect checks

### E. Capabilities as demos

`examples/capabilities/` are both **regression fixtures** and **demo
scripts**. A human or customer should be able to open Actions → run
workflow → read JSON/audit output and understand what the runtime did,
without cloning `engineering-runtime` or configuring a laptop.

### F. Future state — capabilities come from the company store

**Runtime Home is not the long-term home for a company's capability
library.** Bootstrap may seed example capabilities into Home for a fresh
install, but real orgs will keep capabilities wherever *they* already
store engineering knowledge:

| Store (examples) | How CI / the runtime reaches it today |
|---|---|
| A GitHub (or GitLab) capabilities repo | Clone → set `RUNTIME_CAPABILITIES_DIR` → execute by name |
| An internal Capability Registry | Future: `runtime` fetch/install into a local dir, then the same `RUNTIME_CAPABILITIES_DIR` (or successor) seam |
| A shared filesystem / monorepo path | Point `RUNTIME_CAPABILITIES_DIR` at that path |

The invariant to protect in this samples repo:

- **Binary owns** Runtime Home refresh (config, policy, specs, commands)
  on every install/upgrade.
- **Company owns** where capabilities live. The runtime only needs a
  directory of Markdown capabilities to validate/execute — it must not
  assume GitHub vs registry vs disk.
- **`capabilities-from-registry.yaml`** is the CI stand-in for that
  future: Home still refreshes from the binary; capabilities are pulled
  from an external store (today: clone
  `engineering-runtime-capabilities`) and resolved by name.

When adding workflows, prefer exercising that external-store path over
growing `examples/capabilities/` — examples stay for smoke/demos; the
company store path is what customers will actually run.

---

## Runtime Home contract (always)

```yaml
ENGINEERING_RUNTIME_HOME: ${{ github.workspace }}/runtime-home
RUNTIME_CONFIG_FILE:      ${{ github.workspace }}/runtime-home/config.yaml
RUNTIME_POLICY_FILE:      ${{ github.workspace }}/runtime-home/policy-config.yaml
ENGINEERING_RUNTIME_CONSUMER: ci
```

1. Bootstrap seeds home from the binary.
2. `RUNTIME_CONFIG_FILE` / `RUNTIME_POLICY_FILE` refer to **that** home.
3. Never commit `runtime-home/`.
4. Never set `RUNTIME_CAPABILITIES_DIR` to `examples/capabilities` —
   execute in-repo sample capabilities by **full path** instead.
5. Capabilities are resolved from **wherever the company stores them**
   (GitHub repo, Capability Registry, shared disk — see §F). After
   bootstrap, clone/fetch the **entire** store, set
   `RUNTIME_CAPABILITIES_DIR` to its `capabilities/` tree, then validate
   (and optionally execute) **by name**. Today's CI stand-in:
   `.github/workflows/capabilities-from-registry.yaml` (clone all → set
   env → validate all → optional single execute).

Upgrade / backward-compat drill:

```bash
export ENGINEERING_RUNTIME_HOME="$PWD/runtime-home"
export RUNTIME_CONFIG_FILE="$ENGINEERING_RUNTIME_HOME/config.yaml"
export RUNTIME_POLICY_FILE="$ENGINEERING_RUNTIME_HOME/policy-config.yaml"
rm -rf "$ENGINEERING_RUNTIME_HOME"
# install other runtime version on PATH, then:
runtime bootstrap
runtime config validate
# re-run the same capability / workflow steps
```

---

## Layout

| Path | Role |
|---|---|
| `.github/actions/setup-runtime/` | Install release + bootstrap + optional context org |
| `.github/workflows/` | CI tests **and** demo entrypoints (`workflow_dispatch`) |
| `examples/capabilities/{files,github}/` | Vendored demo/regression capabilities (full-path execute) |
| `engineering-runtime-capabilities/` | Ephemeral clone in CI (gitignored) — one example of a company capability store (GitHub); a registry unpack dir would plug in the same way |
| `runtime-home/` | Generated by bootstrap (gitignored) |

Specs and cheatsheets are **not** checked in — read
`runtime-home/specs/`, `runtime-home/commands/`, or
`../engineering-runtime/`.

---

## Conventions for agents working in this repo

1. **Prefer a new or extended workflow over a local-only script** when
   demonstrating or testing runtime behavior. Demos belong in Actions.
2. **If you add a runtime env var in `../engineering-runtime`, add CI
   coverage here** in the same effort (or leave a failing/TODO job that
   names the gap). Source list:
   `../engineering-runtime/commands/runtime_env_variables.txt`.
3. **If you change bootstrap / install / home layout in the runtime,
   add or tighten a workflow here** that would have caught the break.
4. **Only execute what the installed binary can do.** Prefer
   `provider: github` / `provider: files`. Use `binary: gh` only as an
   escape hatch. New operations land in `../engineering-runtime` first.
5. **`runtime capability validate` before commit** for every new/edited
   example capability.
6. **No hardcoded orgs/repos/secrets in capabilities** — declare
   `inputs:`; workflows pass them from `github.repository`,
   `vars.GITHUB_ORG`, or `workflow_dispatch` inputs. Tokens only via
   `RUNTIME_GITHUB_TOKEN` (Actions secret).
7. **Reusable library capabilities** belong in the company store
   (today: `../engineering-runtime-capabilities` on GitHub; later: also
   a registry). This samples repo keeps only CI smoke/demo fixtures.
8. **Keep vendored examples in sync** with
   `../engineering-runtime/capabilities/` when you depend on a change.
9. **Destructive packs** (repo create + protection, release cut, DELETE)
   stay dispatch-only or local — default policy denies `api DELETE`;
   don't put irreversible steps on `push`/`schedule`.

---

## Secrets & variables

| Kind | Name | Purpose |
|---|---|---|
| Secret | `RUNTIME_GITHUB_TOKEN` | (1) Download binaries from `engineering-runtime-releases` via `gh release download` (`contents: read`); (2) Auth Engine + GitHub provider. Passed into `setup-runtime` as `runtime_github_token` → `GH_TOKEN`. |
| Variable (optional) | `GITHUB_ORG` | Seeded into Runtime Context; default `github.repository_owner` |

`github.token` from this repo cannot read another **private** repo's
releases — that produces `release not found` even when releases exist.
Either keep using `RUNTIME_GITHUB_TOKEN`, or make
`engineering-runtime-releases` public (preferred long-term).

---

## Architecture reminder

```
Intent → Capability (from company store: GitHub / registry / …)
       → Engineering Runtime → Engines → Industry Tools
```

This repo sits **outside** the binary: it installs the runtime, prepares
its environment (Home from the binary; capabilities from the company
store), and runs real intents so CI and customers see the same
deterministic path a human would — without needing the author's laptop.
