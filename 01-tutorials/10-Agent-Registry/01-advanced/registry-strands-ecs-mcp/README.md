# AWS Agent Registry + Strands Agent — Financial Skills Demo

A reference implementation of a **dynamic, registry-driven AI agent** built on AWS. The agent has no hardcoded skills or tools. It discovers what it can do at runtime by querying the **AWS Agent Registry** (Amazon Bedrock AgentCore), loading skills as instructions and connecting to an MCP server for live tool execution.

The financial analysis domain is the vehicle. The architecture is the point.

---

## Core Idea

> **Separate what an agent knows how to do from the agent itself.**

Skills, procedures, and tool dependencies are stored in the AWS Agent Registry as `SKILL.md` files. The agent is a generic executor. You can add or update skills without touching agent code or redeploying.

```
User prompt
  → Registry search (find the right skill)
    → Parse SKILL.md frontmatter (which MCP tools are needed)
      → Load only those tools from the MCP server
        → Bedrock reasoning loop (follow the skill procedure)
          → Final answer
```

---

## Architecture

```
Internet
    │
CloudFront
    ├── /* → S3 (index.html)
    └── /chat, /chat/stream → Chat ALB (public)
                                    │
                              Chat ECS (private subnet)
                                    │
                              Agent ALB (internal)
                                    │
                              Agent ECS (private subnet)
                               │          │
                         Registry      API Gateway (HTTPS + SigV4)
                         (AWS API)         │
                           via           VPC Link
                          NAT GW            │
                               │        MCP ALB (internal)
                               │            │
                          Bedrock       MCP ECS (private subnet)
```

| Component | Technology | Role |
|---|---|---|
| AWS Agent Registry | Amazon Bedrock AgentCore | Stores skills and MCP server schemas; serves semantic search |
| Strands Agent | Python / Strands SDK | Per-request: search registry → read skill → load tools → invoke Bedrock |
| MCP Server | FastMCP / Python | Exposes `get_financial_data` and `get_kpi_benchmarks` tools |
| Chat Interface | FastAPI + SSE | Browser-facing proxy; streams agent step events to the UI |
| Frontend | React (single HTML) | Chat UI with registry search visualization and pipeline steps |
| Infrastructure | ECS Fargate, API GW, VPC, CloudFormation | Private VPC; API Gateway bridges internal MCP server to the registry crawler |

---

## Repository Layout

```
deploy/
  agent/          Strands agent (FastAPI, Strands SDK, SigV4 MCP transport)
  chat/           Chat interface (FastAPI, SSE proxy, React frontend)
  mcp/            MCP server (FastMCP, financial data tools)
  infra/
    cfn.yaml            CloudFormation stack (VPC, ECS, ALBs, API GW, IAM, Cognito)
    setup.py            One-time registry + S3 setup after stack creation
    register_skills.py  Add new AGENT_SKILLS records to an existing registry

my_skills/
  quarterly-kpi-calculator/     KPI calculation (Gross Margin, EBITDA, OpEx, QoQ Growth)
  cost-efficiency-analyzer/     Cost structure and expense efficiency analysis
  revenue-growth-analyst/       Top-line revenue growth deep-dive
  multi-quarter-trend-analysis/ 4-quarter trend narrative
  executive-financial-briefing/ One-page CFO/board briefing
```

---

## How It Works

### Two-Phase Request Flow

Every user message triggers two phases before a response is returned.

**Phase 1 — Pre-flight (framework code, before Bedrock is called)**

1. **Registry search** — the raw user message is used as the search query. The registry returns all record types (AGENT_SKILLS, MCP, A2A, CUSTOM) ranked by vector similarity. The agent filters client-side to `AGENT_SKILLS` records. No intent classification, no hardcoded routing. Note: the MCP server URL is **not** discovered here — it was cached at ECS task startup via a separate one-time search.
2. **Parse frontmatter** — the top-ranked `AGENT_SKILLS` record contains a `SKILL.md`. The agent reads the `mcp_tools:` list from its YAML frontmatter to find out which MCP tools this skill needs.
3. **Selective MCP tool loading** — opens a connection to the MCP server, lists all tools, and loads *only* the declared ones. An MCP server with 50 tools does not pollute the agent's context.
4. **Build the Strands agent** — constructs the agent with 3 base tools + the skill's MCP tools.

