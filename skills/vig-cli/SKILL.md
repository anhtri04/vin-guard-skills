---
name: vig-cli
description: Use the Vin Guard vig CLI to configure authentication and tenant context, manage projects, keys, providers, policies, and MCP servers/toolsets, or inspect usage and requests. Also covers the legacy aigw binary; use for control-plane terminal operations, not inference chat calls.
---

# Operate the Vin Guard CLI

Read [CLI workflows](references/workflows.md) for configuration, commands,
secrets, pagination, versioned writes and troubleshooting.

## Resolve the executable and environment

At P-079 `feat/cli-mcp-update` commit `68c2c28` (2026-10-17, PR #49 to
`origin/main`; previous baseline `28bba51` on `origin/main`), the executable
is `vig` (`@79ricons/vig-cli` 0.5.1, `bin: {vig: dist/vig.js}`). The MCP
commands (`mcp-servers`, `mcp-toolsets`, `projects attach-mcp-servers`,
`keys --mcp-*`) exist only from `68c2c28` onward; re-pin this baseline to an
`origin/main` commit once PR #49 merges. Check executable availability,
`--version` and `--help` before issuing commands.
Only pre-rebrand checkouts use the legacy `aigw` binary; if that is all the
user has, replace the leading `vig` in the examples with `aigw` and explain
the name difference.

From a trusted P-079 source checkout, `cli/` supports `bun run start --help`
and `bun run start <resource> <verb> ...`; dependencies require Bun 1.4.2+
and `bun install --frozen-lockfile`. A native release binary needs neither
Bun nor Node. `vig upgrade` can self-update an installed release (npm-global
reinstalls; standalone/source prints instructions). Use the user's
installed/approved binary or trusted release artifact, not an assumed npm
package. Python SDK installation does not install this CLI.

The CLI manages `/api/v1` with OIDC/Cognito JWTs, not `gw_live_` inference
keys. Settings are `VIG_*` (`VIG_API_URL/AUTHORITY/CLIENT_ID/ORG/PROJECT/
TOKEN/PROFILE/CONFIG_DIR/PROJECT_CONFIG`), repo `vig.json`, and the `vig`
config directory. A `hybrid-sg` preset supplies the fixed hybrid-deployment
apiUrl/authority/clientId when no explicit config wins, so a fresh machine
can `vig auth login` with no manual `config set`. Do not use legacy `AIGW_*`
or `aigw.json` names on this baseline.

## Execute the requested workflow

1. Resolve the configured profile, management URL, organization and project.
   Use `config list`, `auth status`, `whoami`, and scoped reads as needed.
   Do not change active context merely to inspect another tenant: explicit
   `--profile`, `--org`, `--project` flags can scope one command.
2. Inspect `<resource> <verb> --help` for the installed binary. The command
   tree is resource-first: `vig projects create --name ...`, not
   `vig create project`. There is no inference `vig chat` command here.
3. For scripts, use explicit resource IDs, `--no-input`, `--output json`,
   and `--all` when the request needs all records. Preserve list envelopes.
4. Apply only the requested changes. For a preview/review request use
   `--dry-run`; it can make supporting reads but sends no API mutation.
   A dry-run is not mandatory when the user already authorized the change.
5. Before key creation/rotation, choose a secret destination that will not
   expose the returned secret in tool output. Prefer `--save-to` with a new
   protected path outside version control; report metadata and the destination.
6. Use `--yes` for an already-authorized destructive operation in automation.
   Do not treat it as permission to expand the task. On an uncertain write
   outcome or 412, inspect state and reconcile intent before another write.
7. Verify with relevant reads; distinguish persisted management state from
   propagation/enforcement in the data plane. Report actual resource IDs and
   remaining deployment limitations.

When repository source is available, `cli/src/commands/` and
`cli/src/client/` define actual behavior. Read `cli/README.md` and
`docs/API_CONTRACT_AUDIT.md` before promising behavior affected by a known
server mismatch. The bundled reference is a dated baseline, not a substitute
for checking a newer installed release.
