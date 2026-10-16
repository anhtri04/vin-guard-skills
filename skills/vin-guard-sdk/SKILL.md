---
name: vin-guard-sdk
description: Integrate Vin Guard's Python SDK into applications, including sync/async chat, gateway metadata, LangChain, control-plane reads, and error handling. Use for vin-guard-sdk or legacy ai-gateway-sdk and ai_gateway imports; not for managing resources with the CLI.
---

# Integrate the Vin Guard Python SDK

Read [integration recipes](references/integration.md) for the relevant client
and copy only the example needed by the application.

## Identify the installed contract first

At P-079 `origin/main` commit `28bba51` (2026-10-16), the distribution is
still `ai-gateway-sdk` 0.3.0 and the import package is `ai_gateway`.
`vin-guard-sdk` is the requested new distribution name; this snapshot does
not establish that it has been published. Inspect `sdk-python/pyproject.toml`,
the application's lockfile, or `python -m pip show vin-guard-sdk ai-gateway-sdk`
before choosing an install command. Keep imports as `ai_gateway` unless the
actual installed version documents a migration. Do not invent `vin_guard`.

Use the user's approved package/version or trusted build artifact. If the
renamed release is confirmed available, install `vin-guard-sdk` (optionally
`vin-guard-sdk[langchain]`). Otherwise a P-079 checkout supports
`python -m pip install -e ./sdk-python` from the repository root. Do not install
both distributions into one environment: they own the same import namespace.
The baseline requires Python 3.10+; the CLI is installed separately.

## Integrate with the right service

1. Determine whether the app needs synchronous or asynchronous inference,
   LangChain, or read-only management data. Inspect its existing dependency
   and secret configuration instead of replacing them with a new convention.
2. Inference requires a data-plane URL including `/v1`, a `gw_live_...` key,
   and an allowed model from that deployment. The checkout default is the
   hybrid data-plane IP (`http://136.81.141.176/v1`); prefer the deployment's
   documented data-plane URL when one is provided. Management reads require a
   separate `/api/v1` URL and an accepted OIDC JWT.
3. Use `GatewayClient` with `with`, or `AsyncGatewayClient` with `async with`.
   Pass secrets from the application's secret mechanism, never source code.
4. Start with non-streaming chat and supported `temperature`/`max_tokens`.
   Read `.content`, `.usage`, `.gateway`, and `.headers`; preserve the raw
   response only when needed, without logging sensitive payloads by default.
5. Handle `GatewayError` and transport subclasses. Do not add an automatic
   retry loop around chat POSTs: the baseline server does not deduplicate
   them even if `idempotency_key` is set.
6. Verify URL, auth headers, payload and error handling with an HTTPX mock
   transport. Run a real smoke only against the intended configured gateway
   within the user's task; label mock validation separately from live success.

## Capability checks that affect implementation

- The SDK has SSE helpers, but the chat backend returns JSON.
  `chat_completions(stream=True)` raises `ValueError` (use `stream_*`);
  a `stream_*` attempt against this backend raises
  `GatewayError(code="streaming_not_supported")`. Do not silently issue a
  second chat request or pretend buffered text is token SSE. (MCP upstream
  transport separately parses upstream SSE; that does not make chat stream.)
- The chat DTO accepts `stream`, `tools`, and `response_format` but the
  provider call consumes only `model/messages/temperature/max_tokens`
  (`ChatController.toProviderInput`); `stream` is only recorded into the
  usage event. `tool_choice`, `top_p`, `stop`, `user`, and `seed` remain
  unaccepted. Forwarding an option does not implement it.
- Tags, metadata, route policy and no-store are request headers. Their
  presence is not proof of server-side routing, storage or privacy enforcement.
- `ControlPlaneClient` only reads. Use management tooling for key creation,
  rotation or other mutations. Its `dev_subject` option is legacy and is not
  supported by the current verified-JWT control-plane auth.
- If the app already uses an OpenAI-compatible SDK successfully, preserve
  that approach when the task only needs a base URL/key change. Do not force
  migration to the wrapper unless its helpers are needed or requested.

When source is available, confirm changes against `sdk-python/src/ai_gateway/`,
`sdk-python/README.md`, and the backend chat DTO. For LangChain, also inspect
`sdk-python/src/ai_gateway/langchain.py`. Deployed behavior takes precedence
over this dated baseline when supported by evidence.
