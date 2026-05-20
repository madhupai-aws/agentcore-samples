# How the Agent Works — Code Snippets & Sequence

*A technical reference for replicating the AWS Agent Registry + Strands Agent pattern.*

---

## Part 1 — Key Code Snippets

The five snippets below answer the core question: **how does the agent find skills, load tools, and call MCP servers?**

---

### 1. Discovering the MCP Server URL from the Registry at Startup

The agent queries the registry once when the ECS task starts. No hardcoded URL — it reads the URL the registry stored when it crawled the MCP server at registration time.

```python
def _discover_mcp_url() -> str:
    registry_id = REGISTRY_ARN.split("/")[-1]
    response = search_client.search_registry_records(
        registryIds=[registry_id],
        searchQuery="financial tools MCP server",
        maxResults=10,
    )
    for record in response.get("registryRecords", []):
        if record.get("descriptorType") == "MCP":
            # Registry crawled the MCP server at registration time and stored
            # its manifest in descriptors.mcp.server.inlineContent.
            # The URL is at: JSON → remotes[0].url
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

### 2. Per-Request Registry Search (Phase 1 Pre-flight)

Every user message triggers a fresh registry search. The raw user text is the query — no intent classification, no hardcoded routing. The registry's vector search does the matching.

```python
# Phase 1 — runs before the Strands agent is invoked

registry_id = REGISTRY_ARN.split("/")[-1]
search_resp = search_client.search_registry_records(
    registryIds=[registry_id],
    searchQuery=req.message,    # ← raw user message, no pre-processing
    maxResults=10,
)

all_records   = search_resp.get("registryRecords", [])
skill_records = [r for r in all_records
                 if r.get("descriptorType") == "AGENT_SKILLS"]

# skill_records[0] is the highest-ranked skill for this query
```

---

### 3. Parsing `mcp_tools:` from SKILL.md Frontmatter

The SKILL.md frontmatter declares exactly which MCP tools that skill needs. This is a custom convention — the registry stores the SKILL.md text opaquely and the agent code parses it.

```python
def _parse_mcp_tools_from_frontmatter(skill_md: str) -> list[str]:
    """
    Reads the mcp_tools: block from YAML frontmatter, e.g.:
        ---
        name: quarterly-kpi-calculator
        mcp_tools:
          - get_financial_data
          - get_kpi_benchmarks
        ---
    Returns: ['get_financial_data', 'get_kpi_benchmarks']
    """
    in_fm = False
    in_tools_block = False
    tool_names: list[str] = []

    for line in skill_md.splitlines():
        stripped = line.strip()
        if stripped == "---":
            if not in_fm:
                in_fm = True; continue
            else:
                break                         # end of frontmatter
        if not in_fm:
            continue
        if stripped.startswith("mcp_tools:"):
            in_tools_block = True; continue
        if in_tools_block:
            if stripped.startswith("- "):
                tool_names.append(stripped[2:].strip())
            elif stripped and not stripped.startswith("#"):
                in_tools_block = False        # next top-level key

    return tool_names
```

---

### 4. Connecting to the MCP Server and Loading Only Declared Tools

Opens a fresh MCP connection per request, lists all available tools on the server, then filters to only the ones the skill declared. If the server exposes 50 tools, only 2 are loaded into this request's context.

```python
def _get_selective_mcp_tools(declared_tool_names: list[str]):
    # Fresh connection per request (FastMCP sessions expire on idle)
    client = _make_sigv4_mcp_client(_discovered_mcp_url)
    client.start()
    all_tools = client.list_tools_sync()        # GET /mcp/tools/list

    name_set = set(declared_tool_names)
    selected = [t for t in all_tools if t.tool_name in name_set]

    # Example: server has [get_financial_data, get_kpi_benchmarks]
    # declared_tool_names = ['get_financial_data', 'get_kpi_benchmarks']
    # selected = both tools (in a larger server, only these 2 would be loaded)

    return selected, client
```

The MCP transport uses SigV4 because the server sits behind API Gateway with IAM auth:

```python
def _make_sigv4_mcp_client(url: str) -> MCPClient:
    from streamable_http_sigv4 import streamablehttp_client_with_sigv4
    credentials = boto_session.get_credentials()
    return MCPClient(
        lambda: streamablehttp_client_with_sigv4(
            url=url,
            credentials=credentials,
            service="execute-api",   # API Gateway SigV4
            region=AWS_REGION,
        )
    )
```

---

### 5. Building the Strands Agent and Invoking Bedrock

The agent is built fresh per request with 3 base tools + however many MCP tools the skill declared. A single call to `a(req.message)` hands control to the Strands reasoning loop.

```python
# tools = 3 base tools + selective MCP tools loaded in Phase 1
tools = [
    search_and_load_skill,   # searches registry + loads full SKILL.md into context
    file_read,               # reads local files
    python_exec,             # runs Python in a subprocess
    *mcp_tools,              # e.g. [get_financial_data, get_kpi_benchmarks]
]

a = Agent(
    model=BedrockModel(model_id="us.anthropic.claude-sonnet-4-6"),
    tools=tools,
    system_prompt=_agent_system_prompt,
    callback_handler=cb,     # streams step events to the UI via SSE
)

