# Capability Specification (generic)

## Purpose

An Engineering Capability is a Markdown file that any human or AI can
author, describing a reusable engineering workflow. The runtime never
executes the Markdown — only the fenced `runtime` block inside it.
Everything else in the file exists for humans and AI to read; it has no
effect on execution.

> AI authors. The runtime is the compiler. A capability that doesn't
> validate doesn't run, no matter how well-reasoned it looks.

## Grammar

````markdown
```runtime
version: v1

inputs:
  <name>:
    description: <plain-English description>
    required: true|false

workflow:
  - command: <registered Runtime Command name>
    args: [<positional arg>, ...]

  - command: command.run
    binary: <allowed binary>
    args: [<positional arg>, ...]
```
````

- `version` — spec version this block was authored against. Required.
- `inputs` — named parameters the workflow needs beyond the active Runtime
  Context. Each is referenced as `${name}` inside any step's `args`.
- `workflow` — an ordered list of steps, executed in order. Each step is
  either:
  - a **registered Runtime Command** (`command: github.request`, etc. —
    see the per-provider spec for what's registered), or
  - a **raw Command Engine invocation** (`command: command.run`,
    `binary: <name>`) for any binary in `allowed_binaries`.

`args` are positional — exactly what `runtime run`/`runtime command run`
already accept on the CLI. This is the same grammar, just sequenced inside
a file instead of typed one at a time.

## Rules

1. **Never hardcode auth.** Every step already goes through the Auth
   Engine via the command/binary it names — a capability never embeds a
   token, credential, or auth flow of its own.
2. **Never assume a Runtime Context beyond what's active.** If a workflow
   genuinely needs a specific org/project/namespace beyond whatever context
   is currently active, declare it as an `input` — don't hardcode it into
   `args`.
3. **Ask the user if a required input is missing.** Don't invent a
   placeholder value and execute anyway — `runtime capability execute`
   will refuse to run with a missing required input; respect that at
   authoring time too.
4. **Only reference what the runtime can actually execute.** A step's
   `command` must be a registered Runtime Command, or `command.run` with a
   binary already in `allowed_binaries`. There is no "propose a new
   command" escape hatch — if the runtime can't run it today, the
   capability can't ask it to.
5. **Don't generate until required inputs are resolved.** A capability
   that references `${organization}` should not be treated as finished
   until the author knows where that value comes from.

## Validation

`runtime capability validate <path>` checks, and reports every problem it
finds (not just the first):

- `version` is present
- the workflow has at least one step
- every step declares a `command`
- every non-`command.run` step names a registered Runtime Command
- every `command.run` step names a binary in `allowed_binaries`

A capability that passes validation is not guaranteed to *succeed* at
execution (a step can still fail at runtime — missing credentials, a
denied `command_policy` rule, a network error) — validation only
guarantees the capability is well-formed and only references things this
runtime version can actually run.

## Output

`runtime capability execute <path> [--input key=value ...]` re-validates,
checks every `required` input was supplied, substitutes `${name}`
placeholders, then runs each step through the exact same
Bootstrap→Auth→Context→Policy→Engine→Audit lifecycle as any other Runtime
Command or `command run` invocation. Execution stops at the first failing
step. Every step's own audit record is what gets written — there is no
separate "capability" audit entry, since the capability itself contributes
no execution logic of its own.
