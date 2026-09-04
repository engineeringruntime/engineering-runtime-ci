# Engineering Runtime CI — test harness and live demos

## Mission

This repository is the out-of-band proving ground for released Engineering
Runtime behavior. It installs a public release and exercises it as a user or
pipeline would. It has no runtime source; canonical behavior remains in
`../engineering-runtime`. Capability source content remains in
`../engineering-runtime-capabilities`.

The repository-level map is
[`../../engineering-runtime-workspace/CLAUDE.md`](../../engineering-runtime-workspace/CLAUDE.md).
Where that map and this file differ about this repository, this file wins.

## Required contract

### Public install

- Use `.github/actions/setup-runtime/` for workflow installation.
- The action runs the public installer from `engineering-runtime-releases`.
- The installer must verify the archive against `SHA256SUMS.txt` before use.
- Installation needs no credential. Do not pass `RUNTIME_GITHUB_TOKEN` or
  `github.token` merely to install Runtime.
- Pin `version: v0.6.0` when a test requires reproducibility; an empty version
  intentionally follows the latest public release.

### Fresh Runtime Home

All workflows use a disposable, gitignored Home:

```yaml
RUNTIME_HOME: ${{ github.workspace }}/runtime-home
RUNTIME_CONSUMER: ci
RUNTIME_ACTOR_NAME: github-actions/${{ github.workflow }}
RUNTIME_SESSION_ID: ${{ github.run_id }}-${{ github.run_attempt }}
```

After `runtime bootstrap`, assert:

- no `config.yaml`, `policy-config.yaml`, or `context.yaml` exists;
- `capabilities/` exists and is empty;
- release-owned contracts under `commands/` and `specs/` are available; and
- `runtime config validate` succeeds using compiled defaults.

Bootstrap does not seed user-owned governance, context, or capability
definitions. The Home `capabilities/` directory is an implicit,
non-authoritative compatibility cache. Never claim it is absent and never put
the public reference corpus there.

If a workflow needs configuration or policy, create/select the document
outside Home, set `RUNTIME_CONFIG_FILE` or `RUNTIME_POLICY_FILE` explicitly,
and keep the document's ownership clear. Do not point those variables at files
a fresh bootstrap does not create.

Runtime owns no context document. Pass organization, repository, project,
cluster, namespace, and similar values explicitly to the operation that uses
them, or let the native tool select its own context.

### Capability sources

This repository vendors no capability Markdown. Use
`.github/actions/setup-capabilities/` or an equivalent exact checkout. Every
run must capture:

- the repository or catalog identity;
- the resolved immutable commit/revision or digest; and
- the exact capability file path.

Validate and execute the exact Markdown path, for example:

```bash
runtime capability validate "${CAPABILITY_SOURCE_DIR}/github/github-repo-health.md"
runtime capability execute "${CAPABILITY_SOURCE_DIR}/github/github-repo-health.md" \
  --input "repository=${REPOSITORY}"
```

Leave `RUNTIME_CAPABILITIES_DIR` unset in these workflows, so a job resolves
only the capability it checked out and its evidence names that exact file. This
is scoping, not a claim about the variable: since Runtime 0.9.2 a directory named
there is an authoritative source reported as `capabilities-dir`. Only the unset
fallback inside Runtime Home is the non-authoritative cache.
A workflow that only needs one checked-out capability
should execute the exact path. A workflow testing named authoritative sources
must instead supply a `capabilities.sources` config document outside Home and
assert the source name, path, and immutable identity explicitly.

## Environment coverage

The canonical variable list is
`../engineering-runtime/commands/runtime_env_variables.txt` (also refreshed
into `runtime-home/commands/`).

| Variable | CI responsibility |
|---|---|
| `RUNTIME_HOME` | Prove Home is relocatable and isolated. |
| `RUNTIME_CONSUMER` | Set to `ci`; verify it appears in audit evidence. |
| `RUNTIME_ACTOR_NAME` | Name the workflow as a self-asserted searchable actor. |
| `RUNTIME_SESSION_ID` | Correlate every record to one run ID and attempt. |
| `RUNTIME_GITHUB_TOKEN` | Scope only to Auth Engine and GitHub provider steps. |
| `RUNTIME_CONFIG_FILE` | When used, select an explicit document outside Home. |
| `RUNTIME_POLICY_FILE` | When used, select an explicit governance document outside Home. |
| `RUNTIME_CAPABILITIES_DIR` | Left unset here to scope a job to its own checkout. Since 0.9.2 a named directory is an authoritative source (`capabilities-dir`); the unset Home fallback is the cache. |
| `CI` | GitHub Actions provides it; consumer fallback may be tested. |
| `KUBECONFIG` | Cover when a Kubernetes-auth workflow exists. |
| `USER` / `USERNAME` | Verify executor identity in audit records where relevant. |

Environment overrides form a closed, registered interface. Never infer a new
override by uppercasing a config key.

## CLI coverage

Smoke and specialized workflows should collectively cover:

- `runtime bootstrap`;
- `runtime config validate`;
- `runtime context show` with no Runtime-owned context document;
- `runtime auth status`, `login`, and `logout`;
- curated GitHub commands plus representative API/GraphQL operations;
- `runtime capability validate` and `execute` for auth-free files and GitHub;
- native-output `gh.repo.list`, structured File Engine dry-run/apply, and typed
  per-file JSON under policy-narrowed edit budgets;
- `runtime capability list` on an empty fresh Home; and
- `runtime audit tail`.

## Workflow rules

1. Prefer a workflow over a laptop-only demo.
2. When runtime install, bootstrap, Home layout, or environment semantics
   change, add or tighten a workflow that catches the break.
3. Use the installed binary only. New provider operations land in
   `../engineering-runtime` before this repository exercises them.
4. Validate every capability file a touched workflow executes.
5. Capture and print the capability source's immutable revision.
6. Capabilities receive orgs, repos, paths, and other targets through declared
   inputs. Never hardcode secrets or ambient target context.
7. Tokens are passed through `RUNTIME_GITHUB_TOKEN` only to steps that need
   GitHub authentication. Private source/target checkout may use the same
   secret explicitly.
8. Reusable capability definitions belong in
   `../engineering-runtime-capabilities`, never this repository.
9. Destructive packs—repository creation, protection changes, release cuts,
   DELETE operations—remain dispatch-only or local.
10. Do not commit `runtime-home/`, temporary source checkouts, credentials, or
    generated audit logs.

## Secrets and variables

| Kind | Name | Purpose |
|---|---|---|
| Secret | `RUNTIME_GITHUB_TOKEN` | Auth Engine, GitHub provider operations, and explicitly private source/target checkout. Not public runtime installation. |
| Variable (optional) | `GITHUB_ORG` | Passed explicitly to org-scoped operations; defaults to `github.repository_owner`. |

## Layout

| Path | Role |
|---|---|
| `.github/actions/setup-runtime/` | Public install, bootstrap, and fresh-Home assertions. |
| `.github/actions/setup-capabilities/` | Capability checkout with directory and resolved-revision outputs. |
| `.github/workflows/` | CI tests and `workflow_dispatch` demo entry points. |
| `engineering-runtime-capabilities/` | Ephemeral, gitignored source checkout used by Actions. |
| `runtime-home/` | Generated, gitignored Runtime state and empty compatibility cache. |

Architecture reminder:

```text
Intent → capability from a reviewed source
       → Engineering Runtime → engines → industry tools
```