**Phase 2 — Strands reasoning loop (Bedrock drives this)**

5. **Load full SKILL.md** — Bedrock's first tool call is `search_and_load_skill`, which performs a second registry search and returns the complete skill procedure into context.
6. **Follow the procedure** — Bedrock calls the MCP tools and `python_exec` as instructed by the SKILL.md steps.
7. **Return answer** — results are streamed to the UI via SSE.

The split exists because Strands requires tools to be present when the `Agent` is constructed. MCP tools cannot be added mid-reasoning.

### SKILL.md Format

Each skill is a Markdown file with YAML frontmatter. The frontmatter is parsed programmatically; the body is read by Bedrock as instructions.

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

# Quarterly KPI Calculator

## Steps

### Step 1: Retrieve benchmark thresholds
Call the get_kpi_benchmarks tool to get current KPI formulas and benchmark values.

### Step 2: Get P&L data
If the user specified a quarter (e.g. "Q3 2025"), call:
    get_financial_data(period="Q3 2025")
...
```

The `mcp_tools:` field is the key convention — it declares exactly which tools the skill needs at request time.

### MCP Server URL — When It Is Written and When It Is Read

**Written once by `setup.py`** — when `create_registry_record()` is called with `synchronizationType="URL"`, the registry crawler assumes the provided IAM role, fetches the MCP server at the API Gateway URL, and stores the full MCP server manifest as inline content in the record (`descriptors.mcp.server.inlineContent`). That manifest includes `remotes[0].url` — the URL the crawler fetched. This happens once at registration time and does not repeat.

**Read once at ECS task startup** — `_discover_mcp_url()` searches the registry with the hardcoded query `"financial tools MCP server"`, finds the MCP record, and parses `remotes[0].url` from the stored manifest. The URL is cached in `_discovered_mcp_url` for the life of the task. This is a read — the URL was already in the registry from registration.

**Not involved in per-request search** — Phase 1's registry search uses the user's message as the query and filters to `AGENT_SKILLS` records only. It does not touch the MCP record or the URL.

**Used per-request for every tool call** — when Phase 1 determines which MCP tools are needed, `_get_selective_mcp_tools()` creates a fresh `MCPClient` using the cached URL, calls `list_tools_sync()` to get the available tools, and returns proxy objects bound to that client. When Bedrock then calls `get_financial_data` or `get_kpi_benchmarks` during the reasoning loop, the Strands SDK dispatches each call as a SigV4-signed HTTP POST to the MCP server at that same cached URL. The tools do not execute locally — every call crosses the network to the MCP container.

```
setup.py (once, after CloudFormation deploy)
  └─► create_registry_record(synchronizationType="URL", url=<api-gw-url>)
        └─► registry crawler fetches the MCP server
              └─► stores manifest + URL in descriptors.mcp.server.inlineContent
                    └─► record approved → searchable

ECS task startup (once per task)
  └─► search_registry_records("financial tools MCP server")
        └─► reads remotes[0].url from the stored manifest → caches in _discovered_mcp_url

Per-request Phase 1
  └─► search_registry_records(user message) → AGENT_SKILLS only → no MCP URL involved
  └─► _get_selective_mcp_tools(_discovered_mcp_url)
        └─► MCPClient.start() → HTTP connection to MCP server at cached URL
        └─► list_tools_sync() → filter to declared tools → return proxy objects

Per-request Phase 2 (each MCP tool call by Bedrock)
  └─► Bedrock calls get_financial_data(period="Q3 2025")
        └─► Strands dispatches via MCPClient → SigV4-signed POST to cached URL
              └─► MCP server executes tool → returns result → back to Bedrock
```

---

## Prerequisites

- AWS account with Amazon Bedrock AgentCore (public preview) enabled in `us-east-1`
- Access to `us.anthropic.claude-sonnet-4-6` (cross-region inference profile)
- Docker and AWS CLI configured
- Three ECR repositories for the agent, chat, and MCP server images

---

## Deployment

### 1. Build and Push Docker Images

```bash
AWS_ACCOUNT=<your-account-id>
REGION=us-east-1

