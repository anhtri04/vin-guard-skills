# CLI workflows

Examples use `vig` (`@79ricons/vig-cli` 0.4.1 at P-079 `28bba51`). Only
pre-rebrand checkouts use the legacy `aigw` binary; resolve the executable
as described in SKILL.md before running them. Replace example URLs/IDs with
deployment values; do not execute every example as a setup checklist.

## Profile and authentication

```sh
vig --profile dev config set apiUrl 'https://control.example.com/api/v1'
vig --profile dev config set authority 'https://cognito-idp.<region>.amazonaws.com/<poolId>'
vig --profile dev config set clientId '<publicClientId>'
vig --profile dev auth login
vig --profile dev auth status --check
vig --profile dev whoami
vig --profile dev orgs list --all --output json
vig --profile dev --org org_example projects list --all --output json
```

The API URL preserves execute-api/stage prefixes and adds `/api/v1` when
absent. Floci URLs therefore need their complete execute-api prefix.
OIDC login uses Authorization Code + PKCE with a public client. The default
callback is `http://localhost:53682/callback` and must be registered exactly.
`auth login --redirect-port` and `VIG_REDIRECT_URI` can change it to another
registered loopback callback. `--no-browser` prints a URL and waits for that
callback; it is not a device-code flow and still needs browser interaction.

By default the CLI sends an ID token; issuer, audience and email claim must
match the authorizer/provisioning setup. The `hybrid-sg` preset
(`HYBRID_SG_PRESET` in `cli/src/client/config.ts`) supplies the fixed
hybrid-deployment apiUrl/authority/clientId, so `vig --profile hybrid-sg
auth login` (or the `default` profile) needs no manual `config set`;
explicit flags/env/repo config/stored profile always win over the preset.
Refresh credentials live in the OS keychain (`Bun.secrets`), scoped to
profile and issuer/client. No refresh token means later commands need
another supported credential source. If keychain support fails, explain the
error; do not silently enable plaintext storage.
`VIG_CREDENTIAL_STORE=file` is an explicit plaintext option, and `memory`
does not persist across processes. `auth logout` only removes local credentials.

Headless commands can use an externally supplied `VIG_TOKEN` secret and
`VIG_API_URL`, `VIG_ORG`, `VIG_PROJECT`. Do not print the token or put it
on the command line when an environment/secret mechanism is available.
Static tokens are not refreshed. There is no implemented `gw_mgmt_` token flow.
`auth status --check` checks/refreshes credential state; decoded claims are
not independent signature verification. Requests rely on the JWT authorizer.
`vig upgrade` checks the npm registry and upgrades the installed release
(`npm-global` reinstalls in place; standalone/source prints instructions).

## Configuration and context

Precedence: flags > environment > repo `vig.json` > global profile > defaults.
Global settings live in `%APPDATA%/vig/config.json` on Windows or
`${XDG_CONFIG_HOME:-~/.config}/vig/config.json` elsewhere;
`VIG_CONFIG_DIR` overrides the directory. `VIG_PROJECT_CONFIG` pins the repo
file; `-` disables discovery. No token or provider credential belongs there.

`config set` writes the global profile, which repo settings can override.
`init --profile dev --org org_example --project prj_example` creates
`vig.json` in the current directory; do not add `--force` merely to bypass an
existing file. `orgs use`/`projects use` persist context; switching org clears
the saved project. IDs are best for automation; names are resolved across all
pages and ambiguous matches fail. `--no-input` disables prompts/browser launch.

## Common operations

These are independent examples for an already configured tenant:

```sh
vig --org org_example projects create --name support-bot
vig --org org_example --project prj_example keys list --all --output json
vig --org org_example providers list --all --output json
vig providers test prov_example
vig providers models prov_example
vig --org org_example models list --all --output json
vig --org org_example routing list --all --output json
vig routing versions pol_example --all --output json
vig --org org_example usage --group-by model --output json
vig --org org_example requests list --all --output json
vig requests get req_example --output json
```

Other groups include `members`, `guardrails`, `limits`, `budgets`, `alerts`,
`insights` and `audit`; use their help for subcommands and scope requirements.
Use `providers add --credentials-file <protected-file>` for provider setup,
or `--credentials-file -` to consume JSON from stdin. Check the expected
credential shape for that provider. Policy draft creation/update consumes
`--file <policy.json>`; validate the schema from installed help or
`cli/src/validation/routing.ts`, not an invented policy structure.
`routing publish pol_example --policy-version 1` publishes an existing
definition version; draft creation alone does not publish it.

