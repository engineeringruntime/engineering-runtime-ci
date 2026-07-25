# Sample capabilities

Used by `.github/workflows/*` via **full paths**. Do not set
`RUNTIME_CAPABILITIES_DIR` to this folder — every `runtime` command
re-runs bootstrap and would seed the binary's full capability tree here.

See the table in the repo README for which workflow uses which file.
