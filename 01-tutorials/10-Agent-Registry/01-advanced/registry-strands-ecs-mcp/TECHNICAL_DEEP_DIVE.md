# Technical Deep Dive
## AWS Agent Registry + Strands Agent — Financial Skills Demo

---

## Table of Contents

1. [What We Built — The Core Idea](#1-what-we-built)
2. [AWS Agent Registry — Concepts and Record Types](#2-aws-agent-registry)
3. [The SKILL.md Format](#3-the-skillmd-format)
4. [Registering an MCP Server — What It Actually Takes](#4-registering-an-mcp-server)
5. [The Agent — Per-Request Flow with Code](#5-the-agent)
6. [The MCP Server](#6-the-mcp-server)
7. [Infrastructure — Why It's Built This Way](#7-infrastructure)
8. [What We Had to Figure Out](#8-what-we-had-to-figure-out)
9. [End-to-End Data Flow Diagram](#9-end-to-end-data-flow)
10. [Registry Record Reference](#10-registry-record-reference)

---

## 1. What We Built

A **Strands agent** running on ECS Fargate that has no hardcoded skills or tools. It discovers what it can do at runtime by querying the **AWS Agent Registry** — a unified, semantically-searchable catalog that stores both skill instructions (AGENT_SKILLS records) and MCP server capabilities (MCP records).

The financial analysis domain is the vehicle. The architecture is the point.

**The core insight:** If you separate *what an agent knows how to do* from *the agent itself*, you can update agent behavior — add skills, change procedures, introduce new tools — without touching the agent code or redeploying. The registry is the control plane. The agent is a generic executor.

### Components

| Component | Technology | Role |
|---|---|---|
| AWS Agent Registry | Amazon Bedrock AgentCore | Unified catalog — stores skills and MCP servers, serves semantic search |
| Strands Agent | Python / Strands SDK | Per-request: search registry → read skill → load tools → call Bedrock |
| MCP Server | FastMCP / Python | Exposes `get_financial_data` and `get_kpi_benchmarks` tools |
| Chat Interface | Python / FastAPI | Browser-facing SSE proxy between user and agent |
| Frontend | React (single HTML) | Renders chat, registry search visualization, pipeline steps |
| Infrastructure | ECS Fargate, API GW, VPC | Private VPC hosting, HTTPS endpoint for registry crawler |

---

## 2. AWS Agent Registry

### What It Is

The AWS Agent Registry (`bedrock-agentcore` / `bedrock-agentcore-control`) is a managed service that acts as a unified catalog for agent capabilities. It supports four record types:

| Record Type | What It Stores | Used For |
|---|---|---|
| `AGENT_SKILLS` | `SKILL.md` — step-by-step instructions + metadata | Telling an agent *how* to do a task |
| `MCP` | Tool schemas (auto-crawled) + server URL | Registering an MCP server so agents can discover it |
| `A2A` | Agent-to-agent protocol info | Multi-agent delegation |
| `CUSTOM` | Any structured content | Custom use cases |

All record types live in the same semantic search index. A single `search_registry_records` call returns records ranked by vector similarity across all types.

### The Two API Clients

```python
# Control plane — create, approve, delete records
registry_client = boto3.client("bedrock-agentcore-control")

# Data plane — semantic search
search_client = boto3.client("bedrock-agentcore")
```

### Creating a Registry

```python
resp = registry_client.create_registry(
    name="financial-skills-registry",
    description="Registry for financial skills and MCP tools",
    approvalConfiguration={"autoApproval": False},
)
registry_arn = resp["registryArn"]
registry_id  = registry_arn.split("/")[-1]
```

### Record Lifecycle

```
create_registry_record()
        ↓
    CREATING → DRAFT
        ↓
submit_registry_record_for_approval()
        ↓
    PENDING_APPROVAL
        ↓
update_registry_record_status(status="APPROVED")
        ↓
    APPROVED  ← now searchable (~100s for search index to update)
```

### Semantic Search

```python
response = search_client.search_registry_records(
    registryIds=[registry_id],
    searchQuery="calculate quarterly KPIs from P&L data",
    maxResults=10,
)

# Returns all record types ranked by semantic similarity
for record in response["registryRecords"]:
    print(record["descriptorType"], record["name"], record["description"])
# MCP    financial-tools-mcp          MCP server providing financial tools...
# AGENT_SKILLS  quarterly-kpi-calculator    Calculates quarterly financial KPIs...
# AGENT_SKILLS  executive-financial-briefing ...
```

---

## 3. The SKILL.md Format

Each AGENT_SKILLS record stores a `SKILL.md` file as inline content. The format is Markdown with a YAML frontmatter block. The frontmatter is what the agent reads programmatically; the body is what the agent follows as instructions.

### Frontmatter Fields

```yaml
---
name: quarterly-kpi-calculator
description: >
  Calculates quarterly financial KPIs from P&L data.
  Use when the user wants Gross Margin %, EBITDA Margin %,
  Operating Expense Ratio, or Revenue Growth % QoQ.
metadata:
  version: "2.0"
  tags: finance, kpi, analysis
mcp_tools:
  - get_financial_data
  - get_kpi_benchmarks
---
```

The `mcp_tools:` field is the key one. It declares exactly which MCP tools this skill needs. The agent reads this at request time and loads **only those tools** — not the full MCP server catalog.

### Full Example: quarterly-kpi-calculator SKILL.md

```markdown
---
name: quarterly-kpi-calculator
description: Calculates quarterly financial KPIs from P&L data. P&L figures can be
  provided directly by the user or fetched from the financial data MCP server.
  Use when the user wants KPI calculations such as Gross Margin %, EBITDA Margin %,
  Operating Expense Ratio, or Revenue Growth % QoQ.
metadata:
  version: "2.0"
  tags: finance, kpi, analysis, mcp
mcp_tools:
  - get_financial_data
  - get_kpi_benchmarks
---

# Quarterly KPI Calculator

## Steps

### Step 1: Retrieve benchmark thresholds
Call the get_kpi_benchmarks tool to get current KPI formulas and benchmark values.

### Step 2: Get P&L data
If the user specified a quarter (e.g. "Q3 2025"), call:
    get_financial_data(period="Q3 2025")

### Step 3: Calculate KPIs
Use python_exec to calculate:
- Gross Margin %          = (Revenue - COGS) / Revenue * 100
- EBITDA Margin %         = EBITDA / Revenue * 100
- Operating Expense Ratio = Operating Expenses / Revenue * 100
- Revenue Growth % QoQ    = (Current - Prior) / Prior * 100

### Step 4: Interpret against benchmarks
GREEN  : at or above general_benchmark
YELLOW : within 5 percentage points below
RED    : more than 5 percentage points below

### Step 5: Present results
Format as a KPI table: | Metric | Value | Benchmark | Status |
Followed by a 2-3 sentence executive commentary.
```

### Publishing an AGENT_SKILLS Record

```python
with open("my_skills/quarterly-kpi-calculator/SKILL.md") as f:
    skill_md = f.read()

resp = registry_client.create_registry_record(
    registryId=registry_id,
    name="quarterly-kpi-calculator",
    description=(
        "Calculates quarterly financial KPIs from P&L data. "
        "Use for Gross Margin %, EBITDA Margin %, Operating Expense Ratio, "
        "Revenue Growth % QoQ, quarterly performance review, or P&L analysis."
    ),
    descriptorType="AGENT_SKILLS",
    descriptors={
        "agentSkills": {
            "skillMd":         {"inlineContent": skill_md},
            "skillDefinition": {"inlineContent": json.dumps({"packages": []})},
        }
    },
    recordVersion="1.0",
)
```

---

## 4. Registering an MCP Server

This is the most technically involved part of the setup. It required solving three sequential problems before it worked.

### The Goal

Register an MCP server so the registry:
1. Crawls the server's tools and stores the schemas inline
2. Makes those tools semantically searchable alongside skills
3. Stores the server URL in the record so agents can discover it at startup

### The API Call

```python
resp = registry_client.create_registry_record(
    registryId=registry_id,
    name="financial-tools-mcp",
    description=(
        "MCP server providing financial tools: "
        "get_financial_data (quarterly P&L data retrieval), "
        "get_kpi_benchmarks (industry benchmark thresholds and formulas)."
    ),
    descriptorType="MCP",
    synchronizationType="URL",
    synchronizationConfiguration={
        "fromUrl": {
            "url": "https://1hybttfhe2.execute-api.us-east-1.amazonaws.com/mcp",
            "credentialProviderConfigurations": [
                {
                    "credentialProviderType": "IAM",
                    "credentialProvider": {
                        "iamCredentialProvider": {
                            "roleArn": "arn:aws:iam::336468392708:role/financial-agent-agent-task-role",
                            "service": "execute-api",
                            "region":  "us-east-1",
                        }
                    },
                }
            ],
        }
    },
    recordVersion="1.0",
)
```

**`synchronizationType="URL"`** tells the registry to crawl the endpoint and auto-populate tool schemas.
**`credentialProviderConfigurations`** tells the registry which IAM role to assume and which AWS service to sign for when crawling.

### Problem 1: IAM Trust Policy

**Error:** `Unable to assume the provided IAM role for MCP server authentication.`

The registry crawler calls `sts:AssumeRole` on the role you provide. The role's trust policy must explicitly allow `bedrock-agentcore.amazonaws.com` to assume it.

**Fix — add to the IAM role's trust policy:**

```yaml
# CloudFormation: AssumeRolePolicyDocument in the agent task role
Statement:
  - Effect: Allow
    Principal:
      Service: ecs-tasks.amazonaws.com
    Action: sts:AssumeRole
  # Required: allows the registry crawler to assume this role
  # when crawling the MCP server URL
  - Effect: Allow
    Principal:
      Service: bedrock-agentcore.amazonaws.com
    Action: sts:AssumeRole
```

### Problem 2: SSE Transport Not Supported

**Error:** `SSE streams from MCP server is currently not supported.`

The registry crawler sends a one-shot HTTP POST and expects a plain JSON response back. FastMCP defaults to SSE (Server-Sent Events) responses — it opens a stream rather than returning a JSON body. The AWS docs [explicitly state](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/registry-sync-records.html) this is a public-preview limitation that applies to any MCP server.

**Fix — enable `json_response=True` on FastMCP:**

```python
# mcp_server.py — before (default, SSE):
mcp_app = mcp.http_app()

# mcp_server.py — after (plain JSON responses):
mcp_app = mcp.http_app(json_response=True)
```

The MCP client the agent uses (`streamablehttp_client`) negotiates response format via `Accept: application/json, text/event-stream` and handles both formats — so this change is fully backward compatible. Once AWS adds SSE support to the crawler, this setting can be removed.

### Problem 3: URL Extraction from Search Results

Once the MCP record was approved, the agent's startup search found the record but couldn't extract the URL. The `synchronizationConfiguration` field (which contains the original URL you provided) is **not returned** by `search_registry_records`.

The URL is stored by the registry in a different location — the MCP server manifest JSON that the crawler populated during crawling:

**What the registry stores in `descriptors.mcp.server.inlineContent`:**

```json
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "financial-tools-mcp",
  "version": "3.2.4",
  "remotes": [
    {
      "type": "streamable-http",
      "url": "https://1hybttfhe2.execute-api.us-east-1.amazonaws.com/mcp"
    }
  ]
}
```

**Fix — parse `remotes[0].url`:**

```python
def _discover_mcp_url() -> str:
    response = search_client.search_registry_records(
        registryIds=[registry_id],
        searchQuery="financial tools MCP server",
        maxResults=10,
    )
    for record in response.get("registryRecords", []):
        if record.get("descriptorType") == "MCP":
            inline = (
                record.get("descriptors", {})
                .get("mcp", {})
                .get("server", {})
                .get("inlineContent", "")
            )
            if inline:
                server_json = json.loads(inline)
                remotes = server_json.get("remotes", [])
                if remotes:
                    return remotes[0].get("url", "")
    return ""
```

---

## 5. The Agent

### Startup — Discover MCP URL Once

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    global _discovered_mcp_url
    # Query registry for MCP record → cache URL only.
    # No MCP connection at startup — tools are loaded selectively per request.
    _discovered_mcp_url = _discover_mcp_url()
    _build_static_tools()
    log.info("Agent ready. MCP URL: %s", _discovered_mcp_url or "(not discovered)")
    yield
```

### Per-Request Flow — Five Steps

The full per-request logic lives in `_run()` inside the `/invoke/stream` endpoint. Here is each step with the actual code:

#### Step 1: Semantic Search — All Record Types

```python
search_resp = search_client.search_registry_records(
    registryIds=[registry_id],
    searchQuery=req.message,   # the user's actual question
    maxResults=10,
)

all_records   = search_resp.get("registryRecords", [])
skill_records = [r for r in all_records if r.get("descriptorType") == "AGENT_SKILLS"]
```

The search returns all record types ranked together. We filter client-side to find AGENT_SKILLS records. The top skill record drives the rest of the request.

#### Step 2: Parse mcp_tools from Frontmatter

```python
def _parse_mcp_tools_from_frontmatter(skill_md: str) -> list[str]:
    """Read the mcp_tools: list from a SKILL.md YAML frontmatter block."""
    in_fm = False
    in_tools_block = False
    tool_names = []

    for line in skill_md.splitlines():
        stripped = line.strip()
        if stripped == "---":
            if not in_fm:
                in_fm = True
                continue
            else:
                break  # end of frontmatter
        if not in_fm:
            continue
        if stripped.startswith("mcp_tools:"):
            in_tools_block = True
            continue
        if in_tools_block:
            if stripped.startswith("- "):
                tool_names.append(stripped[2:].strip())
            elif stripped and not stripped.startswith("#"):
                in_tools_block = False

    return tool_names
```

This is called on the top skill's `inlineContent` before making any MCP connection — so we know exactly which tools to load before touching the server.

#### Step 3: Selective MCP Tool Loading

```python
def _get_selective_mcp_tools(declared_tool_names: list[str]) -> tuple[list, MCPClient]:
    """Connect to MCP server and return ONLY the tools this skill declared."""
    client = _make_sigv4_mcp_client(_discovered_mcp_url)
    client.start()
    all_tools = client.list_tools_sync()

    name_set = set(declared_tool_names)
    selected = [t for t in all_tools if t.tool_name in name_set]
    # e.g. declared=["get_financial_data", "get_kpi_benchmarks"]
    # all_tools may have more — we load only what this skill needs
    return selected, client
```

This is the selective loading pattern. An MCP server with 50 tools doesn't pollute the agent's context — each skill declares its dependencies and only those are loaded.

#### Step 4: Build Agent with Scoped Tool Set

```python
tools = [*_static_tools, *mcp_tools]
# _static_tools: [search_and_load_skill, file_read, python_exec]
# mcp_tools:     [get_financial_data, get_kpi_benchmarks]  (this request only)

agent = Agent(
    model=BedrockModel(model_id="us.anthropic.claude-sonnet-4-6", region_name="us-east-1"),
    tools=tools,
    system_prompt=_agent_system_prompt,
    callback_handler=cb,       # streams tool call steps to SSE
    messages=prior_messages,   # conversation history
)
```

A fresh agent is built per request with exactly the tools needed. The MCP client is also per-request — it's created, used, and stopped within the request lifecycle.

#### Step 5: Invoke and Stream

```python
result = agent(req.message)
# Strands agent calls search_and_load_skill → reads SKILL.md → follows instructions
# → calls MCP tools → calls python_exec → produces final answer
```

The `callback_handler` emits tool call events as SSE `step` events in real time. The UI renders these as the pipeline visualization.

### SigV4-Signed MCP Transport

The MCP server is behind API Gateway with AWS_IAM auth. Every request must be SigV4-signed with `service=execute-api`.

```python
def _make_sigv4_mcp_client(url: str) -> MCPClient:
    from streamable_http_sigv4 import streamablehttp_client_with_sigv4
    credentials = boto_session.get_credentials()
    return MCPClient(
        lambda u=url, c=credentials: streamablehttp_client_with_sigv4(
            url=u,
            credentials=c,
            service="execute-api",
            region="us-east-1",
        )
    )
```

The ECS task role has `execute-api:Invoke` permission scoped to the specific API Gateway resource. Credentials are sourced automatically from the task role — no keys, no secrets.

### The search_and_load_skill Tool

The agent has a `search_and_load_skill` tool that it can call during reasoning (in addition to the pre-flight search the framework does). This tool searches the registry and returns the skill's `SKILL.md` to the agent as context:

```python
@tool
def search_and_load_skill(query: str) -> str:
    """Search the AWS Agent Registry for a skill and load it locally."""
    response = search_client.search_registry_records(
        registryIds=[registry_id],
        searchQuery=query,
        maxResults=10,
    )
    skill_records = [r for r in response["registryRecords"]
                     if r.get("descriptorType") == "AGENT_SKILLS"]

    # Load full SKILL.md for the top match only (progressive disclosure)
    skill_dir, skill_md = load_skill_from_registry(response, record_index=0)

    return (
        f"Registry returned {len(skill_records)} candidate skill(s):\n"
        f"[candidate list...]\n\n"
        f"SKILL.md instructions:\n\n{skill_md}"
    )
```

---

## 6. The MCP Server

### Tool Definitions

```python
from fastmcp import FastMCP

mcp = FastMCP("financial-tools-mcp")

@mcp.tool()
def get_financial_data(period: str) -> dict:
    """Retrieve P&L financial data for a given quarter.
    
    Args:
        period: Quarter identifier, e.g. 'Q3 2025', 'Q2 2025', 'Q1 2025', 'Q4 2024'.
    """
    data = FINANCIAL_DATA.get(period)
    if data is None:
        return {"error": f"No data for '{period}'."}
    return {"period": period, **data}
    # Returns: {period, revenue, cogs, operating_expenses, ebitda}

@mcp.tool()
def get_kpi_benchmarks() -> dict:
    """Retrieve industry benchmark thresholds and formulas for financial KPIs."""
    return {
        "kpis": {
            "gross_margin_pct":    {"formula": "(Revenue - COGS) / Revenue * 100",
                                    "general_benchmark": 40.0},
            "ebitda_margin_pct":   {"formula": "EBITDA / Revenue * 100",
                                    "general_benchmark": 15.0},
            "opex_ratio":          {"formula": "Operating Expenses / Revenue * 100",
                                    "general_benchmark": 30.0, "higher_is_better": False},
            "revenue_growth_qoq":  {"formula": "(Current - Prior) / Prior * 100"},
        },
        "status_thresholds": {
            "GREEN":  "At or above general_benchmark",
            "YELLOW": "Within 5pp below general_benchmark",
            "RED":    "More than 5pp below general_benchmark",
        },
    }
```

### Starlette Wrapper with Lifespan

```python
# json_response=True required for registry URL-sync crawler (SSE not yet supported)
mcp_app = mcp.http_app(json_response=True)

app = Starlette(
    routes=[
        Route("/health", health),   # ALB health check
        Mount("/", app=mcp_app),    # MCP endpoint at /mcp
    ],
    lifespan=mcp_app.lifespan,      # required — initialises FastMCP session manager
)
```

Two things are critical here:
- `json_response=True` — enables registry crawling (plain JSON instead of SSE stream)
- `lifespan=mcp_app.lifespan` — without this, every request raises `StreamableHTTPSessionManager task group was not initialized`

---

## 7. Infrastructure

### Overview

```
Internet
    │
CloudFront (d15k2eii8qdkf8.cloudfront.net)
    ├── Default (/*) → S3 bucket (index.html)
    └── /chat, /chat/stream → Chat ALB (public)
                                    │
                              Chat ECS (private subnet)
                                    │
                              Agent ALB (internal)
                                    │
                              Agent ECS (private subnet)
                               │          │
                         Registry      API Gateway (HTTPS)
                         (AWS API)         │
                           via           VPC Link
                          NAT GW            │
                               │        MCP ALB (internal)
                               │            │
                          Bedrock       MCP ECS (private subnet)
                          (AWS API)
                           via
                          NAT GW
```

### VPC + Private Subnets

All ECS tasks run in private subnets with no public IPs. Two private subnets across two availability zones for the ECS tasks; one public subnet for the NAT Gateway.

### Why API Gateway + VPC Link

The registry crawler is AWS-managed — it lives outside the VPC and cannot resolve internal ALB DNS names. To give it an HTTPS endpoint that routes to the internal MCP server:

1. Create an **API Gateway HTTP API**
2. Create a **VPC Link** to the private subnet
3. Add a route: `ANY /mcp/{proxy+}` → VPC Link → MCP ALB

The MCP server's endpoint becomes:
`https://1hybttfhe2.execute-api.us-east-1.amazonaws.com/mcp`

This is IAM-protected with `AWS_IAM` auth on the API Gateway route. Only callers with `execute-api:Invoke` permission (agent task role, registry crawler via assumed role) can invoke it.

### CloudFormation — Key Resources

```yaml
# API Gateway HTTP API
McpApi:
  Type: AWS::ApiGatewayV2::Api
  Properties:
    Name: financial-agent-mcp-api
    ProtocolType: HTTP

# VPC Link — bridges API GW to private subnet
McpVpcLink:
  Type: AWS::ApiGatewayV2::VpcLink
  Properties:
    Name: financial-agent-mcp-vpc-link
    SubnetIds: [PrivateSubnetA, PrivateSubnetB]
    SecurityGroupIds: [McpVpcLinkSg]

# Route — IAM auth, forward to MCP ALB via VPC Link
McpApiRoute:
  Type: AWS::ApiGatewayV2::Route
  Properties:
    ApiId: !Ref McpApi
    RouteKey: "ANY /mcp/{proxy+}"
    AuthorizationType: AWS_IAM
    Target: !Sub "integrations/${McpApiIntegration}"
```

### NAT Gateway — Why It Exists

The agent ECS task calls outbound to two AWS service APIs:
- `bedrock-agentcore` — registry search (`search_registry_records`)
- `bedrock-runtime` — Bedrock model inference

Private subnets have no internet route by default. A NAT Gateway in the public subnet provides outbound internet access so those API calls can reach the AWS service endpoints.

**Production alternative:** Configure VPC PrivateLink endpoints for `bedrock-agentcore` and `bedrock-runtime`. Traffic stays on the AWS backbone, the NAT Gateway is no longer needed, and the private subnets are fully locked down with no internet dependency.

### IAM — Agent Task Role

The agent task role has four permission boundaries:

```yaml
Policies:
  - bedrock-agentcore:SearchRegistryRecords
    bedrock-agentcore:GetRegistryRecord
    bedrock-agentcore:ListRegistries
    bedrock-agentcore:GetRegistry
    Resource: "*"

  - bedrock:InvokeModel
    bedrock:InvokeModelWithResponseStream
    Resource: "arn:aws:bedrock:*::foundation-model/*"
              "arn:aws:bedrock:*:{account}:inference-profile/*"

  - execute-api:Invoke
    Resource: "arn:aws:execute-api:{region}:{account}:{McpApi}/*"

  - s3:GetObject
    s3:ListBucket
    Resource: "{SkillsBucket}/*"
```

**Trust policy — two principals:**

```yaml
AssumeRolePolicyDocument:
  Statement:
    - Effect: Allow
      Principal:
        Service: ecs-tasks.amazonaws.com   # ECS task assumes this role
      Action: sts:AssumeRole
    - Effect: Allow
      Principal:
        Service: bedrock-agentcore.amazonaws.com   # registry crawler assumes this role
      Action: sts:AssumeRole                       # to sign MCP crawl requests
```

The second trust entry is what enables `synchronizationType=URL` with `credentialProviderType=IAM` to work.

---

## 8. What We Had to Figure Out

These were the non-obvious problems that required debugging API responses and library source code — none were documented end-to-end.

### 1. Registry Crawler Trust Policy

The `credentialProviderConfigurations` block in `CreateRegistryRecord` specifies which IAM role the crawler should assume. The assumption fails unless `bedrock-agentcore.amazonaws.com` is explicitly listed in the role's trust policy. This isn't mentioned in the error message or in the `CreateRegistryRecord` API docs — the error is `Unable to assume the provided IAM role`.

### 2. FastMCP SSE vs. JSON

AWS Agent Registry docs state: *"At public preview launch, SSE stream from MCP server is not supported yet."* FastMCP defaults to SSE responses. The fix is `json_response=True`. This is not documented anywhere as a recipe — it's the correct inference from the documented constraint.

This limitation applies to all MCP servers regardless of whether they're internal or internet-facing. A public MCP server on the internet with default FastMCP settings would hit the same error.

### 3. URL Extraction from Search Results

`search_registry_records` and `get_registry_record` return different shapes:
- `get_registry_record` includes `synchronizationConfiguration.fromUrl.url`
- `search_registry_records` does **not** include `synchronizationConfiguration`

The URL the registry discovers during crawling is stored in the MCP server manifest under `descriptors.mcp.server.inlineContent` as a JSON string following the `modelcontextprotocol.io/schemas/2025-12-11` schema. The URL is at `remotes[0].url`. This required inspecting the actual API response to discover.

### 4. FastMCP Lifespan

When wrapping FastMCP inside a Starlette app, the Starlette app must receive `lifespan=mcp_app.lifespan`. Without it, the `StreamableHTTPSessionManager` task group is never initialized and every request raises a `RuntimeError`. FastMCP's error message is excellent here — it explains exactly what to do — but it only surfaces at runtime when the first request hits.

---

## 9. End-to-End Data Flow

### Agent Startup (once per ECS task)

```
Agent ECS starts
    │
    └─► search_registry_records("financial tools MCP server")
              │
              ◄── registry returns MCP record
              │
    └─► parse descriptors.mcp.server.inlineContent → remotes[0].url
              │
    _discovered_mcp_url = "https://1hybttfhe2.execute-api.us-east-1.amazonaws.com/mcp"
              │
    Agent ready
```

### Per Request

```
User: "Calculate KPIs for Q3 2025"
    │
[Pre-flight — framework, before Strands agent runs]
    │
    ├─1─► search_registry_records("Calculate KPIs for Q3 2025")
    │         ◄── 6 records ranked: quarterly-kpi-calculator top match
    │
    ├─2─► parse mcp_tools from quarterly-kpi-calculator frontmatter
    │         → ["get_financial_data", "get_kpi_benchmarks"]
    │
    ├─3─► MCPClient.start() → API GW → VPC Link → MCP ALB → MCP container
    │     list_tools_sync() → filter to declared tools only
    │
    ├─4─► emit SSE "search" event → UI renders registry search panel
    │
[Strands Agent runs]
    │
    ├─5─► agent calls search_and_load_skill("Calculate KPIs for Q3 2025")
    │         ◄── SKILL.md returned with 5-step procedure
    │
    ├─6─► agent calls get_kpi_benchmarks()
    │         → API GW (SigV4) → VPC Link → MCP → returns benchmarks JSON
    │
    ├─7─► agent calls get_financial_data(period="Q3 2025")
    │         → API GW (SigV4) → VPC Link → MCP → returns P&L dict
    │
    ├─8─► agent calls get_financial_data(period="Q2 2025")  [for QoQ growth]
    │
    ├─9─► agent calls python_exec(code="...KPI calculations...")
    │         → subprocess in agent container → returns calculated values
    │
    ├─10► Bedrock (Claude Sonnet 4.6) composes final answer
    │
    └─11► SSE "result" event → chat UI renders formatted KPI table
```

---

## 10. Registry Record Reference

### Records in the Registry (all APPROVED)

| Record ID | Type | Name | Tools / Skills |
|---|---|---|---|
| `ZaczjktQMqu1` | MCP | financial-tools-mcp | `get_financial_data`, `get_kpi_benchmarks` |
| `yi0mFgdvv2It` | AGENT_SKILLS | quarterly-kpi-calculator | KPIs: Gross Margin, EBITDA, OpEx, QoQ Growth |
| `omV8fUKfMHFA` | AGENT_SKILLS | cost-efficiency-analyzer | Cost structure, COGS ratios, OpEx trends |
| `f0DbmGOKIiTB` | AGENT_SKILLS | revenue-growth-analyst | Revenue trends, growth rate analysis |
| `Qib3aGFdjEUN` | AGENT_SKILLS | multi-quarter-trend-analysis | 4-quarter trend narrative |
| `wvsoX1mKTuM9` | AGENT_SKILLS | executive-financial-briefing | One-page CFO briefing, 5 structured sections |

### Registry ARN
```
arn:aws:bedrock-agentcore:us-east-1:336468392708:registry/0ybA4HzNZgVlIJl6
```

### MCP Server Endpoint
```
https://1hybttfhe2.execute-api.us-east-1.amazonaws.com/mcp
```
Routes via API Gateway → VPC Link → internal ALB → MCP ECS container (port 8080).

### Key API Operations Used

| Operation | Client | Purpose |
|---|---|---|
| `create_registry` | `bedrock-agentcore-control` | Create the registry |
| `create_registry_record` | `bedrock-agentcore-control` | Publish MCP or AGENT_SKILLS record |
| `submit_registry_record_for_approval` | `bedrock-agentcore-control` | Move record to PENDING_APPROVAL |
| `update_registry_record_status` | `bedrock-agentcore-control` | Approve or reject a record |
| `get_registry_record` | `bedrock-agentcore-control` | Get full record details (includes synchronizationConfiguration) |
| `search_registry_records` | `bedrock-agentcore` | Semantic search across all record types |
| `list_registry_records` | `bedrock-agentcore-control` | List all records in a registry |
| `delete_registry_record` | `bedrock-agentcore-control` | Delete a record |