## One-time inference keys

```sh
vig --org org_example --project prj_example keys create --name support-bot --save-to '<new-protected-secret-path>'
vig keys get key_example --output json
```

`--save-to` writes the secret and emits metadata only; it refuses overwrite.
Pick a destination with suitable directory permissions outside tracked files.
The application can load the key from a secret manager/environment configured
from that destination. Do not read the secret back into the conversation.

Create/rotate default table mode prints the secret to stdout; JSON/YAML also
include it. `--secret-only` emits only the key. `--env-out` emits a POSIX export
line, not PowerShell syntax. These three destination flags (`--save-to`,
`--secret-only`, `--env-out`) are mutually exclusive. Avoid returning any of
those secret-bearing streams through an agent tool transcript.

For an authorized rotation, inspect `keys rotate --help`, choose an overlap
window and a new secret destination, then verify the returned key metadata
and update the application's credential as requested. Do not assume that
old/new key behavior or overlap is verified merely because rotation returned
success. If a create/rotate response is lost, do not blindly repeat the write.
Reads will not recover the secret. Reconcile state and the original operation
before deciding whether an explicit recovery rotation is needed.

## Pagination, dry-run and write concurrency

List output preserves `{data, pagination: {nextCursor, hasMore}}`. Default is
one page (20 items); use `--all` or explicit `--cursor` with `--limit 1..100`.
`--all` concatenates data and keeps the final pagination envelope. Table output
reports more pages on stderr. JSON is default when piped, but use
`--output json` explicitly in automation. Diagnostics go to stderr.

`--dry-run` previews API mutations with secret fields redacted. Name/context
resolution and version lookup can still make authenticated GETs. It does not
generate a secret or prove server validation. Config/auth/context commands
reject dry-run. `routing simulate` is a real server simulation using `signals`,
not a dry-run or live chat classification.

Versioned writes use `--if-match <row-version>`. Routing `activeVersion` and
definition `version` are not row concurrency versions. Model catalog reads
expose price versions, not the model-definition version required for writes;
do not substitute `price.version`.

Writes are not retried on network errors or 412. An authenticated refresh
source can refresh once on 401 and replay using the same idempotency key;
static tokens cannot, and 403 never refreshes. Each new invocation normally
generates a new idempotency key. A deliberate replay of the identical request
uses `--idempotency-key <original-key>` within the 24-hour retention window.
Do not reuse it with a changed body or assume every repeated command is safe.

## Known server mismatches at the recorded baseline

- Key RPM/TPM/monthly-budget nested-field parser FIXED at this baseline:
  CLI sends nested `{limits:{rpm,tpm,monthlyBudgetUsd}}` and the server
  `parseLimits(body["limits"])` reads flat inner fields (regression test in
  `controlplane/functions/projects-keys/src/keys.test.ts`). Still read back
  limits after writes; do not invent alternate payload keys.
- `providers test --model` is accepted but ignored server-side.
- Routing simulation consumes signals, not chat messages; its response lacks
  `classifier.model`. Simulation success is not proof of Jev integration.
- Usage is daily; hourly granularity and live tail are not implemented.
- Guardrails support create/test and project attachment, without update/publish
  routes. Management success is not proof of chat hot-path guardrail
  execution (chat `checkInput` remains allow-stub; MCP tool path enforces).
- Project deletion can return 409 when keys remain. Do not delete/revoke keys
  automatically just to make a project deletion succeed outside the request.

## Troubleshooting and exit codes

| Exit | Meaning | Next step |
|---|---|---|
| 0 | Success | Inspect the returned data; dry-run also returns 0 |
| 1 | Unexpected/network/cancel | Establish whether a write reached the server |
| 2 | Invalid arguments | Check installed subcommand help |
| 3 | Authentication | Check profile, token expiry, issuer/audience and login |
| 4 | Not found | Check resource ID, API URL and selected tenant |
| 5 | Forbidden | Check membership/role; do not retry as auth refresh |
| 6 | Validation (400/422) | Read RFC 7807 field errors and fix input |
| 7 | Precondition (412) | Re-read state/version and reconcile intent |
| 8 | Conflict (409) | Inspect the conflicting state/dependencies |
| 9 | Server error (5xx) | Preserve trace ID and inspect outcome before replay |

`--request-timeout` bounds HTTP/OIDC calls (default 30 seconds).
`auth login --timeout` is separately the callback wait time. Mock CLI tests
and help checks do not prove Cognito login, snapshot propagation, provider
reachability or end-to-end enforcement.