# Phase 2 starts here — Strands + Bedrock take over.
# Bedrock will call search_and_load_skill first (gets full SKILL.md procedure),
# then follow the procedure by calling MCP tools and python_exec as instructed.
r = a(req.message)
```

`search_and_load_skill` — the tool Bedrock calls first inside Phase 2 to get the full skill procedure:

```python
@tool
def search_and_load_skill(query: str) -> str:
    """Searches the registry and returns the full SKILL.md for the top match."""
    response = search_client.search_registry_records(
        registryIds=[registry_id],
        searchQuery=query,
        maxResults=10,
    )
    skill_records = [r for r in response["registryRecords"]
                     if r.get("descriptorType") == "AGENT_SKILLS"]

    # Load full SKILL.md only for the top match (progressive disclosure)
    skill_md = (skill_records[0]
                ["descriptors"]["agentSkills"]["skillMd"]["inlineContent"])

    # Returns the full procedure text into the agent's context
    return f"SKILL.md instructions:\n\n{skill_md}"
```

---

### Quick Reference

| What | Function | Key point |
|------|----------|-----------|
| MCP URL discovery | `_discover_mcp_url()` | Read from registry record at startup, not from config/SSM |
| Skill matching | `search_registry_records(req.message)` | Raw user text → vector search → ranked results |
| Tool selection | `_parse_mcp_tools_from_frontmatter()` | Custom convention in SKILL.md frontmatter — registry stores it opaquely |
| MCP connection | `_get_selective_mcp_tools()` | Fresh per-request, filtered to declared tools only |
| Agent invocation | `Agent(tools=...) → a(req.message)` | Strands SDK; Bedrock handles the reasoning loop |
| Skill procedure loading | `search_and_load_skill` tool | Second registry search inside Phase 2 — full SKILL.md loaded on demand |

---

## Part 2 — End-to-End Sequence for a Finance Report

Prompt: **"Calculate KPIs for Q3 2025"**

---

### Phase 0 — Before the User Sends Anything (ECS Task Startup)

1. Agent starts → calls `search_registry_records("financial tools MCP server")`
2. Finds the MCP record → parses `remotes[0].url` from the stored manifest → caches the MCP server URL
3. Agent is now idle, waiting. No MCP connection open. No skills loaded.

---

### Phase 1 — Pre-flight (Framework Code, Before Bedrock Is Called)

**Step 1 — Registry search**

User sends "Calculate KPIs for Q3 2025" → agent calls:
```
search_registry_records(query="Calculate KPIs for Q3 2025")
```
Registry does a vector search across all 6 records → returns ranked list → top AGENT_SKILLS match = `quarterly-kpi-calculator`

**Step 2 — Parse tool dependencies**

Agent reads the frontmatter of that skill record:
```yaml
mcp_tools:
  - get_financial_data
  - get_kpi_benchmarks
```

**Step 3 — Load only those MCP tools**

Opens MCP connection → calls `list_tools_sync()` → filters to just those 2 tools

**Step 4 — Build the agent**

```python
Agent(tools=[search_and_load_skill, file_read, python_exec,
             get_financial_data, get_kpi_benchmarks])
```
5 tools total. First Bedrock call happens now.

---

### Phase 2 — Strands Agent Reasoning Loop (Bedrock Drives This)

**Turn 1** — Bedrock decides: call `search_and_load_skill("Calculate KPIs for Q3 2025")`
→ Second registry search → loads full SKILL.md procedure into context
→ Agent now has step-by-step instructions: *fetch benchmarks → fetch P&L → calculate → format*

**Turn 2** — Bedrock follows Step 1 of SKILL.md: call `get_kpi_benchmarks()`
→ MCP server returns: `{gross_margin: 40%, ebitda: 15%, opex: 30%}`

**Turn 3** — Bedrock follows Step 2: call `get_financial_data(period="Q3 2025")`
→ MCP server returns: `{revenue: 4.2M, cogs: 1.89M, opex: 1.05M, ebitda: 1.26M}`

**Turn 4** — Bedrock follows Step 2 again: call `get_financial_data(period="Q2 2025")`
→ Needed for QoQ growth calculation

**Turn 5** — Bedrock follows Step 3: call `python_exec(code="gross_margin = ...")`
→ Subprocess runs in agent container → returns: `GM=55%, EBITDA=30%, OpEx=25%, Growth=+10.5%`

**Turn 6** — Bedrock has all results → formats KPI table with GREEN/YELLOW/RED benchmark status → final answer

---

### The One-Liner

```
User prompt
  → registry search (find skill)
    → parse frontmatter (find tools)
      → MCP connect (load tools)
        → Bedrock turn 1: load full SKILL.md
        → Bedrock turn 2–4: call MCP tools per skill instructions
        → Bedrock turn 5: python_exec to calculate
        → Bedrock turn 6: format and return answer
```

**Key insight:** The agent has no financial logic in it. The skill procedure — what to fetch, how to calculate, how to format — all comes from the SKILL.md stored in the registry. The agent is a generic executor. The registry is the control plane.

---

### Two-Phase Summary

| Phase | Who runs it | What happens |
|-------|-------------|--------------|
| **Phase 1 — Pre-flight** | Framework code (before Bedrock) | Registry search → parse frontmatter → load MCP tools → build Agent |
| **Phase 2 — Strands loop** | Bedrock (Claude Sonnet 4.6) | Load full SKILL.md → follow procedure → call MCP tools → calculate → answer |

The split exists because Strands requires tools to be present when the Agent is constructed. MCP tools cannot be added mid-reasoning, so they must be loaded before the first Bedrock call.
