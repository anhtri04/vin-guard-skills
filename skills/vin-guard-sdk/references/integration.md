# Python integration recipes

These examples use the `ai_gateway` API verified at P-079 `28bba51`.
The distribution is still `ai-gateway-sdk` 0.3.0 (`import ai_gateway`).
Environment variable names below are application conventions: pass their
values explicitly; the SDK does not automatically discover them. The client
source default `base_url` is the hybrid data-plane IP
(`http://136.81.141.176/v1`); prefer the deployment's documented data-plane
URL when one is provided.

## Synchronous chat

```python
import os
import httpx
from ai_gateway import GatewayClient, GatewayError, GatewayTransportError

with GatewayClient(
    api_key=os.environ["GATEWAY_API_KEY"],
    base_url=os.environ["GATEWAY_BASE_URL"],  # data-plane URL ending in /v1
    timeout=httpx.Timeout(30.0, connect=5.0),
) as client:
    try:
        response = client.chat_completions(
            model=os.environ["GATEWAY_MODEL"],  # configured, allowed model
            messages=[{"role": "user", "content": "Explain gateway routing briefly."}],
            max_tokens=200,
            tags={"team": "support"},
            metadata={"ticket": "t-example"},
        )
    except GatewayTransportError as exc:
        # Outcome may be unknown. Let the caller decide whether to retry.
        print("Transport failure:", type(exc).__name__, exc.status)
        raise
    except GatewayError as exc:
        # Avoid dumping exc.payload, which can contain sensitive server data.
        print("Gateway failure:", exc.status, exc.code, exc.request_id)
        raise
    else:
        print(response.content)
        print(response.gateway.request_id, response.usage.total_tokens)
```

`GatewayTransportError` subclasses include `GatewayConnectionError` and
`GatewayTimeoutError`; status is 0 when there is no HTTP response. Timeout
errors have a `phase`. Catch transport errors before the broader `GatewayError`.
For HTTP errors inspect `.status`, `.code`, `.request_id`, `.retry_after`.
Resolve 401 credentials and 403 permission/scope issues rather than retrying
blindly. For 429, inspect the error and retry hint; rate exhaustion, token
limits and budgets need different remedies. Even a transient error does not
prove a previous chat POST was unprocessed.

`response.gateway` includes request ID, tier, policy, confidence and cost USD;
fields can be absent/defaulted by the server. Do not interpret missing
telemetry defaulted to zero as proof of zero billing. `.raw` preserves the
body; `.choices` preserves tool calls and additional message structure.
Tool-only responses may have an empty `.content` string.

## Async chat

```python
import asyncio
import os
from ai_gateway import AsyncGatewayClient

async def main():
    async with AsyncGatewayClient(
        api_key=os.environ["GATEWAY_API_KEY"],
        base_url=os.environ["GATEWAY_BASE_URL"],
    ) as client:
        result = await client.chat_completions(
            model=os.environ["GATEWAY_MODEL"],
            messages=[{"role": "user", "content": "Hello"}],
            max_tokens=100,
        )
        print(result.content)

asyncio.run(main())
```

In an existing event loop, await `main()` or integrate the client's lifecycle
into the application instead of nesting `asyncio.run`. Reuse a client for the
application's appropriate lifetime and close it on shutdown.

## Gateway-specific arguments

| Keyword | Wire header | Meaning for the client |
|---|---|---|
| `tags={"team": "support"}` | `X-Gateway-Tags` | Encodes comma-separated key/value pairs |
| `metadata={"ticket": "t-example"}` | `X-Gateway-Metadata` | JSON, at most 4096 UTF-8 bytes |
| `route_policy="policy-name"` | `X-Gateway-Route-Policy` | Sends a policy hint; check server support |
| `no_store=True` | `X-Gateway-No-Store` | Sends a hint; does not prove non-retention |
| `idempotency_key="operation-id"` | `Idempotency-Key` | No deduplication guarantee in the baseline hot path |

`list_models()` returns `ModelEntry` objects. `health()` and `ready()` probe
the service root without an API key, preserving proxy prefixes:
`/gateway/v1` becomes `/gateway/healthz` or `/gateway/readyz`.

## LangChain, only when the application needs it

Use the confirmed distribution's `[langchain]` extra; in this checkout:
`python -m pip install -e './sdk-python[langchain]'`.

```python
import os
from ai_gateway.langchain import ChatGateway
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "Answer briefly."), ("human", "{question}"),
])
with ChatGateway(
    api_key=os.environ["GATEWAY_API_KEY"],
    base_url=os.environ["GATEWAY_BASE_URL"],
    model=os.environ["GATEWAY_MODEL"],
    max_tokens=200,
    gateway_tags={"team": "support"},
) as model:
    result = (prompt | model).invoke({"question": "What is a gateway?"})
    print(result.content)
```

`invoke`/`ainvoke` and `batch`/`abatch` are supported. `stream`/`astream` use
LangChain's buffered full-result fallback, not server token streaming.
`gateway_tags`/`gateway_metadata` send gateway headers; LangChain `config`
tags/metadata control tracing. Per-call `.bind(...)` supports gateway options.
Do not bind a fixed idempotency key to a reusable model.

Advanced tools/structured-output options are rejected by default. Only use
`allow_advanced_options=True` after proving the deployed backend supports
them; it only enables forwarding and does not execute tools. If using both
sync and async methods, finish with `await model.aclose()`. Injected SDK
clients remain owned by the caller.

## Read management data

```python
import os
from ai_gateway import ControlPlaneClient

with ControlPlaneClient(
    base_url=os.environ["CONTROL_PLANE_BASE_URL"],  # includes /api/v1
    access_token=os.environ["CONTROL_PLANE_ACCESS_TOKEN"],
) as control:
    for project in control.iter_items(
        control.list_projects, os.environ["VIN_GUARD_ORG_ID"], limit=100,
    ):
        print(project["id"], project["name"])
```

Despite the parameter's name, supply a bearer JWT accepted by the deployment
(the CLI commonly uses a Cognito ID token); never a `gw_live_` inference key.
A callable can return a fresh token per request; this client does not log in
or refresh on its own.

List methods return `Page(data, next_cursor, has_more)`. `iter_items` traverses
pages and rejects missing/repeated cursors. Read methods include `me`, orgs,
projects, API-key metadata, providers, models, routing policies/versions,
requests/detail, usage, routing insights and audit logs. Query kwargs use wire
names such as `projectId` and `minLatencyMs`; `get_usage` instead has explicit
`from_`, `to`, `group_by` arguments. Usage is daily, without hourly/live-tail
support. Preserve nullable fields rather than assuming every metric exists.

## Verification

Both inference clients and `ControlPlaneClient` accept an HTTPX transport.
Use `httpx.MockTransport` in application tests to observe the resulting path,
bearer header, chat body and metadata headers, and exercise representative
HTTP and transport failures without spending provider tokens. A mocked
success cannot verify snapshots, provider credentials or live enforcement.
For a live smoke, use an allowed configured model and a small `max_tokens`;
record success and request ID without printing the key or management token.
