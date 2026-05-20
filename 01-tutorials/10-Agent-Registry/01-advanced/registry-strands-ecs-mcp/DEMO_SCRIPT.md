# Demo Discussion Script
## AWS Agent Registry + Strands Agent — Financial Skills Demo

---

## 1. What We're Showing

This demo runs a production Strands agent on ECS Fargate that has **no hardcoded skills or tools**. Every request follows the same pattern: query the AWS Agent Registry, read the matching skill, load only the tools that skill declares, answer the question.

The financial analysis is incidental. The point is to show **how the AWS Agent Registry makes agents composable and discoverable** — so that adding a new skill requires no code changes to the agent, just publishing a new record to the registry.

---

## 2. The AWS Agent Registry — The Core Concept

### What It Is

The AWS Agent Registry is a unified, searchable catalog for agent capabilities. It stores records of different types — skills, MCP servers, other agents — and surfaces them through a single semantic search API. An agent calls `search_registry_records` with a natural-language query and gets back a ranked list across all record types.

The key idea: **the registry is the control plane for agent behavior**. The agent itself is a generic executor. What it knows how to do, and which tools it uses to do it, are discovered at runtime from the registry — not compiled in.

### Two Record Types in This Demo

**AGENT_SKILLS records** store a `SKILL.md` — a structured markdown file with YAML frontmatter and step-by-step instructions. The frontmatter declares what the skill does (used for semantic matching) and which MCP tools it needs:

```yaml
name: quarterly-kpi-calculator
description: Calculates quarterly financial KPIs from P&L data...
mcp_tools:
  - get_financial_data
  - get_kpi_benchmarks
```

The body is a literal procedure the agent follows — fetch this data, run this calculation, format the result this way. The agent reads it and executes it. No skill logic lives in the agent code.

**MCP records** describe a server's capabilities. The registry crawls the server at registration time, stores the tool schemas inline, and makes them searchable alongside skills. When the agent starts up, it queries the registry for the MCP record, reads the server URL from the stored manifest, and caches it — no hardcoded URLs, no SSM parameters.

Both types live in the same index. One `search_registry_records` call returns them together, ranked by semantic similarity.

### The Per-Request Flow

1. User asks a question
2. Agent calls `search_registry_records` — all record types returned, ranked
3. Top AGENT_SKILLS result is selected → its `SKILL.md` is read from the record
4. `mcp_tools:` frontmatter declares exactly which tools this skill needs
5. Agent connects to MCP server, loads **only those tools** — not the full server catalog
6. Strands Agent runs with skill instructions + scoped tools → Bedrock produces the answer

Step 5 matters: a real MCP server might expose dozens of tools. Loading all of them into every request wastes context tokens and gives the model irrelevant choices. The skill declares its dependencies; the agent loads exactly those. This pattern scales — many skills, many servers, each request loads only what it needs.

### What We Have in the Registry (6 records, all APPROVED)

| Record | Type | Description |
|--------|------|-------------|
| `financial-tools-mcp` | MCP | Tools: `get_financial_data`, `get_kpi_benchmarks` |
| `quarterly-kpi-calculator` | AGENT_SKILLS | Calculates Gross Margin, EBITDA Margin, OpEx Ratio, QoQ Growth |
| `cost-efficiency-analyzer` | AGENT_SKILLS | Analyzes cost structure and efficiency ratios |
| `revenue-growth-analyst` | AGENT_SKILLS | Revenue trend and growth analysis |
| `multi-quarter-trend-analysis` | AGENT_SKILLS | Trend narrative across multiple quarters |
| `executive-financial-briefing` | AGENT_SKILLS | One-page CFO-ready briefing |

---

## 3. Infrastructure — Built to Serve the Registry

The infrastructure choices in this demo are all driven by one constraint: **the registry is an AWS-managed service that lives outside your VPC, and it needs to reach your MCP server to crawl it**.

### Private VPC, Internal Services

All three ECS services (MCP server, Strands agent, chat interface) run in private subnets — no public IPs. The MCP server and agent are only reachable from within the VPC. The chat service is the only one exposed through CloudFront.

### Why API Gateway + VPC Link

The registry crawler needs an HTTPS endpoint to reach the MCP server. A private ALB DNS name is unresolvable from outside the VPC — you cannot hand that to the registry.

