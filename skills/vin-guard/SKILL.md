---
name: vin-guard
description: Explain Vin Guard architecture, tenant and credential boundaries, routing concepts, and implemented versus planned capabilities. Use for Vin Guard onboarding and deciding between its inference SDK, management CLI, and dashboard.
---

# Vin Guard overview

Vin Guard is this repository's multi-tenant AI Gateway. It presents an
OpenAI-compatible inference API and a separate management API. Older source
and documentation use AI Gateway, `aigw`, and `ai_gateway` names.

Read [architecture and capability boundaries](references/architecture.md)
for onboarding, routing explanations, and questions about feature support.
Distinguish product design from the behavior of the user's deployed version.
The reference records the `origin/main` baseline `28bba51` checked on
2026-10-16; inspect newer source or deployed-version evidence before treating
that snapshot as current.

## Choose the integration surface

| User's task | Surface | Credential |
|---|---|---|
| Send chat requests from an application | Python SDK / OpenAI-compatible `/v1` | Project inference key `gw_live_...` |
| Manage projects, keys, providers, policies | `vig` CLI / control plane `/api/v1` | OIDC/Cognito bearer JWT accepted by that deployment |
| Read usage and request history in Python | `ControlPlaneClient` / `/api/v1` | Control-plane JWT, never an inference key |
| Configure or inspect resources visually | Dashboard / control plane | OIDC login and server-side membership |

For implementation tasks, the companion `vin-guard-sdk` and `vig-cli` skills
provide detailed workflows when installed. They are not prerequisites for
explaining the architecture.

## Keep these distinctions explicit

- An organization owns membership, provider configuration and policies;
  projects group applications and their inference keys. Use actual `org_`,
  `prj_`, `key_`, `prov_`, `pol_`, `req_`, and `mcp_` identifiers returned by
  the API.
- Roles come from `/me` and server-side membership, not unverified JWT claims.
- The data plane and control plane can have different hosts. Obtain both URLs
  from deployment configuration; `gateway.example.com` is only a placeholder.
- The dashboard/CLI writes management state; the publisher asynchronously
  mirrors configuration into Valkey for the data plane. A successful write
  does not prove propagation or enforcement on an inference request.
- A feature in a design document, client option, or feature branch does not
  establish support in the deployed server. In particular, do not promise
  chat token streaming or chat hot-path PII filtering without checking the
  corresponding implementation: chat remains non-streaming JSON and
  `Guardrail.checkInput()` remains an allow-stub (see reference). MCP tool
  execution, RPM/TPM/budget enforcement, and async usage publication are
  implemented in this baseline, but still require deployment configuration
  (routing flag, pricing, credentials, Kinesis) — verify before promising
  behavior on a given deployment.

## Repository evidence

When working inside P-079, resolve these paths from the repository root:

- `docs/ai-gateway-design.md`: goals and architecture; includes planned flows.
- `docs/API_DESIGN.md` and `docs/API_CONTRACT_AUDIT.md`: intended wire contract
  and known deviations; reconcile them with handlers before coding.
- `docs/CONTROL_PLANE_SERVERLESS_DESIGN.md`: actual control-plane architecture.
- `sdk-python/README.md`, `sdk-python/src/ai_gateway/`: SDK behavior.
- `cli/README.md`, `cli/src/commands/`: CLI behavior and supported flags.
- `backend/src/main/java/com/ricons/aigateway/gateway/`: data-plane behavior.

Outside this repository, use the bundled reference as a dated starting point
and the installed SDK/CLI help plus the deployment's documentation as evidence.
Explain the relevant limitation and a supported path forward rather than
silently substituting a planned feature.
