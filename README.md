# engineering-runtime-ci

CI test harness and live demo surface for Engineering Runtime.

Workflows here exercise the public install, a fresh Runtime Home, external
capability sources, auth, commands, capability execution, and the audit trail.
The runtime binary is never built in this repository. Agent guidance lives in
[`CLAUDE.md`](./CLAUDE.md).

## Runtime and Home contract

The shared [`setup-runtime`](.github/actions/setup-runtime/action.yml) action:

1. runs the public installer from
   [`engineering-runtime-releases`](https://github.com/engineeringruntime/engineering-runtime-releases);
2. relies on the installer's archive checksum verification;
3. bootstraps a disposable Home; and
4. verifies Runtime 0.6.0's fresh-Home ownership contract.

Every workflow sets only the common runtime environment it needs:

```yaml
ENGINEERING_RUNTIME_HOME: ${{ github.workspace }}/runtime-home
ENGINEERING_RUNTIME_CONSUMER: ci
```

On a fresh bootstrap:

- `config.yaml`, `policy-config.yaml`, and `context.yaml` are absent;
- compiled config and policy defaults apply;
- `commands/`, `specs/`, and other release-owned contracts are refreshed; and
- `capabilities/` exists as an empty, non-authoritative compatibility cache.

Organization, repository, and similar context is passed explicitly to the
operation that needs it. GitHub credentials are supplied only to auth/provider
steps.

## Capability sources

This repository checks in no capability Markdown. The shared
[`setup-capabilities`](.github/actions/setup-capabilities/action.yml) action
checks out the public
[`engineering-runtime-capabilities`](https://github.com/engineeringruntime/engineering-runtime-capabilities)
source (or another selected source) and returns:

- the exact `capabilities/` directory; and
- the checked-out commit SHA.

Workflows validate and execute the selected Markdown file by its exact path.
They do not present an external checkout as Runtime Home or depend on mutable
name resolution. The SHA in the action output identifies the content used by
the run.

`RUNTIME_CAPABILITIES_DIR` is not used as an authoritative-source selector in
these workflows. In Runtime 0.6.0 it relocates the implicit compatibility cache,
which `runtime capability list` reports as non-authoritative.

## Setup

| Kind | Name | Purpose |
|---|---|---|
| Secret | `RUNTIME_GITHUB_TOKEN` | GitHub Auth Engine and provider operations; also required when a selected capability source or target repository is private. It is not needed to install the public runtime. |
| Variable (optional) | `GITHUB_ORG` | Organization passed explicitly to org-scoped operations; defaults to `github.repository_owner`. |

## Workflows

| Workflow | Trigger | What it runs |
|---|---|---|
| [`runtime-ci.yaml`](.github/workflows/runtime-ci.yaml) | push/PR/dispatch | Public install → fresh-Home assertions → external source checkout → validate all → auth-free files round trip → GitHub auth/commands → audit |
| [`capabilities-from-registry.yaml`](.github/workflows/capabilities-from-registry.yaml) | weekly + dispatch | Clone the complete selected source → capture commit SHA → validate all exact paths → optionally execute one exact path |
| [`scaffold-service-docs-push.yaml`](.github/workflows/scaffold-service-docs-push.yaml) | dispatch | Execute `files/scaffold-service-docs.md` from the selected source and push the generated files |
| [`java-service-scaffold-and-ship.yaml`](.github/workflows/java-service-scaffold-and-ship.yaml) | dispatch | Capture a registry revision, create a fresh repo, and execute the Java scaffold capability by exact path |
| [`github-repo-health.yaml`](.github/workflows/github-repo-health.yaml) | weekly + dispatch | `github/github-repo-health.md` |
| [`github-org-health.yaml`](.github/workflows/github-org-health.yaml) | weekly + dispatch | `github/github-org-health-check.md` |
| [`github-security-posture.yaml`](.github/workflows/github-security-posture.yaml) | weekly + dispatch | `github/github-security-posture.md` |
| [`github-review-queue.yaml`](.github/workflows/github-review-queue.yaml) | weekdays + dispatch | `github/github-review-queue.md` |
| [`github-standards-audit.yaml`](.github/workflows/github-standards-audit.yaml) | weekly + dispatch | `github/github-repo-standards-audit.md` |
| [`github-branch-protection-audit.yaml`](.github/workflows/github-branch-protection-audit.yaml) | weekly + dispatch | `github/github-branch-protection-audit.md` |
| [`github-secrets-inventory.yaml`](.github/workflows/github-secrets-inventory.yaml) | monthly + dispatch | `github/github-secrets-inventory.md` |

Destructive packs remain dispatch-only or local.

## Local dry-run

Use an isolated Home and an exact capability file from a reviewed checkout:

```bash
export ENGINEERING_RUNTIME_HOME="$(mktemp -d)/runtime-home"
export ENGINEERING_RUNTIME_CONSUMER=ci

runtime bootstrap
runtime config validate
runtime capability list

CAPABILITY_SOURCE=../engineering-runtime-capabilities/capabilities
runtime capability validate "${CAPABILITY_SOURCE}/files/notes-roundtrip.md"
runtime capability execute "${CAPABILITY_SOURCE}/files/notes-roundtrip.md" \
  --input path=./notes.txt \
  --input message="hello from engineering-runtime-ci"
runtime audit tail -n 20
```

Record `git -C ../engineering-runtime-capabilities rev-parse HEAD` with the
result when the run is evidence. `runtime-home/` is gitignored and must never
be committed.

## Layout

```text
.github/
  actions/setup-runtime/       # public install, bootstrap, fresh-Home checks
  actions/setup-capabilities/  # checkout plus directory/revision outputs
  workflows/                   # CI samples and dispatchable demos
runtime-home/                  # generated and gitignored
```
