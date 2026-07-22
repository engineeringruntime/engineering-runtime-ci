# Capability Specification — GitHub

Extends `specs/capability-spec.md`. Read that first — this file only lists
what's actually available for GitHub-flavored workflow steps.

## Supported Commands

| Command                              | Kind              | Args |
|----------------------------------------|-------------------|------|
| `github.user.get`                     | Runtime Command   | none (fixed `GET /user`) |
| `github.organizations.list`           | Runtime Command   | none (fixed `GET /user/orgs`) |
| `github.notifications.list`           | Runtime Command   | none (fixed `GET /notifications`) |
| `github.repositories.list`            | Runtime Command   | none (fixed `GET /user/repos`) |
| `github.repositories.list_for_org`    | Runtime Command   | none (fixed `GET /orgs/{org}/repos`; org from Runtime Context) |
| `github.repositories.create`          | Runtime Command   | `[key=value ...]` -> JSON body (fixed `POST /user/repos`), e.g. `name=foo`, `private=true` |
| `github.issues.list_for_org`          | Runtime Command   | none (fixed `GET /orgs/{org}/issues`; org from Runtime Context) |
| `github.teams.list`                   | Runtime Command   | none (fixed `GET /orgs/{org}/teams`; org from Runtime Context) |
| `github.request`                      | Runtime Command   | `[METHOD, path, key=value ...]` — query params for GET/DELETE, JSON body for POST/PUT/PATCH |
| `command.run` with `binary: gh`       | Command Engine    | any `gh` subcommand, forwarded verbatim |

Every `github.*` Runtime Command requires the `github` Auth Engine provider
— `RUNTIME_GITHUB_TOKEN` must be set and valid. `command.run gh ...` also
authenticates as `github` (see `command_providers` in `config.yaml`).

Commands whose path contains `{org}` resolve it from the active Runtime
Context's `github.organization` — no need to declare it as a capability
input unless the workflow needs to target a *different* org than the
active context.

## Typical inputs

- `organization` — when a workflow needs a specific GitHub org's data
  (`github.request GET /orgs/${organization}/repos`), declare it as a
  required input rather than hardcoding an org name.

## Example

````markdown
```runtime
version: v1

inputs:
  organization:
    description: GitHub organization to inspect
    required: true

workflow:
  - command: github.request
    args: [GET, "/orgs/${organization}/repos"]

  - command: command.run
    binary: gh
    args: [pr, list]
```
````
