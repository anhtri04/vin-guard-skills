# Architecture and capability boundaries

Source snapshot: P-079 `origin/main` at `28bba51`, checked 2026-10-16.
Schema is `V1–V10` (`V7__mcp.sql`, `V8__mcp_usage_index.sql`,
`V9__request_error.sql`, `V10__catalog_sync_source.sql`).

## Request and configuration flows

```text
Application -> SDK -> Java data plane /v1 -> provider adapter -> LLM
                         |
                         +-> Valkey: key auth, config, limits, reservations

Dashboard / CLI -> serverless control plane /api/v1 -> PostgreSQL
                                                       |
                                                   outbox
                                                       |
                                                   publisher -> Valkey
```

Java uses Spring MVC with virtual threads and blocking I/O. The control plane
uses Node/TypeScript Lambda handlers; the React dashboard calls only that
control plane. The PostgreSQL schema is shared, with Flyway migrations owned
by `backend/` and mirrored in `controlplane/packages/common/src/schema.ts`.
The hot path reads Valkey rather than calling the management API per request.

Inference authentication uses a project key. Management authentication uses
verified API Gateway authorizer claims and membership/RBAC. The CLI defaults
to a Cognito ID token because provisioning needs an email claim; the issuer,
audience and token type must match the deployment. The legacy SDK `dev_subject`
option sends `x-dev-sub`, but current management auth does not accept that as
an authentication bypass.

Provider credentials belong to control-plane provider configuration. They
are different from project inference keys and user JWTs. Key creation/rotation
returns the secret once; list/get operations expose metadata only.

## Routing vocabulary and limits of the baseline

The design accepts `auto`, `policy/<name>`, explicit `<provider>/<model>` and
aliases. Jev/TypeSafe classifies request difficulty for policy selection; it
is not the LLM producing the answer. The flow adds guardrail checks,
selection/fallback, and asynchronous usage publication.

At the recorded baseline, routing is implemented via `RoutingPlanner` (there
is no `RoutingEngine` class): model-mode binding → policy load → normalize →
`classify` → `EligibilityFilter` re-filter → ordered targets, with
`fallbackTier` retry else 502 `no_available_target`. `RoutingClassifier`
returns `Optional<ClassificationResult>` (`empty` = abstain, never null):
`RuleClassifier` evaluates deterministic rules first, `HybridClassifier`
(`@Primary`) is rule-first then optionally Jev via `JevClassifier` (TypeSafe
HTTP `POST {baseUrl}/v1/systemone`, fail-soft to `Optional.empty()`), with
`JevCache`, confidence floor (`gateway.classifier.min-confidence`) and
`escalateIfProbability`. Routing is gated by `gateway.routing.enabled`
(default `false` = direct target); `shadowMode` executes direct but reports.
Prefer a configured, allowed explicit model when verifying an integration; a
successful `model="auto"` request alone does not prove classifier-based
routing.

