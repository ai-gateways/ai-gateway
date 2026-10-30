# AI Gateway

An AI gateway is a control point that sits between the applications, agents, and developer tools that use AI and the services they call, applying one set of identity, policy, routing, and logging rules to model calls, MCP tool calls, and agent-to-agent calls. Most teams first adopt one as a proxy for model APIs, then extend the same control point to the tools and agents those models start calling.

This guide explains what an AI gateway does, how it differs from an API gateway, which kinds of AI traffic it governs, and how to deploy and evaluate one. Each traffic type has its own detailed guide in this organization, linked from the section where it applies.

## Contents

- [What is an AI gateway?](#what-is-an-ai-gateway)
- [Why teams adopt an AI gateway](#why-teams-adopt-an-ai-gateway)
- [AI gateway vs API gateway](#ai-gateway-vs-api-gateway)
- [The three kinds of AI traffic](#the-three-kinds-of-ai-traffic)
- [Core capabilities](#core-capabilities)
- [Where an AI gateway sits](#where-an-ai-gateway-sits)
- [Deployment models](#deployment-models)
- [How to evaluate an AI gateway](#how-to-evaluate-an-ai-gateway)
- [Open-source AI gateway projects](#open-source-ai-gateway-projects)
- [Related guides](#related-guides)

## What is an AI gateway?

An AI gateway is a reverse proxy and policy engine for AI traffic. Callers send model requests, tool calls, and agent tasks to one endpoint. The gateway identifies the caller, applies budgets, permissions, and guardrails, routes each request to the right upstream service, and records what happened. Applications keep their own logic; the gateway owns the controls around it.

```text
apps, agents, coding tools
            |
            v
        AI gateway        identity, policy, routing, logs
            |
            +--> model providers   (model calls)
            +--> MCP servers       (tool calls)
            +--> remote agents     (agent-to-agent calls)
```

The term overlaps with three narrower ones. An [LLM gateway](https://github.com/ai-gateways/llm-gateway#what-is-an-llm-gateway) governs calls to model providers. An [MCP gateway](https://github.com/ai-gateways/mcp-gateway#what-is-an-mcp-gateway) governs tool calls between MCP clients and MCP servers. An [agent gateway](https://github.com/ai-gateways/agent-gateway#what-is-an-agent-gateway) governs tasks that one agent delegates to another over the A2A protocol. "AI gateway" is the umbrella term, and products use all four labels loosely, so the useful question is which of these traffic types a product actually governs.

## Why teams adopt an AI gateway

Teams adopt an AI gateway when AI usage spreads faster than the controls around it. The usual triggers are several model providers with separate keys, spend that cannot be attributed to a team, agents that call tools and other agents with no audit trail, and a security review that asks for one place to enforce AI policy.

- **Provider sprawl and outages.** Each provider enforces its own limits; [OpenAI's rate limits](https://developers.openai.com/api/docs/guides/rate-limits#how-do-these-rate-limits-work) apply per minute to requests and tokens and return HTTP 429 when exceeded, so every application ends up handling retries and fallbacks on its own.
- **Unattributed spend.** Token costs accumulate per request, and [unbounded consumption](https://genai.owasp.org/llmrisk/llm102025-unbounded-consumption/#:~:text=Unbounded%20Consumption%20refers%20to%20the%20process) is one of the OWASP Top 10 risks for LLM applications.
- **Credentials everywhere.** Provider keys, MCP server tokens, and agent credentials end up copied into many services and laptops. The MCP security guidance names [token passthrough](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices#token-passthrough) as an anti-pattern for exactly this reason.
- **No inventory or audit trail.** The OWASP MCP Top 10 lists [lack of audit and telemetry](https://github.com/OWASP/www-project-mcp-top-10/blob/165fe0f78ef104459237b4a8e0f6e78db9b02391/2025/MCP08-2025%E2%80%93Lack-of-Audit-and-Telemetry.md#description) and [shadow MCP servers](https://github.com/OWASP/www-project-mcp-top-10/blob/165fe0f78ef104459237b4a8e0f6e78db9b02391/2025/MCP09-2025%E2%80%93Shadow-MCP-Servers.md#description) among the main risks in MCP deployments.

The triggers differ by hop: see why teams adopt an [LLM gateway](https://github.com/ai-gateways/llm-gateway#why-teams-adopt-an-llm-gateway), an [MCP gateway](https://github.com/ai-gateways/mcp-gateway#why-teams-use-an-mcp-gateway), and an [agent gateway](https://github.com/ai-gateways/agent-gateway#why-agent-to-agent-traffic-needs-a-gateway). For one application calling one provider, a gateway is usually premature.

## AI gateway vs API gateway

An API gateway manages general HTTP traffic: TLS termination, authentication, request-count rate limits, and routing to backend services. An AI gateway understands what is inside AI requests: tokens, prompts and responses, provider formats, MCP tool calls, and A2A tasks. Most organizations run both, with the API gateway at the edge and the AI gateway behind it.

| Concern | API gateway | AI gateway |
|---|---|---|
| Protocols | HTTP, REST, gRPC | Provider model APIs, MCP, A2A |
| Unit of metering | Requests | Tokens and cost per model, plus requests |
| Payload awareness | Headers, paths, schemas | Prompts, completions, tool names, task states |
| Routing target | Backend services | Model providers, MCP servers, remote agents |
| Failure handling | Retries and circuit breaking across instances | Fallback across providers and model families |
| Security policy | Authentication, IP rules, schema validation | Guardrails on prompts and responses, tool allowlists, agent grants |
| Cost visibility | Not tracked | Spend per team, key, model, and provider |

Some API gateways now ship AI plugins, which blurs the line; test whether a product meters tokens, translates provider formats, and filters tools per caller. The [LLM gateway guide](https://github.com/ai-gateways/llm-gateway#llm-gateway-vs-api-gateway) compares the model hop in more detail, and the [MCP gateway guide](https://github.com/ai-gateways/mcp-gateway#mcp-gateway-vs-mcp-proxy-vs-mcp-server) separates gateways from proxies and servers on the tool hop.

## The three kinds of AI traffic

An AI gateway governs three hops: an application or agent calling a model, an agent calling a tool through MCP, and a caller delegating a task to another agent through A2A. Each hop has its own protocol, unit of work, and policy question, and each needs its own permission even when one gateway handles all three.

| Traffic | Protocol | Unit of work | Policy question | Detailed guide |
|---|---|---|---|---|
| Model calls | Provider APIs, usually OpenAI-compatible | Completion or embedding request | May this caller use this model, within what budget? | [How an LLM gateway works](https://github.com/ai-gateways/llm-gateway#how-an-llm-gateway-works) |
| Tool calls | [Model Context Protocol](https://modelcontextprotocol.io/specification/2026-07-28/server/tools#calling-tools) | `tools/call` request | May this agent call this tool, under whose credential? | [How an MCP gateway works](https://github.com/ai-gateways/mcp-gateway#how-an-mcp-gateway-works) |
| Agent calls | [Agent2Agent (A2A)](https://a2a-protocol.org/latest/specification/#11-key-goals-of-a2a) | Task with a lifecycle | May this caller delegate to this agent? | [How an agent gateway handles an A2A request](https://github.com/ai-gateways/agent-gateway#how-an-agent-gateway-handles-an-a2a-request) |

The A2A specification describes MCP and A2A as [complementary protocols](https://a2a-protocol.org/latest/specification/#appendix-b-relationship-to-mcp-model-context-protocol): MCP connects an agent to tools and data, and A2A connects one agent to another. One delegated task often crosses all three hops, so permission to call an agent should not imply permission to use an expensive model or a tool that writes to production data. The [agent gateway guide](https://github.com/ai-gateways/agent-gateway#agent-gateway-vs-mcp-gateway-vs-llm-gateway) shows how one task moves across the three.

## Core capabilities

An AI gateway combines a small set of controls that apply on every hop: caller identity, scoped permissions, budgets and rate limits, guardrails, routing with failover, credential brokering, caching, and observability. Products differ in how deeply they implement each control and how many traffic types each one covers.

```text
request
  -> identify caller (key, SSO token, client certificate)
  -> check permissions and budget ...... denied: reject and log
  -> input guardrails .................. blocked: reject and log
  -> route to upstream ................. error before delivery: fall back
  -> output guardrails
  -> record usage, cost, audit entry
response
```

- **Identity and permissions.** Gateway-issued keys or identity provider tokens, with grants that decide which models, tools, and agents each caller may use. See the [AI governance guide](https://github.com/ai-gateways/ai-governance#identity-and-access).
- **Budgets and rate limits.** Token and dollar limits per team, application, or key, enforced before a request reaches a provider.
- **Guardrails.** Checks on prompts, responses, and tool results for [prompt injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/#:~:text=A%20Prompt%20Injection%20Vulnerability%20occurs%20when%20user%20prompts%20alter), [sensitive information](https://genai.owasp.org/llmrisk/llm022025-sensitive-information-disclosure/#:~:text=Sensitive%20information%20can%20affect%20both%20the%20LLM%20and%20its%20application%20context), and policy violations. See the [AI guardrails guide](https://github.com/ai-gateways/ai-guardrails#types-of-ai-guardrails).
- **Routing and failover.** Weighted, cost-based, or latency-based routing with fallback chains across providers, as in the [LLM gateway features](https://github.com/ai-gateways/llm-gateway#core-features).
- **Credential brokering.** Upstream keys and OAuth tokens stay in the gateway. MCP servers must validate that each token was [issued for them](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization#token-handling), so the gateway brokers upstream credentials instead of forwarding client tokens, as the [MCP gateway capabilities](https://github.com/ai-gateways/mcp-gateway#core-mcp-gateway-capabilities) describe.
- **Cost reduction.** Caching for repeated prompts and tool filtering for [large tool sets](https://github.com/ai-gateways/mcp-gateway#reducing-token-cost-from-large-tool-sets).
- **Observability.** Logs, metrics, and traces for every hop, using OpenTelemetry attributes such as [`gen_ai.usage.input_tokens`](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/#gen-ai-usage-input-tokens), [`gen_ai.tool.name`](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/#gen-ai-tool-name), and [`gen_ai.agent.name`](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/#gen-ai-agent-name). The [LLM observability guide](https://github.com/ai-gateways/llm-observability#opentelemetry-for-llm-applications) covers tracing in depth.

## Where an AI gateway sits

An AI gateway sits behind the edge and in front of every AI upstream. Clients reach it through the existing API gateway or load balancer; it forwards approved requests to model providers, MCP servers, and remote agents; and it exports telemetry to the monitoring stack the organization already runs.

```text
clients       apps, agents, IDEs, coding agents
                 |
edge          API gateway or load balancer ...... TLS, WAF, request limits
                 |
AI layer      AI gateway ........................ identity, budgets, guardrails, routing
                 |                                   |
upstreams     model APIs, MCP servers, agents     logs, metrics, traces
                                                  -> existing monitoring
```

Applications and SDKs point their base URL at the gateway, MCP clients list it as their server, and A2A callers fetch Agent Cards from it, as described under [handling an A2A request](https://github.com/ai-gateways/agent-gateway#how-an-agent-gateway-handles-an-a2a-request). The MCP specification anticipates this layer: Streamable HTTP [mirrors request metadata into headers](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http#request-metadata) so that gateways can route requests without parsing the body.

Traffic that never passes the gateway is not governed by it. Desktop AI apps and coding agents on employee machines often call providers directly, so full coverage also means routing that endpoint traffic through the gateway.

## Deployment models

An AI gateway runs as a hosted service operated by a vendor, as a self-hosted service inside the company's cloud account or data center, or as a library embedded in each application. The choice decides where prompts, responses, tool arguments, and logs are processed and stored.

- **Hosted.** The vendor operates the gateway and AI traffic transits its infrastructure; data residency depends on the vendor's regions and retention policy.
- **Self-hosted.** The gateway runs in the company's VPC, Kubernetes cluster, or data center. Prompts, tool results, and logs stay inside that boundary, and air-gapped deployments with no outbound telemetry are possible.
- **Embedded library.** Routing logic runs inside each application, with no extra network hop but with policy enforced per application, which reintroduces the duplication a gateway removes.

The [LLM gateway deployment section](https://github.com/ai-gateways/llm-gateway#deployment-models) covers the trade-offs for model traffic. For regulated workloads, the NIST AI Risk Management Framework calls for [ongoing monitoring and periodic review](https://airc.nist.gov/airmf-resources/airmf/5-sec-core/#:~:text=Ongoing%20monitoring%20and%20periodic%20review) of AI risk management, and logs kept inside the organization are a direct source for that record.

## How to evaluate an AI gateway

Evaluate an AI gateway on which traffic it covers, what it adds to every request, where it can run, and how it connects to existing identity and monitoring systems. A gateway that governs only model calls leaves tool and agent traffic to separate control points, each with its own keys, policies, and logs.

- **Traffic coverage.** Check whether model calls, MCP tool calls, and A2A agent calls fall under one identity and policy model, with separate grants for each hop.
- **Added latency under load.** Look for a published benchmark that states per-request overhead at a stated throughput and instance size, and reproduce it. Microsecond-level overhead at thousands of requests per second is achievable. Agent loops make dozens of model and tool calls per task, so millisecond-level overhead compounds.
- **Deployment and data boundary.** Confirm the gateway runs inside your VPC or on-premises, including air-gapped, and that no prompts, responses, tool results, or telemetry leave your infrastructure.
- **Identity and governance.** Single sign-on through your existing identity provider, such as Okta or Microsoft Entra ID, plus role-based access control, scoped keys, and budgets per organization, team, and user.
- **Security controls.** Guardrails on prompts, responses, and tool results; per-user credentials on the tool hop; and audit logs that cannot be edited.
- **Observability.** OpenTelemetry tracing, Prometheus metrics, and log export, with one trace ID joining model, tool, and agent calls.
- **Reliability.** Clustering without a single leader node, circuit breaking, adaptive load balancing, and configuration changes that apply without a restart.
- **License and tiers.** A permissive license such as Apache 2.0 or MIT allows self-hosting and code review. Check which features sit in the open-source core and which require a paid tier or a separately licensed directory.
- **Compatibility.** An OpenAI-compatible API, support for the current MCP specification revision, and the A2A protocol bindings your agents use.

Each detailed guide has its own checklist: [LLM gateway](https://github.com/ai-gateways/llm-gateway#how-to-evaluate-an-llm-gateway), [MCP gateway](https://github.com/ai-gateways/mcp-gateway#how-to-evaluate-an-mcp-gateway), and [agent gateway](https://github.com/ai-gateways/agent-gateway#how-to-evaluate-an-agent-gateway). Before choosing, load test with your own prompts, tools, and providers, and force an upstream failure to confirm fallback works.

## Open-source AI gateway projects

Several open-source projects implement an AI gateway, with different coverage of the three traffic types. The table lists facts each project publishes in its own repository; benchmark figures use different test setups and are not directly comparable.

| Project | Language, license | Model calls | MCP tool calls | A2A agent calls | Published overhead benchmark | Deployment |
|---|---|---|---|---|---|---|
| [Bifrost](https://github.com/maximhq/bifrost#key-features) | Go, Apache 2.0 | Yes | Yes | Yes | About 11 µs added per request at 5,000 RPS on one t3.xlarge | Single binary via npx, Docker, Kubernetes, or Go SDK; in-VPC, on-premises, or air-gapped |
| [LiteLLM](https://github.com/BerriAI/litellm#features) | Python with a Rust core; MIT, except the `enterprise/` directory | Yes | Yes | Yes | 8 ms P95 latency at 1,000 RPS | Self-hosted proxy (Docker) or Python SDK |
| [Agent Router](https://github.com/theagentrouter/agent-router#formerly-envoy-ai-gateway) (formerly Envoy AI Gateway) | Go, Apache 2.0 | Yes | Yes | Not published | Not published in the repository | Kubernetes with Envoy Gateway, or the `aigw` CLI |
| [Kong Gateway](https://github.com/Kong/kong#getting-started-with-ai-gateway-for-llm-and-mcp) | Lua, Apache 2.0 | Yes | Yes | Not published | Not published in the repository | Docker or Kubernetes with the Kong Ingress Controller |
| [ContextForge](https://github.com/IBM/mcp-context-forge#overview--goals) (IBM) | Python, Apache 2.0 | Not published | Yes | Yes | Not published in the repository | PyPI, Docker, or Kubernetes with Helm |

Details reflect each repository in October 2026; confirm current capabilities in each project's documentation before choosing.

Coverage differs most on agent-to-agent traffic, which three of the five projects document.

## Related guides

- [How an LLM gateway works](https://github.com/ai-gateways/llm-gateway#how-an-llm-gateway-works): routing, budgets, and failover for model calls
- [MCP gateway security](https://github.com/ai-gateways/mcp-gateway#mcp-gateway-security): token handling and tool poisoning on the tool hop
- [How the A2A protocol works](https://github.com/ai-gateways/agent-gateway#how-the-a2a-protocol-works): Agent Cards, tasks, and bindings on the agent hop
- [Core AI governance controls](https://github.com/ai-gateways/ai-governance#core-ai-governance-controls): policies, permissions, and audit for AI traffic
- [Traces, metrics, and logs](https://github.com/ai-gateways/llm-observability#traces-metrics-and-logs): tracing model, tool, and agent calls
- [Input and output guardrails](https://github.com/ai-gateways/ai-guardrails#input-and-output-guardrails): checks on prompts, responses, and tool results
- [AI Gateways](https://github.com/ai-gateways): all guides in this organization