**API Gateway HTTP API with a VPC Link** solves this: API Gateway creates a managed network interface inside your private subnet, bridging the public API Gateway endpoint to the internal MCP ALB. The registry calls `https://1hybttfhe2.execute-api.us-east-1.amazonaws.com/mcp`, API Gateway routes it through the VPC Link, the internal ALB forwards it to the MCP container. The MCP server is never directly exposed.

The endpoint is IAM-protected with SigV4 — only callers with `execute-api:Invoke` permission can reach it. The registry crawler signs its requests using the task role you provide in `credentialProviderConfigurations`.

Without API Gateway + VPC Link, the only option is opening the MCP server directly to the internet — which is not acceptable for a private service.

### Why VPC Egress (NAT Gateway) Is Needed

The agent ECS task makes outbound calls to two AWS service APIs — `bedrock-agentcore` (registry search) and `bedrock-runtime` (model inference). Private subnets have no internet route by default, so without egress those calls fail. A **NAT Gateway** in the public subnet provides that outbound route.

The production-grade alternative: **VPC PrivateLink endpoints** for `bedrock-agentcore` and `bedrock-runtime`. Traffic stays on the AWS backbone without touching the internet, and the NAT Gateway is no longer needed.

### VPC Lattice — The Production Pattern

API Gateway + VPC Link works for a single MCP server in a single account. At scale — many agents, many MCP servers, potentially across accounts — **VPC Lattice** is the right answer. Each MCP server registers as a Lattice service. Agents connect through the Lattice service mesh with built-in auth, traffic policies, and observability. No per-server API Gateway overhead, and the registry URL-sync model works identically. This demo uses API Gateway because it's simpler to wire in one account, and the concept is the same.

---

## 4. What We Had to Work Through

Getting the MCP record registered in the registry involved three sequential problems, none documented end-to-end.

**Problem 1: IAM trust policy**

Error: `Unable to assume the provided IAM role for MCP server authentication.`

The registry crawler assumes the IAM role you provide in `credentialProviderConfigurations` when it calls `sts:AssumeRole`. The role's trust policy only listed `ecs-tasks.amazonaws.com`. Fix: add `bedrock-agentcore.amazonaws.com` as a trusted principal.

**Problem 2: SSE transport not supported**

Error: `SSE streams from MCP server is currently not supported.`

This is a [documented public-preview limitation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/registry-sync-records.html). The registry crawler sends a one-shot POST and expects a plain JSON response. FastMCP defaults to SSE responses. Fix: `json_response=True` on `mcp.http_app()` — the server returns plain JSON for POST requests. The MCP client the agent uses negotiates response format via `Accept: application/json, text/event-stream` and handles both, so this is fully backward compatible. AWS has stated SSE support is coming.

**Problem 3: URL extraction from search results**

Once the record was approved, the agent's startup search found the MCP record but couldn't extract the URL. `synchronizationConfiguration` (which contains the original URL) is not returned by `search_registry_records`. The URL lives in `descriptors.mcp.server.inlineContent` — a JSON object following the MCP server manifest schema (`modelcontextprotocol.io/schemas/2025-12-11`) — under `remotes[0].url`. Fix: parse that path.

---

## 5. Precise Mechanics — What Happens When, and Why

Before walking through the demo, this section answers the questions that come up most often: when does initialization happen, when does the registry get called, what gets loaded, and how does this compare to AgentCore Gateway. Everything here is grounded in AWS documentation; where we built something custom on top, that is called out explicitly.

---

### When Does "Initialization" Happen?

Initialization happens when the **ECS task starts** — not when a user opens the browser, not when they log in, not when they send their first message. The ECS task is already running and warm before any user arrives.

When the browser reaches the chat screen, the agent process has been running for however long the ECS task has been up. The user hitting the chat screen does not trigger any agent or registry activity.

The agent is invoked for the first time when the user **submits a message**.

---

### What Happens at Agent Startup (Before Any Request)

When the ECS task starts, the agent does exactly one thing with the registry:

> It calls `search_registry_records` with the query `"financial tools MCP server"`, finds the MCP record, parses `descriptors.mcp.server.inlineContent → remotes[0].url`, and caches the MCP server URL.

That is all that happens at startup. No MCP connection is made. No tools are loaded. No skills are read. The agent does not know what skills exist yet. The cached URL is just a string — it sits in memory until a request arrives.

