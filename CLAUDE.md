# Engineering Runtime Samples

## What this repo is

Per the workspace map (`../CLAUDE.md`): the intended home for **standalone
sample/reference projects built with the runtime** — demonstrations of
deterministic engineering workflows, not the runtime's source code and not
the canonical capability library.

This repo does not execute anything itself. It has no `internal/`, no
`main.go`. Everything runnable here (`runtime capability validate|execute`,
`runtime github ...`, `runtime command run ...`) is executed by the
`runtime` binary built from the sibling `../engineering-runtime` repo —
this repo only supplies the inputs (capability Markdown files, config
samples, cheatsheets).

## What's here today

- `runtime-commands/*.txt` — copy-paste CLI cheatsheets, one file per
  command area (`github_commands.txt`, `files_commands.txt`,
  `command_commands.txt`, `capability_commands.txt`, `auth_commands.txt`,
  `config_commands.txt`), plus sample `config.yaml` /
  `policy-config.yaml` to copy into a Runtime Home.
- `specs/` — the published capability-authoring contracts
  (`capability-spec.md` + `specs/github/capability-spec-github.md`).
- `examples/capabilities/` — two worked capability files
  (`notes-roundtrip.md`, `github-repositories.md`).

**All of the above are copies seeded from `../engineering-runtime`**
(`runtime-commands/`, `specs/`, `examples/capabilities/` there are the
source of truth). This repo does not maintain its own divergent copy of
the grammar or the Runtime Command registry — if a spec or cheatsheet here
and the one in `../engineering-runtime` ever disagree, `../engineering-runtime`
wins; update this repo's copy to match, not the other way around.

## Conventions for adding future sample capabilities/projects

1. **Read `specs/capability-spec.md` and the relevant per-provider spec
   before authoring anything new.** They are the same files seeded into a
   user's Runtime Home — not a paraphrase. A capability that doesn't
   validate doesn't run, no matter how well-reasoned it looks.
2. **Only reference what `../engineering-runtime` can actually execute.**
   A workflow step's `command` must be a registered Runtime Command
   (`../engineering-runtime/internal/commands/commands.go`) or
   `command.run` with a binary already in `allowed_binaries`
   (`runtime-commands/policy-config.yaml`). There is no "propose a new
   command" escape hatch from a sample — if the runtime can't run it
   today, add the Runtime Command/binary there first, in a separate
   change, before writing a sample that depends on it.
3. **Validate before checking anything in**:
   `runtime capability validate <path>` against every new capability file,
   reporting every problem at once — don't hand-verify by inspection alone.
4. **Never hardcode auth, an org/project/namespace, or any other
   Runtime-Context-shaped value.** Declare it as an `input` (see
   `capability-spec.md`'s "Rules"). Samples exist to demonstrate the
   grammar correctly — a sample that hardcodes what should be an input
   teaches the wrong pattern.
5. **New standalone sample projects get their own top-level folder**
   (e.g. a future `ci-pipeline-demo/`, `k8s-rollout-demo/`), each with its
   own short `README.md` explaining what it demonstrates and how to run
   it. Don't grow `examples/capabilities/` into a dumping ground for
   unrelated demos — reserve it for small, generically useful reference
   capabilities the way `notes-roundtrip.md`/`github-repositories.md`
   already are.
6. **Reusable, general-purpose capabilities belong in
   `../engineering-runtime-capabilities`, not here.** This repo is for
   *demonstrations* (a sample project showing the runtime doing something
   end-to-end); that repo is the intended home for the capability library
   itself. If a capability written here turns out to be generically
   useful beyond illustrating one sample, move it there instead of
   duplicating it in both places.
7. **Keep the copied reference material in sync deliberately, not
   automatically.** When `../engineering-runtime` changes its
   `runtime-commands/`, `specs/`, or `examples/capabilities/`, re-copy the
   updated files here in the same change that depends on the update —
   don't let this repo's copies silently drift stale.
8. **Never commit secrets.** `runtime-commands/config.yaml` is a sample —
   it names *which* environment variable holds a token
   (`RUNTIME_GITHUB_TOKEN`) but never a real value. Keep it that way for
   any new provider samples.

## Architecture reminder

```
Intent → Capability → Engineering Runtime → Engines → Industry Tools
```

The runtime remains provider agnostic; capabilities abstract providers.
Nothing in this repo should try to be either of those things itself — it
only shows how to use them.
