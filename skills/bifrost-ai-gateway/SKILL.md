---
name: bifrost-ai-gateway
description: Set up Bifrost, the open-source AI gateway by Maxim AI, as one self-hosted gateway for LLM calls, MCP tool calls, and A2A agent traffic. Installs it with npx or Docker, writes one config.json with model providers, MCP servers, and agents, issues virtual keys that carry budgets, rate limits, tool allowlists, and agent grants, and verifies all three endpoints. Use when the user wants an AI gateway or AI control plane, a self-hosted OpenAI-compatible gateway, one place to route and govern model, MCP, and agent traffic, AI cost and access control across teams, or help choosing between an LLM gateway, MCP gateway, and agent gateway. Points to the focused bifrost-llm-gateway, bifrost-mcp-gateway, and bifrost-agent-gateway skills.
compatibility: Runs Bifrost with npx (Node.js) or Docker and needs network access to model providers, MCP servers, and agents.
metadata:
  author: Maxim AI
  version: "1.0"
---

# Bifrost AI gateway

Bifrost is an open-source AI gateway (Apache 2.0) built by Maxim AI. One Bifrost deployment serves three kinds of AI traffic on one port, with one set of virtual keys:

```
                      +--------------------- Bifrost ---------------------+
 apps, SDKs   ------> | /v1/... , /openai, /anthropic   LLM gateway       | ---> model providers
 MCP clients  ------> | /mcp                            MCP gateway       | ---> MCP servers
 A2A clients  ------> | /agents/a2a/<name>/...          Agent Gateway     | ---> A2A agents
                      |   virtual keys: budgets, rate limits, tool         |
                      |   allowlists, agent grants; logs for all three     |
                      +----------------------------------------------------+
```

This skill sets up all three at a working baseline. For depth on one kind of traffic, install the focused skill:

| Traffic | Focused skill | Install |
|---|---|---|
| Model calls: one OpenAI-compatible API, fallbacks, load balancing, budgets | `bifrost-llm-gateway` | `npx skills add ai-gateways/llm-gateway` |
| MCP tool calls: one `/mcp` endpoint, tool filtering, upstream auth, Code Mode | `bifrost-mcp-gateway` | `npx skills add ai-gateways/mcp-gateway` |
| A2A agent calls: agent registration, agent grants, transports, push, Agent Logs | `bifrost-agent-gateway` | `npx skills add ai-gateways/agent-gateway` |

## Which to set up first

- Apps call model APIs directly with provider keys: start with the LLM gateway.
- Developers or agents use MCP servers from Claude Code, Cursor, or Claude Desktop: add the MCP gateway.
- Services call other agents over A2A: add the Agent Gateway, then route those agents' own model and tool calls through the first two.

## Steps

### 1. Write config.json

Create `config.json` in the Bifrost app directory (`~/.config/bifrost` for npx, or the folder mounted at `/app/data` for Docker). Drop the blocks you do not need yet.

```json
{
  "$schema": "https://www.getbifrost.ai/schema",
  "client": {
    "enforce_auth_on_inference": true,
    "a2a_external_client_url": "http://localhost:8080"
  },
  "providers": {
    "openai": {
      "keys": [{"name": "openai-key-1", "value": "env.OPENAI_API_KEY", "models": ["*"], "weight": 1.0}]
    },
    "anthropic": {
      "keys": [{"name": "anthropic-key-1", "value": "env.ANTHROPIC_API_KEY", "models": ["*"], "weight": 1.0}]
    }
  },
  "mcp": {
    "client_configs": [
      {
        "name": "docs_search",
        "connection_type": "http",
        "connection_string": "env.DOCS_MCP_URL",
        "auth_type": "headers",
        "headers": {"Authorization": "env.DOCS_MCP_AUTHORIZATION"},
        "tools_to_execute": ["*"]
      }
    ]
  },
  "agents": [
    {
      "name": "library-research",
      "agent_card_url": "https://research.example.com/.well-known/agent-card.json",
      "virtual_key_ids": ["vk-platform"],
      "forward_accepted_credential": true
    }
  ],
  "governance": {
    "virtual_keys": [
      {
        "id": "vk-platform",
        "name": "platform-team",
        "value": "env.BIFROST_VK_PLATFORM",
        "is_active": true,
        "provider_configs": [
          {"provider": "openai", "allowed_models": ["gpt-4o-mini"], "key_ids": ["*"], "weight": 1},
          {"provider": "anthropic", "allowed_models": ["*"], "key_ids": ["*"], "weight": 1}
        ],
        "mcp_configs": [
          {"mcp_client_name": "docs_search", "tools_to_execute": ["*"]}
        ],
        "rate_limit_id": "rl-platform"
      }
    ],
    "budgets": [
      {"id": "budget-platform", "virtual_key_id": "vk-platform", "max_limit": 200.00, "reset_duration": "1M"}
    ],
    "rate_limits": [
      {"id": "rl-platform", "request_max_limit": 1000, "request_reset_duration": "1h", "token_max_limit": 1000000, "token_reset_duration": "1h"}
    ]
  }
}
```