| Capability | Evidence at this baseline | Consequence for an integrating agent |
|---|---|---|
| Chat, models, health/readiness | Java controllers and SDK clients exist | Verify deployment configuration and allowed models |
| Token SSE streaming | Backend returns JSON; SDK can parse SSE | Use non-streaming; streaming attempts report `streaming_not_supported` (`chat_completions(stream=True)` raises `ValueError`; `stream_*` raises `GatewayError`) |
| Chat `tools`, `response_format`, `stream` fields | `ChatDtos.Request` accepts `stream/tools/response_format` but `toProviderInput` forwards only `model/messages/temperature/maxTokens`; `stream` is only recorded into the usage event | Do not build a chat tool loop or strict schema guarantee on forwarding alone; `tool_choice/top_p/stop/user/seed` remain unaccepted |
| RPM, TPM, budget enforcement | Implemented: `LimitService.checkRpm` (429 `rate_limit_exceeded`), `reserveUsage`/finalize with unwind, Bucket4j+Lettuce dual-bucket RPM, Valkey Lua TPM/budget reserve, fail-closed 503 | Check actual key/project config, pricing and deployed enforcement; read back limits |
| Key-limit management | Nested-field parser mismatch FIXED: CLI sends nested `{limits:{rpm,tpm,monthlyBudgetUsd}}`, server `parseLimits(body["limits"])` reads flat inner fields (regression test in `controlplane/functions/projects-keys/src/keys.test.ts`) | Still read back limits after writes; do not invent alternate payload keys |
| Guardrail management | Control-plane create/test and project attachment exist; data plane has real `AllowGuardrail.evaluate` + `ValkeyGuardrailRepository` + `McpGuardrails`, but chat `GatewayChatService` still calls `guardrail.checkInput(request); // TODO: real rules (currently allow)` | Management resources do not prove chat hot-path PII enforcement; MCP tool path is the enforced path |
| Usage and traces | Implemented: `UsagePublisher.publish()` is a bounded non-blocking queue → background batch → Kinesis `aigw-usage` (`AWS_KINESIS_ENABLED=false` default logs + counts); `UsageEvent` frozen v1 contract (`chat\|mcp_tool`); control-plane reads + daily rollups exist | Empty/stale data can be a pipeline/config issue (`enabled` flag, stream, Floci endpoint); daily rollups are not live metrics |
| MCP | Implemented: `POST /v1/mcp{,/,/{alias},/toolsets/{name}}` (`McpController`: `initialize/tools/list/tools/call`, stateful Valkey session + distributed lock, `GET→405`), `StreamableHttpMcpClient` parses upstream SSE, `McpToolExecutor/Catalog/Router/Admission/Guardrails/CostCalculator`, dashboard `Mcp.tsx/McpDetail.tsx`, control-plane `mcp` function, `PUT /projects/{id}/mcp-servers`, key scopes `mcpServers/mcpToolsets` | Verify server/toolset attachment, scopes, credentials and upstream reachability; chat streaming rules do not apply to MCP transport |
| A2A / semantic caching | Outside implemented baseline | Treat design/branch work separately |

Check `GatewayChatService.java`, `web/ChatDtos.java`, `routing/RoutingPlanner.java`,
`classifier/RoutingClassifier.java` (`Optional`, not null),
`classifier/HybridClassifier.java`, `classifier/JevClassifier.java`,
`guardrail/Guardrail.java`, `guardrail/AllowGuardrail.java`, and
`usage/UsagePublisher.java` under the gateway source directory to refresh this
table. For management limitations, check `cli/README.md` and the relevant
control-plane handler rather than preserving a stale limitation indefinitely.

The dashboard at this baseline includes an MCP registry (`Mcp.tsx`:
server/toolset CRUD + project attachment; `McpDetail.tsx`:
overview/tools/versions/test/usage tabs), a shared org-catalog model picker
(`ModelPicker.tsx`, qualified `type/modelId` from org connections ∩ catalog),
and a guided setup journey (`SetupJourney.tsx`, provider → models → project
→ key → routing → request). These are management surfaces; they do not prove
data-plane enforcement.

## Management contracts

- Lists use `{data, pagination: {nextCursor, hasMore}}`; default page size is
  20, maximum 100. A first-page result is not necessarily the full tenant.
- Mutations use `Idempotency-Key` (24-hour retention). Versioned writes use
  `If-Match`; 412 means the version changed. Re-read and reconcile intent.
- Control-plane functions are `catalog, governance, identity, ingestor,
  janitor, mcp, policies, price-sync, projects-keys, providers, publisher,
  reads, snapshot-internal`. MCP IDs use the `mcp_…` prefix.
- A policy row's `version`, its `activeVersion`, and a definition's `version`
  serve different purposes. `If-Match` uses the row concurrency version.
- Management errors use RFC 7807; inference uses OpenAI-shaped errors.
  Preserve status/code/request ID for troubleshooting without dumping secrets
  or full request/response bodies.
- Management idempotency does not imply chat POST deduplication. At this
  baseline, retrying a chat can produce another billable provider request.

## Diagnose at the correct boundary

For a newly created key that cannot chat, distinguish tenant/scopes/revocation,
snapshot publication, model/provider setup, and limiter/pricing failures.
Readiness success alone proves neither provider reachability nor permission
to use a model. For an empty usage dashboard, first establish whether request
events are emitted and ingested in that deployment. The `data-pipeline/`
offline evaluator is not proof of a live production analytics pipeline.