# MCP server
docker build -t $AWS_ACCOUNT.dkr.ecr.$REGION.amazonaws.com/financial-agent-mcp:latest deploy/mcp/
docker push $AWS_ACCOUNT.dkr.ecr.$REGION.amazonaws.com/financial-agent-mcp:latest

# Strands agent
docker build -t $AWS_ACCOUNT.dkr.ecr.$REGION.amazonaws.com/financial-agent-agent:latest deploy/agent/
docker push $AWS_ACCOUNT.dkr.ecr.$REGION.amazonaws.com/financial-agent-agent:latest

# Chat interface
docker build -t $AWS_ACCOUNT.dkr.ecr.$REGION.amazonaws.com/financial-agent-chat:latest deploy/chat/
docker push $AWS_ACCOUNT.dkr.ecr.$REGION.amazonaws.com/financial-agent-chat:latest
```

### 2. Deploy the CloudFormation Stack

```bash
aws cloudformation deploy \
  --template-file deploy/infra/cfn.yaml \
  --stack-name financial-agent \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    McpImageUri=$AWS_ACCOUNT.dkr.ecr.$REGION.amazonaws.com/financial-agent-mcp:latest \
    AgentImageUri=$AWS_ACCOUNT.dkr.ecr.$REGION.amazonaws.com/financial-agent-agent:latest \
    ChatImageUri=$AWS_ACCOUNT.dkr.ecr.$REGION.amazonaws.com/financial-agent-chat:latest
```

The stack creates: VPC with private/public subnets, NAT Gateway, ECS Fargate cluster, three ECS services (MCP/Agent/Chat), internal ALBs, API Gateway + VPC Link (for registry crawler access to MCP), S3 skills bucket, Cognito User Pool, IAM roles, and CloudWatch log groups.

### 3. Start the MCP Service

The MCP service must be running before registry setup because the registry crawler will attempt to crawl it.

```bash
aws ecs update-service \
  --cluster financial-agent-cluster \
  --service financial-agent-mcp \
  --desired-count 1
```

### 4. Run One-Time Registry Setup

> **Why a Python script and not CloudFormation?**
> The AWS Agent Registry (`bedrock-agentcore-control`) does not have CloudFormation resource types yet. All registry operations — creating the registry, publishing records, and approving them — are done via direct boto3 API calls in `setup.py`. CloudFormation handles all the infrastructure; `setup.py` is the stand-in until native CloudFormation support is available.
>
> **Approval workflow:** The registry supports a multi-step approval lifecycle (`DRAFT → PENDING_APPROVAL → APPROVED`). This project does not use a human approval gate — `setup.py` and `register_skills.py` call `submit_registry_record_for_approval()` and `update_registry_record_status(status="APPROVED")` back-to-back, so records are approved immediately by the script itself.

Get stack outputs first:

```bash
aws cloudformation describe-stacks \
  --stack-name financial-agent \
  --query "Stacks[0].Outputs"
```

Then run setup:

```bash
python deploy/infra/setup.py \
  --region us-east-1 \
  --bucket <SkillsBucketName from outputs> \
  --apigw-url <McpApiGwUrl from outputs>
```

This will:
1. Upload skill artifacts to S3
2. Create the AWS Agent Registry via `bedrock-agentcore-control` boto3 client
3. Publish the MCP record (`synchronizationType="URL"`) — registry crawls the API GW URL to auto-populate tool schemas — then immediately auto-approve it
4. Publish the `quarterly-kpi-calculator` AGENT_SKILLS record, then immediately auto-approve it
5. Write `REGISTRY_ARN` and `SKILLS_BUCKET` to SSM Parameter Store

### 5. Register Additional Skills

```bash
python deploy/infra/register_skills.py \
  --registry-arn <REGISTRY_ARN from setup output> \
  --region us-east-1