**What the agent has at startup:**
- MCP server URL (a cached string — no connection open, no tools loaded)
- Three base tools built into the agent code: `search_and_load_skill`, `file_read`, `python_exec`
- A system prompt describing how to use the registry

**What the agent does NOT have at startup:**
- Any MCP tools (`get_financial_data`, `get_kpi_benchmarks` — not loaded, no MCP connection)
- Any skill instructions (no SKILL.md has been read)
- Any knowledge of what skills exist in the registry

**Does the agent know which tools and skills are available at startup?**

No. The registry is not queried for skills at startup — only for the MCP server URL. The agent has no inventory of available skills. It discovers them per request, on demand, by searching the registry with the user's actual question.

This is consistent with how the AWS Agent Registry is designed. The documentation describes the registry as a **discovery catalog** — its `search_registry_records` API is the mechanism for finding resources. There is no "load all skills" or "enumerate capabilities" call. Discovery is always query-driven.

**Does the agent get tool schemas from the registry record before connecting to the MCP server?**

Yes, partially. When the registry crawls an MCP server at registration time (`synchronizationType=URL`), it stores the tool schemas inside the MCP record under `descriptors.mcp.tools.inlineContent`. Those schemas are returned when you search or get the record — so an agent could technically read tool definitions from the registry without connecting to the MCP server at all.

In this demo we do not use the registry-stored tool schemas for invocation. We use the MCP server URL from the registry to connect directly to the server and call `list_tools_sync()` to get live tool objects. The registry schemas are used for searchability and display; the live connection is used for execution. This is the correct separation — the registry stores metadata, but tool execution always goes to the actual server.

---

### What Happens Per Request — The Two-Phase Pattern

Every request goes through two phases before Bedrock is called:

**Phase 1 — Pre-flight (framework code, runs before the Strands agent):**

1. `search_registry_records` is called with the user's message as the query
2. The top AGENT_SKILLS result is identified
3. Its `mcp_tools:` frontmatter is parsed — this lists which MCP tools that skill uses
4. An MCP connection is opened and **only those listed tools** are loaded into the agent's context
5. The registry search results are emitted to the UI as the search panel

**Phase 2 — Strands agent runs:**

6. The agent calls `search_and_load_skill` as a tool — this does a second registry search and loads the full SKILL.md for the top match into the agent's context as text
7. The agent reads the SKILL.md instructions and follows them step by step
8. It calls MCP tools and `python_exec` as instructed by the skill
9. Bedrock (Claude Sonnet 4.6) produces the final answer

**The key point on tool loading:** MCP tools are never loaded for every request. They are loaded only when the skill matched to this specific request lists them in its frontmatter. A different skill on a different request may list different tools — or none at all.

**Important: the `mcp_tools:` field in the SKILL.md frontmatter is a custom convention we built, not a native registry feature.** The AWS Agent Registry's AGENT_SKILLS `skillDefinition` schema has no field for declaring MCP tool dependencies. The registry stores and returns the `skillMd` text opaquely — it has no understanding of the `mcp_tools:` list inside it. Our agent code parses the frontmatter and uses it to decide which tools to load. This is a pattern built on top of the registry, not a feature the registry provides natively.

---

### How Does This Compare to AgentCore Gateway?

This is a question worth answering precisely because Gateway and Registry solve adjacent but different problems.

**AgentCore Gateway** ([docs](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway.html)) is an **execution layer**. It aggregates multiple MCP servers into a single unified virtual MCP server. An agent connects to the gateway endpoint and calls `tools/list` — it gets back all tools from all attached MCP servers in one response. The gateway also has a built-in semantic search tool (`x_amz_bedrock_agentcore_search`) that the agent can call with a natural language query to find the most relevant tools from the full set. From the docs:

> *"Semantic Tool Selection — Enables agents to search across available tools to find the most appropriate ones for specific contexts, allowing agents to leverage thousands of tools while minimizing prompt size and reducing latency."*

With Gateway, the semantic tool filtering happens **inside the gateway layer** using that built-in search tool. The agent calls the search tool, gets back a filtered tool list, then calls those tools — all through one endpoint.

**AWS Agent Registry** ([docs](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/registry.html)) is a **discovery catalog**. It is not an execution layer. It stores metadata about MCP servers, skills, agents, and other resources. It provides semantic search across all record types. When an agent finds an MCP record in the registry, it gets the server URL and tool schemas — but must then connect directly to the actual MCP server to invoke tools.

