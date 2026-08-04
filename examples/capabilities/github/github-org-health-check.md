# Organization health check

Chains four read-only operations into a single snapshot of the
active Runtime Context's GitHub organization: the orgs the token
belongs to, the org's repositories, its teams, and issues assigned to
the authenticated user across it. No inputs — every step resolves
`{org}` from the active Runtime Context. Requires
`RUNTIME_GITHUB_TOKEN` to be exported.

**Requires a real GitHub organization.** `repo list` resolves to
`/orgs/{org}/repos` and `team list` has no user-account equivalent, so
pointing Runtime Context at a personal account returns `404 Not Found`
on step 2. The `github-org-health.yaml` workflow resolves an org via
`github org list` first and skips when the token belongs to none.

Run with:

```
runtime capability validate github/github-org-health-check.md
runtime capability execute github/github-org-health-check.md
```

```runtime
version: v1

workflow:
  - provider: github
    args: [org, list]

  - provider: github
    args: [repo, list]

  - provider: github
    args: [team, list]

  - provider: github
    args: [issue, list]
```
