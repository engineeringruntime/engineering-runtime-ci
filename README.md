# engineering-runtime-samples

**CI test harness and live demo surface** for Engineering Runtime.

Workflows here extensively exercise install → bootstrap → env vars →
auth → capabilities/commands → audit, so upgrades and demos run in
GitHub Actions instead of on a laptop. Agent guidance:
[`CLAUDE.md`](./CLAUDE.md). The runtime binary is never built in this
repo.

The binary always comes from
[`kishore-gutta/engineering-runtime-releases`](https://github.com/kishore-gutta/engineering-runtime-releases)
(latest linux-amd64, checksum-verified).

**Config and policy come only from the Runtime Home.** Each job sets:

```yaml
ENGINEERING_RUNTIME_HOME: ${{ github.workspace }}/runtime-home
RUNTIME_CONFIG_FILE:      ${{ github.workspace }}/runtime-home/config.yaml
RUNTIME_POLICY_FILE:      ${{ github.workspace }}/runtime-home/policy-config.yaml
```

`runtime bootstrap` seeds those files (plus context/commands/specs) from
the binary into `runtime-home/`. The `RUNTIME_*_FILE` vars point at that
same home — not at a separate checked-in copy — so upgrade /
backward-compat testing stays simple: swap the release (or local binary),
wipe `runtime-home`, bootstrap again, re-run.

Workflows execute capability files under `examples/capabilities/` by
**full path**. Do not point `RUNTIME_CAPABILITIES_DIR` at that folder —
every `runtime` invocation re-runs bootstrap and would seed the binary's
full capability tree into the working tree.

## Setup

| Kind | Name | Purpose |
|---|---|---|
| Secret | `RUNTIME_GITHUB_TOKEN` | GitHub PAT for Auth Engine + GitHub provider ops (`repo`, `read:org`, `notifications`, plus scopes each workflow needs — e.g. `security_events` for security posture) |
| Variable (optional) | `GITHUB_ORG` | Org written into Runtime Context; defaults to `github.repository_owner` |

Release download reuses the built-in `${{ github.token }}` as `GH_TOKEN`.

## Workflows

Shared install: [`.github/actions/setup-runtime`](.github/actions/setup-runtime).

| Workflow | Trigger | What it runs |
|---|---|---|
| [`runtime-ci.yaml`](.github/workflows/runtime-ci.yaml) | push/PR/dispatch | Install → bootstrap home → `config validate` → validate sample caps → `files/notes-roundtrip` → `auth` → `github user get` / `org list` / `notification list` → audit |
| [`capabilities-from-registry.yaml`](.github/workflows/capabilities-from-registry.yaml) | weekly + dispatch | Install → bootstrap home → **clone** `engineering-runtime-capabilities` → `RUNTIME_CAPABILITIES_DIR` → validate/execute one capability **by name** (registry stand-in) |
| [`github-repo-health.yaml`](.github/workflows/github-repo-health.yaml) | weekly + dispatch | `github/github-repo-health` |
| [`github-org-health.yaml`](.github/workflows/github-org-health.yaml) | weekly + dispatch | `github/github-org-health-check` |
| [`github-security-posture.yaml`](.github/workflows/github-security-posture.yaml) | weekly + dispatch | `github/github-security-posture` |
| [`github-review-queue.yaml`](.github/workflows/github-review-queue.yaml) | weekdays + dispatch | `github/github-review-queue` |
| [`github-standards-audit.yaml`](.github/workflows/github-standards-audit.yaml) | weekly + dispatch | `github/github-repo-standards-audit` |
| [`github-branch-protection-audit.yaml`](.github/workflows/github-branch-protection-audit.yaml) | weekly + dispatch | `github/github-branch-protection-audit` |
| [`github-secrets-inventory.yaml`](.github/workflows/github-secrets-inventory.yaml) | monthly + dispatch | `github/github-secrets-inventory` |
| [`github-repositories.yaml`](.github/workflows/github-repositories.yaml) | dispatch | `github/github-repositories` |
| [`github-access-review.yaml`](.github/workflows/github-access-review.yaml) | dispatch | `github/github-access-review` (team + username) |
| [`github-incident-forensics.yaml`](.github/workflows/github-incident-forensics.yaml) | dispatch | `github/github-incident-what-changed` |
| [`github-ci-failure-triage.yaml`](.github/workflows/github-ci-failure-triage.yaml) | dispatch | `github/github-ci-failure-triage` (`run_id`) |

Destructive packs (service onboarding, release cut) are **not** wired as
CI workflows — run those locally from
`../engineering-runtime/capabilities/github/`.

## Local dry-run / upgrade check

```bash
export ENGINEERING_RUNTIME_HOME="$PWD/runtime-home"
export RUNTIME_CONFIG_FILE="$ENGINEERING_RUNTIME_HOME/config.yaml"
export RUNTIME_POLICY_FILE="$ENGINEERING_RUNTIME_HOME/policy-config.yaml"
export RUNTIME_GITHUB_TOKEN=ghp_xxx

# Fresh home from whatever binary is on PATH (release or local build)
rm -rf "$ENGINEERING_RUNTIME_HOME"
runtime bootstrap
runtime config validate
runtime capability validate examples/capabilities/github/github-repo-health.md
runtime capability execute examples/capabilities/github/github-repo-health.md \
  --input repository=cli/cli

# Upgrade / backward-compat: install another version, wipe home, repeat
# rm -rf "$ENGINEERING_RUNTIME_HOME" && runtime bootstrap && …
```

`runtime-home/` is gitignored — never commit it.

## Layout

```
.github/
  actions/setup-runtime/     # download release + bootstrap + seed context org
  workflows/                 # CI samples above
examples/capabilities/
  files/                     # auth-free demos
  github/                    # GitHub demos used by workflows
runtime-home/               # generated by bootstrap (gitignored) — config, policy, specs, commands
```