| | AgentCore Gateway | AWS Agent Registry (this demo) |
|---|---|---|
| **What it is** | Execution proxy / aggregator | Discovery catalog |
| **Semantic search** | Built-in `x_amz_bedrock_agentcore_search` tool, runs at invocation time | `search_registry_records` API, called per-request by agent code |
| **Tool loading** | Agent calls `tools/list` on gateway → gets all tools from all attached servers | Agent connects directly to MCP server; loads only declared tools (custom pattern in this demo) |
| **Filtering** | Agent calls search tool; gateway returns semantically relevant subset | Agent parses `mcp_tools:` from SKILL.md frontmatter and filters client-side (custom code) |
| **MCP server connections** | Gateway manages; agent sees one endpoint | Agent manages directly; one connection per MCP server |
| **Skills/procedures** | Not a Gateway concept | AGENT_SKILLS records store step-by-step instructions |
| **Who filters tools** | Gateway (via semantic search tool) | Agent code (custom, not a registry feature) |

**The short version:** Gateway is the right choice when you have many MCP servers and want aggregation, unified auth, and built-in semantic tool routing at the execution layer. Registry is the right choice when you want a searchable catalog of diverse resources — skills, agents, servers — that agents can discover and compose dynamically. They are complementary, not competing. A production system could use both: the registry to discover which gateway and which skills apply to a request, and the gateway to execute tools through a unified interface.

---

### Opening

Point at the description: *"No skills or tools are hardcoded in the agent."*

Show the suggestion chips — five groups, one per skill. These represent what the registry currently knows about. Point out that the grouping itself comes from the registry — each group is one AGENT_SKILLS record. Adding a new group means publishing a new record, nothing else.

---

### Request 1 — Quarterly KPI Calculator

**Say:** `"Calculate KPIs for Q3 2025"`

**Narrate the startup state first:**

"Before I send this, the agent is sitting idle with no skills loaded and no MCP connection open. It knows the MCP server URL from its startup registry query, and it has three base tools: the ability to search the registry, read files, and execute Python. That's it."

**While Phase 1 runs (registry search panel appears):**

"The first thing the framework does is call `search_registry_records` with the exact text of my question. The registry does a semantic vector search across all six approved records — MCP and AGENT_SKILLS together. What comes back is a ranked list. The top AGENT_SKILLS match is `quarterly-kpi-calculator`."

"The framework reads the frontmatter of that skill record — just the YAML header, not the full instructions yet. That frontmatter says: `mcp_tools: [get_financial_data, get_kpi_benchmarks]`. Those are the only two tools this skill needs. So the framework opens an MCP connection right now and loads exactly those two tools — nothing else from the MCP server — into this request's context."

"The MCP server exposes two tools total, so in this case we load both. But if the server had fifty tools, we'd still only load these two. The skill controls what goes into context."

**While Phase 2 runs (pipeline steps advance):**

"Now the Strands agent starts. Its context at this point contains: the three base tools, plus the two MCP tools just loaded — five tools total. It calls `search_and_load_skill` as its first action, which fetches the full SKILL.md from the registry record and returns it as text into the agent's context."

"The agent now has the complete procedure: five steps, exactly specified. Step 1: call `get_kpi_benchmarks`. Step 2: call `get_financial_data` for Q3 2025. Step 3: run Python to calculate the KPIs. Step 4: interpret against benchmarks. Step 5: format a table."

"Each MCP tool call goes from the agent ECS task through API Gateway, through the VPC Link, to the MCP ALB, to the MCP container — SigV4 signed the whole way. `python_exec` runs a subprocess inside the agent container."

**Point at the result:**

"The KPI table and commentary are the output of following those instructions exactly. None of that calculation logic — the formulas, the GREEN/YELLOW/RED thresholds, the output format — is in the agent code. It all came from the SKILL.md in the registry record."

**Context summary for this request:**
- Tools in context: `search_and_load_skill`, `file_read`, `python_exec`, `get_financial_data`, `get_kpi_benchmarks` (5 total)
- Skills loaded: `quarterly-kpi-calculator` SKILL.md (read from registry during agent reasoning)
- MCP calls made: `get_kpi_benchmarks` × 1, `get_financial_data` × 2 (Q3 and Q2 for QoQ growth)