What each piece does:

- `enforce_auth_on_inference`: every model, MCP, and agent request must carry a valid virtual key.
- `a2a_external_client_url`: the public base URL of Bifrost; agent routes return `503` until it is set.
- `providers.*.keys[].value: "env.NAME"`: read from the Bifrost process environment.
- MCP client names must be ASCII without hyphens or spaces (`docs_search`, not `docs-search`).
- Agent names use lowercase letters, digits, and single hyphens; `virtual_key_ids` lists key IDs, not secret values.
- The virtual key is deny-by-default for providers and MCP tools: list what it may use. Its `value` comes from `BIFROST_VK_PLATFORM`; omit `value` to have Bifrost generate one and copy it from the dashboard.

### 2. Start Bifrost

```bash
export OPENAI_API_KEY=... ANTHROPIC_API_KEY=... DOCS_MCP_URL=... DOCS_MCP_AUTHORIZATION="Bearer ..."
export BIFROST_VK_PLATFORM="sk-bf-$(openssl rand -hex 24)"

npx -y @maximhq/bifrost
# or
docker run -p 8080:8080 -v $(pwd)/data:/app/data \
  -e OPENAI_API_KEY -e ANTHROPIC_API_KEY -e DOCS_MCP_URL -e DOCS_MCP_AUTHORIZATION -e BIFROST_VK_PLATFORM \
  maximhq/bifrost
```

Open `http://localhost:8080` for the dashboard. Clients authenticate with the value of `BIFROST_VK_PLATFORM`.

### 3. Protect it before exposing it

Unpinned commands are for local evaluation. Before other machines can reach Bifrost: pin a version, enable dashboard authentication (the first admin account needs a setup token from `BIFROST_SETUP_TOKEN`), and put TLS in front ([Protect the Dashboard](https://docs.getbifrost.ai/quickstart/gateway/setting-up#protect-the-dashboard)). For Kubernetes, use the [Helm chart](https://docs.getbifrost.ai/deployment-guides/helm#step-2-install).

## Verify all three surfaces

```bash
export BIFROST_VK="$BIFROST_VK_PLATFORM"

# Health
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/health

# LLM gateway
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" -H "x-bf-vk: $BIFROST_VK" \
  -d '{"model": "openai/gpt-4o-mini", "messages": [{"role": "user", "content": "ping"}]}'

# MCP gateway: only the tools the key allows
curl -s -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" -H "x-bf-vk: $BIFROST_VK" \
  -d '{"jsonrpc": "2.0", "id": 1, "method": "tools/list"}'

# Agent Gateway: the gateway's copy of the agent card
curl -s http://localhost:8080/agents/a2a/library-research/.well-known/agent-card.json \
  -H "x-bf-vk: $BIFROST_VK"
```

Then repeat one call without `x-bf-vk` and confirm it is refused, and check that each request appears in the dashboard logs (LLM, MCP, and Agent Logs).

## Point clients at it

- SDK code: change the base URL to `http://localhost:8080/openai` (or `/anthropic`, `/genai`, `/langchain`) and use the virtual key as the API key.
- Claude Code as a model client: `ANTHROPIC_BASE_URL=http://localhost:8080/anthropic` and `ANTHROPIC_AUTH_TOKEN=<virtual key>`.
- Claude Code as an MCP client: `claude mcp add --transport http bifrost http://localhost:8080/mcp --header "Authorization: Bearer <virtual key>"`.
- Coding agents in one step: the Bifrost CLI (`npx -y @maximhq/bifrost-cli`) launches Claude Code, Codex CLI, Gemini CLI, and OpenCode through Bifrost ([Bifrost CLI](https://docs.getbifrost.ai/quickstart/cli/getting-started#install-the-cli)).

## Open source and Enterprise

The steps above use open-source Bifrost. These are Bifrost Enterprise features, documented under `docs.getbifrost.ai/enterprise/`: clustering and high availability, adaptive routing, identity-provider SSO with SCIM user provisioning, access profiles, projects, RBAC, audit logs, and log exports ([Enterprise overview](https://docs.getbifrost.ai/enterprise/overview#built-on-open-source)). A single open-source node keeps its state in memory; running several OSS nodes against one Postgres database is not supported.

## Reference

- Bifrost docs: [Setting Up](https://docs.getbifrost.ai/quickstart/gateway/setting-up#30-second-setup), [config.json schema reference](https://docs.getbifrost.ai/deployment-guides/config-json/schema-reference#top-level-keys), [Virtual Keys](https://docs.getbifrost.ai/features/governance/virtual-keys#configuration), [MCP gateway](https://docs.getbifrost.ai/mcp/overview#bifrost-as-an-mcp-gateway), [Agent Gateway](https://docs.getbifrost.ai/features/agent-gateway/overview#start-with-one-agent).
- Concepts: [The three kinds of AI traffic](https://github.com/ai-gateways/ai-gateway#the-three-kinds-of-ai-traffic).
- Source: [maximhq/bifrost](https://github.com/maximhq/bifrost#quick-start).