```

Registers all skills in `my_skills/` not already in the registry. Safe to re-run — skips existing records.

### 6. Start Agent and Chat Services

```bash
aws ecs update-service --cluster financial-agent-cluster --service financial-agent-agent --desired-count 1
aws ecs update-service --cluster financial-agent-cluster --service financial-agent-chat --desired-count 1
```

Access the chat UI at the `ChatEndpoint` URL from the stack outputs.

---

## Non-Obvious Setup Issues

Three problems had to be solved that are not documented end-to-end anywhere.

### 1. IAM Trust Policy for the Registry Crawler

The registry crawler calls `sts:AssumeRole` on the role you specify in `credentialProviderConfigurations`. The role's trust policy must explicitly allow `bedrock-agentcore.amazonaws.com` as a principal — otherwise the error is the opaque `Unable to assume the provided IAM role`.

```yaml
# In the agent task role's AssumeRolePolicyDocument:
- Effect: Allow
  Principal:
    Service: bedrock-agentcore.amazonaws.com
  Action: sts:AssumeRole
```

### 2. FastMCP Must Use `json_response=True`

The registry crawler sends a one-shot HTTP POST and expects a plain JSON response. FastMCP defaults to SSE (Server-Sent Events), which the crawler cannot parse. The AWS docs note SSE is not supported at public preview. Enable JSON responses:

```python
# deploy/mcp/mcp_server.py
mcp_app = mcp.http_app(json_response=True)  # required for registry URL-sync crawling
```

The `streamablehttp_client` used by the agent negotiates format via `Accept` headers, so this is fully backward compatible.

### 3. MCP URL Is in the Crawled Manifest, Not in Search Results

`search_registry_records` does **not** return `synchronizationConfiguration` (which contains the original URL you provided). The URL is stored by the crawler in the MCP server manifest at `descriptors.mcp.server.inlineContent` as a JSON string. Parse it like this:

```python
inline = record["descriptors"]["mcp"]["server"]["inlineContent"]
server_json = json.loads(inline)
url = server_json["remotes"][0]["url"]
```

### 4. FastMCP Lifespan Must Be Passed to Starlette

```python
app = Starlette(
    routes=[...],
    lifespan=mcp_app.lifespan,  # required — omitting this causes RuntimeError on every request
)
```

---

## Skills Reference

| Skill | Trigger phrases | MCP tools used |
|---|---|---|
| `quarterly-kpi-calculator` | "calculate KPIs", "gross margin", "EBITDA", "Q3 2025" | `get_financial_data`, `get_kpi_benchmarks` |
| `cost-efficiency-analyzer` | "cost breakdown", "are we spending too much", "COGS ratio" | `get_financial_data`, `get_kpi_benchmarks` |
| `revenue-growth-analyst` | "revenue growth", "top-line", "growth trajectory" | `get_financial_data` |
| `multi-quarter-trend-analysis` | "show me the trend", "how are we trending", "4 quarters" | `get_financial_data`, `get_kpi_benchmarks` |
| `executive-financial-briefing` | "briefing", "executive summary", "board update", "how is the business doing" | `get_financial_data`, `get_kpi_benchmarks` |

---

## Registry API Quick Reference

```python
# Two clients — control plane and data plane
registry_client = boto3.client("bedrock-agentcore-control")  # create/approve/delete
search_client   = boto3.client("bedrock-agentcore")          # semantic search

# Record lifecycle: CREATING → DRAFT → PENDING_APPROVAL → APPROVED
registry_client.create_registry_record(...)
registry_client.submit_registry_record_for_approval(...)
registry_client.update_registry_record_status(status="APPROVED", ...)

# Search — returns all record types ranked by vector similarity
search_client.search_registry_records(
    registryIds=[registry_id],
    searchQuery="calculate quarterly KPIs",
    maxResults=10,
)
```

---

## Further Reading

- [HOW_IT_WORKS.md](HOW_IT_WORKS.md) — key code snippets and end-to-end sequence diagram
- [TECHNICAL_DEEP_DIVE.md](TECHNICAL_DEEP_DIVE.md) — full technical reference including all API calls, infrastructure rationale, and debugging notes
- [DEMO_SCRIPT.md](DEMO_SCRIPT.md) — walkthrough script for demo sessions
- [architecture.mmd](architecture.mmd) — Mermaid source for the architecture diagram
