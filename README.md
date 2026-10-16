# Vin Guard Agent Skills

Agent Skills for [Vin Guard](https://github.com/AI20K-Build-Phase-Cohort-4/P-079)
(the multi-tenant AI Gateway in `P-079`). Follows the
[Agent Skills spec](https://agentskills.io).

| Skill | What it does |
|---|---|
| `vin-guard` | Architecture, tenancy/credential boundaries, routing concepts, implemented-vs-planned capabilities |
| `vin-guard-sdk` | Python SDK integration (sync/async chat, LangChain, control-plane reads) |
| `vig-cli` | `vig` CLI operations (auth, projects, keys, providers, policies, usage) |

## Install (1 shot)

```sh
# All skills, auto-detect your coding agent
npx skills add <owner>/<repo>

# Specific skills
npx skills add <owner>/<repo> --skill vin-guard --skill vin-guard-sdk --skill vig-cli

# OpenCode, global, non-interactive
npx skills add <owner>/<repo> --skill vin-guard --skill vin-guard-sdk --skill vig-cli -a opencode -g -y
```

Replace `<owner>/<repo>` with this repo's GitHub path after pushing.

Source snapshot: P-079 `origin/main` at `28bba51` (2026-10-16).
