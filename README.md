# engineering-runtime-samples
Reference implementations and sample projects demonstrating deterministic engineering workflows with the Engineering Runtime.

## CI (`.github/workflows/runtime-ci.yaml`)

On every push/PR to `main` (and manual dispatch), the workflow:

1. Downloads the **latest** `runtime` binary release from
   [`kishore-gutta/engineering-runtime-releases`](https://github.com/kishore-gutta/engineering-runtime-releases)
   (linux-amd64 archive, checksum-verified against `SHA256SUMS.txt`) and puts
   it on `PATH` — this repo never builds the runtime itself.
2. Points the runtime at this repo's own `runtime-commands/config.yaml` /
   `policy-config.yaml` via `RUNTIME_CONFIG_FILE` / `RUNTIME_POLICY_FILE`
   (the documented CI/CD override) instead of relying on bootstrap-seeded
   defaults.
3. Runs `runtime config validate` (config diagnostics) and
   `runtime capability validate` against both `examples/capabilities/*.md`
   files.
4. Runs the `auth_commands.txt` / `github_commands.txt` cheatsheets:
   `auth status` → `auth login github` → a handful of context-free
   `runtime github ...` convenience commands (`user get`,
   `organizations list`, `notifications list`) → `auth logout github`.
5. Prints the audit trail (`runtime audit tail`) so every command's
   success/failure/denial is visible in the job log.

Requires one repository secret: `RUNTIME_GITHUB_TOKEN` — a GitHub PAT with
enough scope for the calls above (`repo`, `read:org`, `notifications`). No
other secret is needed; `GH_TOKEN` for the release download step reuses the
built-in `${{ secrets.GITHUB_TOKEN }}`.