---

### Request 2 — Executive Financial Briefing

**Say:** `"Give me an executive financial briefing"`

**Narrate:**

"New request, clean slate. The agent's per-request context is rebuilt from scratch — the MCP connection from the previous request was closed when that request finished. The two MCP tools from request 1 are no longer in context."

"Phase 1 runs again. Same registry search call, different query. The semantic vector search returns a different top match this time: `executive-financial-briefing`. Its frontmatter also declares `[get_financial_data, get_kpi_benchmarks]` — same tools, but that's a coincidence of this domain. In a broader registry with more diverse skills, different skills would declare entirely different tool sets."

"The SKILL.md for this skill is different from the KPI calculator's. It specifies a structured output format with five mandatory sections: Headline Numbers, Performance vs Benchmarks, What's Working, Watch List, Recommended Actions. Under 400 words. Written for a CFO who reads it in 60 seconds."

**Point at the result:**

"The agent produced exactly that structure. Same agent process, same infrastructure, different skill from the registry — completely different behavior and output format."

**Context summary for this request:**
- Tools in context: `search_and_load_skill`, `file_read`, `python_exec`, `get_financial_data`, `get_kpi_benchmarks` (5 total — same tools, different skill instructions)
- Skills loaded: `executive-financial-briefing` SKILL.md
- MCP calls made: `get_financial_data` × 2 (Q3 and Q2), `get_kpi_benchmarks` × 1

---

### Request 3 — Multi-Quarter Trend

**Say:** `"Show me revenue trend across all quarters"`

**Narrate:**

"Again, per-request context rebuilt from scratch. Phase 1 searches the registry — top match is `multi-quarter-trend-analysis`. Same two MCP tools declared in the frontmatter. SKILL.md instructs the agent to fetch all four quarters — Q4 2024 through Q3 2025 — and produce a trend narrative rather than a snapshot."

**Point at the result:**

"Four MCP calls to `get_financial_data`, one per quarter. Trend analysis produced. Same pattern — different skill record, different behavior."

**Context summary for this request:**
- Tools in context: same 5
- Skills loaded: `multi-quarter-trend-analysis` SKILL.md
- MCP calls made: `get_financial_data` × 4 (all quarters), `get_kpi_benchmarks` × 1

---

### Request 4 — What the Registry Search Actually Does

**Say:** `"What financial skills do you have access to?"`

**Narrate:**

"This is a meta-question — the user isn't asking for financial analysis, they're asking what the agent can do. Watch what happens."

"Phase 1 still runs. The registry search still executes with this question as the query. The top AGENT_SKILLS match will be whichever skill is semantically closest to this question — not a special 'list skills' mode. The two MCP tools for that skill are pre-loaded into context."

"The Strands agent then calls `search_and_load_skill` and gets back the ranked candidate list — all five skills with their descriptions. It uses that to answer the meta-question."

"The important point: there is no short-circuit, no hardcoded list of skills, no special handling. The agent answers this question the same way it answers every question — by searching the registry and reading what it finds."

---

### Close

"Every behavior you just saw — the calculations, the formatting, the structured briefing, the tool calls — came from records in the registry, not from the agent code. The agent code has no financial logic in it."

"To add a new skill — say, cash flow forecasting — you write a SKILL.md with the procedure, call `CreateRegistryRecord` with `descriptorType=AGENT_SKILLS`, and approve it. The agent discovers it the next time someone asks a question that matches it semantically. No code change, no redeployment, no restart."

"The registry is the control plane. The agent is a generic executor."

---

## 6. Key Reference

| | |
|---|---|
| Demo URL | `https://d15k2eii8qdkf8.cloudfront.net` |
| Registry ARN | `arn:aws:bedrock-agentcore:us-east-1:336468392708:registry/0ybA4HzNZgVlIJl6` |
| Registry records | 6 approved (1 MCP + 5 AGENT_SKILLS) |
| MCP endpoint | `https://1hybttfhe2.execute-api.us-east-1.amazonaws.com/mcp` |
| MCP tools | `get_financial_data`, `get_kpi_benchmarks` |
| Model | Claude Sonnet 4.6 (cross-region inference profile) |
| Egress | NAT Gateway (production: VPC PrivateLink for bedrock-agentcore, bedrock-runtime) |
