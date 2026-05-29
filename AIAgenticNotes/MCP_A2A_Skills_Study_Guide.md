# MCP, Agent Skills & A2A Protocol — Complete Study Guide
### Covering: Model Context Protocol · Agent Skills · Agent-to-Agent Protocol · Observability

---

## Table of Contents
1. [The Big Picture — Agentic Ecosystem Map](#1-big-picture)
2. [MCP — Model Context Protocol](#2-mcp)
3. [Agent Skills](#3-agent-skills)
4. [A2A — Agent-to-Agent Protocol](#4-a2a-protocol)
5. [Observability for Agents](#5-observability)
6. [The Grand Comparison: Skills vs Prompts vs A2A vs MCP](#6-grand-comparison)
7. [Code Reference — A2A Protocol Implementation](#7-code-reference)
8. [Demo Project Structure](#8-demo-project)
9. [Key Principles & Takeaways](#9-takeaways)

---

## 1. The Big Picture — Agentic Ecosystem Map

```
╔══════════════════════════════════════════════════════════════════╗
║              FULL AGENTIC ORGANIZATION STACK                     ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║   USER / END USER                                                ║
║       │                                                          ║
║       ▼                                                          ║
║   APPLICATION (Agent + LLM)                                      ║
║       │                                                          ║
║       ├──────────────────────────────────────────────────┐       ║
║       │  AGENT SKILLS                  MCP SERVERS        │       ║
║       │  (team playbook)               (nervous system)   │       ║
║       │  • Instructions                • Tools            │       ║
║       │  • Scripts                     • Resources        │       ║
║       │  • Assets                      • Prompts          │       ║
║       │  persisted across sessions     persistent connect │       ║
║       └───────────────────────────────────────────────────┘       ║
║       │                                                  │       ║
║       ▼ A2A Protocol                                              ║
║   SUB-AGENT 1 ◄──────────────► SUB-AGENT 2                      ║
║   (LangGraph)                   (AutoGen / CrewAI)               ║
║       │                              │                           ║
║       ▼                              ▼                           ║
║   MCP Server A                   MCP Server B                    ║
║   (Weather API)                  (Database)                      ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝

MCP  = How agents reach OUT to tools & data
Skills = How agents know WHAT to do and HOW
A2A  = How agents talk TO EACH OTHER
```

### Evolution of Agent Autonomy

```
L0 — Manual: Human does everything
L1 — Assisted: Human + AI suggestions
L2 — Collaborative: AI acts, human reviews
L3 — Delegated: AI acts autonomously, human sets goals
L4 — Agentic Org: Multiple agents coordinate with minimal human oversight
```

---

## 2. MCP — Model Context Protocol

### What Is MCP?

MCP is a **client-server protocol** that connects AI agents (clients) to external systems, APIs, and data sources (servers). Think of it as the **nervous system or hands** of your agent — it lets the agent *reach out* and interact with the world.

```
WITHOUT MCP:
  User ──► LLM ──► Text response only
            (can't access live data, files, APIs)

WITH MCP:
  User ──► LLM ──► MCP Client ──► MCP Server ──► Tools/APIs/DBs
                     ◄──── result ◄────────────────────────────
```

### Core Actors

```
┌──────────────────────────────────────────────────────────┐
│                     MCP ECOSYSTEM                         │
├──────────────────────────────────────────────────────────┤
│                                                          │
│   HOST (e.g., Claude Desktop, VS Code, your app)         │
│       │                                                  │
│       ▼                                                  │
│   CLIENT (lives inside the host)                         │
│   • Knows which MCP servers are available                │
│   • Embeds tool descriptions into LLM context            │
│   • Invokes tools based on LLM's decision                │
│       │                                                  │
│       │  (MCP protocol over stdio / HTTP+SSE)            │
│       ▼                                                  │
│   SERVER (exposes capabilities)                          │
│   • Tools    (functions the LLM can call)                │
│   • Resources (files, data the LLM can read)             │
│   • Prompts  (reusable prompt templates)                 │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### How Tool Invocation Actually Works

This is a critical concept: **the client never hard-codes "invoke tool X"**. The LLM decides.

```
Step 1: MCP Client discovers available tools from MCP Server
        Tools: [get_weather, get_forecast, get_alerts]

Step 2: Client embeds tool definitions into OpenAI API call
        {
          "model": "gpt-4o",
          "messages": [{"role": "user", "content": "What's the weather in Paris?"}],
          "tools": [
            {"name": "get_weather", "description": "Get current weather", ...},
            {"name": "get_forecast", "description": "Get 7-day forecast", ...}
          ]
        }

Step 3: OpenAI responds with tool_call decision
        {"tool_calls": [{"name": "get_weather", "arguments": {"city": "Paris"}}]}

Step 4: MCP Client sees the tool_call → invokes get_weather("Paris") on MCP Server

Step 5: MCP Server returns result → Client sends result back to OpenAI

Step 6: OpenAI generates final natural language response

Key insight: Nowhere in client code do you see "if weather question → call get_weather"
            The LLM figures that out automatically from tool descriptions.
```

### MCP Server — Weather Example

```python
# weather_server.py
from mcp import FastMCP
import httpx

mcp = FastMCP("Weather Server")

@mcp.tool()
async def get_weather(city: str) -> dict:
    """Get current weather for a city."""
    async with httpx.AsyncClient() as client:
        resp = await client.get(f"https://api.weather.com/{city}")
        return resp.json()

@mcp.tool()
async def get_forecast(city: str, days: int = 7) -> dict:
    """Get weather forecast for a city."""
    async with httpx.AsyncClient() as client:
        resp = await client.get(f"https://api.weather.com/{city}/forecast?days={days}")
        return resp.json()

# Run the server
mcp.run()
```

### MCP Client — Connecting to Server

```python
# weather_client.py
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from openai import OpenAI
import json

async def run_client():
    # 1. Connect to MCP Server
    server_params = StdioServerParameters(
        command="python", args=["weather_server.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # 2. Discover available tools from MCP Server
            tools_result = await session.list_tools()
            
            # 3. Convert MCP tools to OpenAI tool format
            openai_tools = [
                {
                    "type": "function",
                    "function": {
                        "name": tool.name,
                        "description": tool.description,
                        "parameters": tool.inputSchema,
                    }
                }
                for tool in tools_result.tools
            ]

            # 4. Send user query + available tools to OpenAI
            client = OpenAI()
            response = client.chat.completions.create(
                model="gpt-4o",
                messages=[{"role": "user", "content": "What's the weather in Paris?"}],
                tools=openai_tools,  # LLM sees what tools exist
            )

            # 5. If LLM chose a tool, invoke it via MCP
            message = response.choices[0].message
            if message.tool_calls:
                for tool_call in message.tool_calls:
                    tool_name = tool_call.function.name
                    tool_args = json.loads(tool_call.function.arguments)
                    
                    # MCP Client invokes the tool on MCP Server
                    result = await session.call_tool(tool_name, tool_args)
                    print(f"Tool: {tool_name}, Result: {result.content}")
```

### Agentic Loop (Real-World Orchestration)

```
Orchestrator receives user query
         │
         ▼
  LLM analyzes query
         │
    ┌────┴────┐
    │ Needs   │
    │ tools?  │
    └────┬────┘
    YES  │  NO ──► Return answer directly
         ▼
  Invoke Tool A via MCP
         │
  Got result — satisfied?
    ┌────┴────┐
    NO        YES ──► Send result back to LLM → Final answer
    │
    ▼
  Invoke Tool B via MCP (different path)
         │
  ... (can loop multiple times)
         │
  Compose final answer

REAL EXAMPLE from transcript:
  User asks GitHub Copilot to "run this project"
  → Copilot invokes: configure_python_env tool
  → Copilot invokes: create_virtual_env tool
  → Copilot invokes: launch_terminal tool
  → Copilot invokes: run_server tool
  All automatically — no human says "invoke this next"
```

### MCP Transport Options

```
stdio (Standard I/O)          HTTP + SSE (Server-Sent Events)
─────────────────────         ────────────────────────────────
Local processes only          Remote servers, web apps
Fastest                       Network-capable
Best for local tools          Best for cloud services
Simple setup                  Scalable, multi-client
```

### MCP Timeout & Guardrails

```python
# Prevent infinite loops and resource exhaustion
from mcp import ClientSession

session = ClientSession(
    read, write,
    # Timeout if tool takes too long
    timeout=30.0,
    # Max retries before giving up
    max_retries=3,
)

# Client-side: detect if agent is looping
MAX_TOOL_CALLS = 10
tool_call_count = 0

while True:
    response = get_llm_response_with_tools(...)
    if not response.tool_calls:
        break  # LLM is done
    
    tool_call_count += 1
    if tool_call_count > MAX_TOOL_CALLS:
        return "Error: Too many tool invocations. Possible loop detected."
    
    # invoke tool...
```

### Debugging MCP — MCP Inspector

```
MCP Inspector (open-source tool):
  • Intercepts client ↔ server communication
  • Shows every tool call with request/response
  • Identifies which tools are being invoked when
  • Shows timing and errors

Usage:
  npx @modelcontextprotocol/inspector python weather_server.py

BUT: More important than debugging tools is OBSERVABILITY:
  → Log every tool invocation (caller ID, timestamp, inputs, outputs)
  → Trace the full session (which tools were called, in what order)
  → Monitor TSR (Task Success Rate): how many tool calls succeeded vs failed
```

---

## 3. Agent Skills

### What Are Agent Skills?

Agent Skills are **directories of organized files** (instructions, scripts, assets) that give your agent **domain expertise and procedural knowledge**. Think of them as the **team playbook or user manual** — you define once, agent uses everywhere.

```
WITHOUT SKILLS:
  Every conversation: "Hey agent, remember our brand guidelines are X,
                       format responses like Y, and when you see Z do W..."
  (repetitive, error-prone, hard to maintain)

WITH SKILLS:
  skill.md defines everything ONCE.
  Agent discovers and loads it automatically when needed.
  Consistent behavior across all conversations.
```

### Skills Directory Structure

```
my-agent/
├── .claude/
│   └── skills/
│       ├── marketing-campaign/
│       │   ├── skill.md          ← ENTRY POINT (always required)
│       │   ├── analyze.py        ← Script agent can execute
│       │   ├── templates/
│       │   │   └── report.html   ← Asset agent can use
│       │   └── resources/
│       │       └── optimization_framework.md
│       │
│       ├── brand-guidelines/
│       │   ├── skill.md
│       │   ├── colors.json
│       │   └── logo_usage.md
│       │
│       ├── bigquery-analysis/
│       │   ├── skill.md
│       │   └── query_templates.sql
│       │
│       └── powerpoint-creator/
│           ├── skill.md
│           └── create_deck.py
```

### skill.md Format

```markdown
# Skill: Analyze Marketing Campaign

## Description
Analyze weekly marketing campaign performance data and provide
actionable recommendations with budget optimization.

## Inputs
- campaign_data: CSV or JSON with impressions, clicks, conversions
- date_range: Start and end date for analysis period
- budget: Current campaign budget

## Outputs
- performance_report: Structured analysis with metrics
- recommendations: List of actionable next steps
- optimized_budget: Suggested budget reallocation

## Funnel Metrics
- Click-through rate (CTR) target: > 2%
- Conversion rate target: > 1.5%
- Cost per acquisition (CPA) target: < $50

## Instructions
1. Load campaign data from provided source
2. Calculate CTR, conversion rate, and CPA for each channel
3. Compare against target metrics above
4. Use optimization_framework.md for budget decisions
5. If CTR < 1%, flag for creative refresh
6. Generate report using templates/report.html

## When to Use
Trigger this skill when user mentions:
- "marketing campaign review"
- "weekly performance"
- "campaign analysis"
- "ad spend optimization"

## Resources
- [Optimization Framework](resources/optimization_framework.md)
- [Report Template](templates/report.html)
```

### Progressive Disclosure — Why It Matters

```
Problem: You have 100 skills. If ALL are loaded at once:
  100 skills × 5,000 tokens each = 500,000 tokens consumed
  → Context window overloaded
  → Extremely expensive
  → Agent confused by too much context

Solution: PROGRESSIVE DISCLOSURE

Session Start:
  Load ONLY metadata for all skills (name + description)
  ≈ 100 skills × 50 tokens = 5,000 tokens ✓

User asks: "Can you analyze our marketing campaign?"
  Agent matches description → loads marketing-campaign skill FULLY
  ≈ 5,000 tokens for that skill only ✓

Work complete:
  Marketing-campaign skill cleared from context

User asks: "Create a PowerPoint for the results"
  Agent loads powerpoint-creator skill
  ≈ 3,000 tokens ✓

┌─────────────────────────────────────────────────────────┐
│  ALWAYS LOADED:  Metadata (name + description) for all  │
│  ON DEMAND:      Full skill content when triggered       │
│  CLEARED AFTER:  Skill unloaded when task complete       │
└─────────────────────────────────────────────────────────┘
```

### Skills Use Cases

| Skill Type | What It Contains | Example |
|---|---|---|
| **Domain Expertise** | Deep knowledge in one area | BigQuery analysis, legal review |
| **Brand Governance** | Brand guidelines, templates | Logo usage, colors, tone of voice |
| **Process Automation** | Step-by-step workflows | Bug triage, customer call prep |
| **Code Execution** | Python scripts for agent to run | Report generator, data processor |
| **Error Recovery** | "If you see error X, run this script" | Auto-remediation scripts |
| **Marketing** | Campaign review, weekly reports | Ad spend optimization |
| **Compliance** | Governance rules, data masking | "Never send customer PII to LLM" |

### Skills vs Traditional Prompt Engineering

```
Traditional Approach:
  Dev: "Hey Claude, you're a marketing analyst.
        Our brand colors are #FF5733 and #33FF57.
        Format all reports with these sections: Executive Summary,
        Metrics, Recommendations. Never mention competitors.
        Use formal tone. Reference our Q3 goals which are..."
  (Every. Single. Conversation.)

Skill Approach:
  brand-guidelines/skill.md: [defined ONCE]
  marketing-analyst/skill.md: [defined ONCE]
  governance/skill.md: [defined ONCE]

  Dev: "Analyze last week's campaign"
  → Agent loads relevant skills automatically
  → Consistent, governed, context-aware behavior
```

### How General-Purpose Agent + Skills Works

```
OLD PATTERN (specialized agents):
  Research Agent ──► web search only
  Finance Agent  ──► financial data only
  Marketing Agent──► campaign data only
  Coding Agent   ──► code generation only

NEW PATTERN (general agent + skills):
  General-Purpose Agent
       │
       ├── marketing-campaign skill ──► becomes marketing expert
       ├── legal-review skill       ──► becomes legal reviewer
       ├── bigquery skill           ──► becomes data analyst
       └── powerpoint skill         ──► becomes presentation maker

Note: This doesn't mean you ALWAYS use one agent.
      Multi-agent patterns still valid for complex systems.
```

### Skills in Multi-Agent Scenarios

```
Agent (Main)
    │
    ├── Shared Skills ────────────────────┐
    │   (brand-guidelines, governance)    │
    │                                     │
    ├──── Sub-Agent 1                     │
    │         │                           │
    │         └── Uses shared skills ─────┤
    │             + isolated context      │
    │             + own tool permissions  │
    │                                     │
    └──── Sub-Agent 2                     │
              │                           │
              └── Uses shared skills ─────┘
                  + different context
```

---

## 4. A2A — Agent-to-Agent Protocol

### What Is A2A?

A2A (Agent-to-Agent) is a **standard protocol for connecting standalone agents so they can collaborate**. Like HTTP unlocked the web for connecting pages, A2A unlocks the agentic ecosystem for connecting agents.

```
HTTP (1989):  "How do web pages talk to each other?"
A2A  (2024+): "How do AI agents talk to each other?"

Without A2A:
  Agent A (LangChain) ─ custom code ─► Agent B (AutoGen)  ← brittle, platform-locked

With A2A:
  Agent A (LangChain) ──A2A Protocol──► Agent B (AutoGen) ← standard, interoperable
  Agent A (CrewAI)    ──A2A Protocol──► Agent B (Semantic Kernel)
  Agent (EU company)  ──A2A Protocol──► Agent (US company)
```

### The Three Core Actors

```
┌─────────────────────────────────────────────────────────────────┐
│                      A2A ECOSYSTEM                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   USER (End User)                                               │
│   • Initiates the request / defines goals (OKRs)               │
│   • Sets the problem statement                                  │
│       │                                                         │
│       ▼                                                         │
│   A2A CLIENT (Application / Service / Another Agent)           │
│   • Acts on behalf of the user                                  │
│   • Requests actions from remote agents                         │
│   • Communicates via A2A Protocol                               │
│       │                                                         │
│       │  ──── HTTPS + JSON-RPC 2.0 ────►                       │
│       │                                                         │
│       ▼                                                         │
│   A2A SERVER (Remote Agent)                                     │
│   • Any AI agent or agentic system                              │
│   • Exposes an HTTP endpoint implementing A2A spec              │
│   • Does NOT need to be a separate "server" — the agent IS     │
│     the server                                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Compare with MCP:   HOST → CLIENT → SERVER
Compare with A2A:   USER → A2A CLIENT → A2A SERVER (which is another agent)
```

### Three Key Facilities

```
┌──────────────────────────────────────────────────────────────────┐
│ FACILITY 1: INTEROPERABILITY                                      │
│                                                                  │
│  Agent A (LangChain) ──A2A──► Agent B (AutoGen)                  │
│  Agent C (CrewAI)    ──A2A──► Agent D (Semantic Kernel)          │
│  EU Organization     ──A2A──► US Organization                    │
│                                                                  │
│  Agents built on DIFFERENT platforms can talk to each other.     │
│  No platform lock-in.                                            │
├──────────────────────────────────────────────────────────────────┤
│ FACILITY 2: COMPLEX WORKFLOWS                                     │
│                                                                  │
│  Travel Agent                                                    │
│      │                                                           │
│      ├──A2A──► Flight Agent   (handles flights)                  │
│      ├──A2A──► Hotel Agent    (handles accommodation)            │
│      ├──A2A──► Activity Agent (handles activities)               │
│      └──A2A──► Guide Agent    (local expertise)                  │
│                                                                  │
│  • Delegate subtasks to specialized agents                       │
│  • Exchange information between agents                           │
│  • Coordinate actions across agents                              │
│  • Handle conflicts between agents (with defined resolution rules)│
├──────────────────────────────────────────────────────────────────┤
│ FACILITY 3: SECURE & OPAQUE INTERACTION                          │
│                                                                  │
│  Agent B invokes Agent A                                         │
│  Agent B does NOT know:                                          │
│    • Agent A's internal memory                                   │
│    • Agent A's proprietary logic                                 │
│    • What MCP tools Agent A uses internally                      │
│                                                                  │
│  Like OOP abstraction: call the method, don't peek at internals  │
│  → Each agent's implementation is a black box to others         │
└──────────────────────────────────────────────────────────────────┘
```

### A2A Communication Elements

```
5 FUNDAMENTAL ELEMENTS:

┌──────────────────────────────────────────────────────────────────┐
│ 1. AGENT CARD                                                    │
│    JSON metadata document at a well-known URL                    │
│    Describes what the agent can do (like a business card)        │
│    Always discoverable, helps ecosystem know all agents running  │
│                                                                  │
│    {                                                             │
│      "name": "WeatherAgent",                                     │
│      "description": "Provides real-time weather information",    │
│      "url": "https://agents.mycompany.com/weather",              │
│      "version": "1.0.0",                                         │
│      "capabilities": ["get_weather", "get_forecast"],            │
│      "skills": ["weather-analysis", "climate-reporting"]         │
│    }                                                             │
├──────────────────────────────────────────────────────────────────┤
│ 2. TASK                                                          │
│    A unit of work the agent must complete                        │
│    Created when a message requires stateful processing           │
│    Tracks state: submitted → in_progress → completed/failed      │
│                                                                  │
│    {                                                             │
│      "id": "task-abc123",                                        │
│      "status": "in_progress",                                    │
│      "goal": "Get weather forecast for Paris for 7 days",        │
│      "created_at": "2025-05-27T10:00:00Z"                        │
│    }                                                             │
├──────────────────────────────────────────────────────────────────┤
│ 3. ARTIFACT                                                      │
│    Tangible output/result produced by the agent for a task       │
│    Could be text, JSON, file, image                              │
│                                                                  │
│    {                                                             │
│      "task_id": "task-abc123",                                   │
│      "type": "json",                                             │
│      "content": {"forecast": [...7 days of data...]}             │
│    }                                                             │
├──────────────────────────────────────────────────────────────────┤
│ 4. MESSAGE                                                       │
│    Single turn of communication between client and agent         │
│    Contains one or more PARTS                                    │
│                                                                  │
│    {                                                             │
│      "role": "user",                                             │
│      "parts": [{"type": "text", "content": "Weather in Paris?"}] │
│    }                                                             │
├──────────────────────────────────────────────────────────────────┤
│ 5. PART                                                          │
│    Fundamental unit of content within a message or artifact      │
│    Can be: text, data (JSON), file, image, audio, video          │
│                                                                  │
│    Text Part: {"type": "text", "text": "Hello"}                  │
│    Data Part: {"type": "data", "data": {"key": "value"}}         │
│    File Part: {"type": "file", "file": {"uri": "file://..."}}    │
└──────────────────────────────────────────────────────────────────┘
```

### A2A Protocol — Design Principles

```
┌──────────────────────────────────────────────────────────────────┐
│ 1. NATURAL UNSTRUCTURED MODALITIES                               │
│    Agents communicate in natural language — no rigid formats     │
│    LLM interprets and converts to structured actions             │
├──────────────────────────────────────────────────────────────────┤
│ 2. BUILT ON EXISTING STANDARDS                                   │
│    HTTP (transport)                                              │
│    JSON-RPC 2.0 (message format)                                 │
│    OAuth 2.0 / OpenID Connect / API Keys (auth)                  │
│    TLS/HTTPS (encryption)                                        │
│    No new security primitives — reuses battle-tested web infra   │
├──────────────────────────────────────────────────────────────────┤
│ 3. LONG-RUNNING TASKS                                            │
│    Supports tasks from milliseconds to hours/days               │
│    Quick: "What's 2+2?" → immediate response                     │
│    Long: "Research this topic deeply" → background processing    │
│    Streaming updates during long tasks (SSE)                     │
├──────────────────────────────────────────────────────────────────┤
│ 4. MULTI-MODAL                                                   │
│    Communication not limited to text                             │
│    Supports: text, images, audio, video, files, JSON data        │
└──────────────────────────────────────────────────────────────────┘
```

### A2A Enterprise Security

```
Layer 1: Transport Security
  HTTPS / TLS — all communication encrypted in transit

Layer 2: Authentication
  OAuth 2.0   — industry standard token-based auth
  OpenID Connect — identity federation
  API Keys    — simple service-to-service auth
  Choice of method left to implementation partner

Layer 3: Authorization
  Each A2A Server is responsible for authorizing requests
  "Can Agent B call Agent A?" → Agent A's server decides
  Principle of Least Privilege: agents only access what they need

Layer 4: Agent Identity
  Each agent has its own identity (like a service account)
  Enables fine-grained access control between agents

Layer 5: Data Privacy
  Cross-geo compliance (GDPR if agents cross EU boundary)
  Implementers must understand data sensitivity in messages
  Artifacts may contain regulated data → must be handled carefully

Layer 6: Observability (Implementation Responsibility)
  Distributed tracing across agent calls
  Comprehensive logging and operational metrics
  API management: throttling, rate limiting, governance
```

### A2A + MCP Working Together

```
                    AGENTIC APPLICATION
                           │
                    MAIN AGENT (LLM)
                    │            │
                    │            │
              SKILLS          MCP NETWORK
            (playbook)    (tools & data)
                    │            │
                    │            │
              SUB-AGENT 1   SUB-AGENT 2
              (LangChain)   (AutoGen)
                    │
                    │── A2A Protocol ──►  BLACK BOX AGENT 1
                    │                    (external, unknown internals)
                    │                    [has its own MCP server]
                    │
                    └── A2A Protocol ──►  BLACK BOX AGENT 2
                                         (external, unknown internals)
                                         [has its own skills]

KEY: Sub-agents communicate with external agents via A2A,
     but those external agents' internals remain opaque.
     Sub-agents don't need to know what tools Black Box Agent 1 uses.
```

### Conflict Resolution Between Agents

```
Agent A says: "I should process this request via Path X"
Agent B says: "I should process this request via Path Y"
         ─── CONFLICT ───
A2A Protocol provides conflict resolution rules:

Pattern Options:
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   HIERARCHICAL  │  │  CENTRALIZED    │  │   FEDERATED     │
│   CHAIN         │  │  ARBITRATOR     │  │   CONSENSUS     │
│                 │  │                 │  │                 │
│  Orchestrator   │  │  Central Agent  │  │  Agents vote    │
│  has final say  │  │  decides        │  │  Majority wins  │
└─────────────────┘  └─────────────────┘  └─────────────────┘
Implementation partner defines which pattern to use.
```

---

## 5. Observability

### The Ground Rule

> **"Never create an application where your agent has access to do things that a human cannot do."**

If an agent can reach system state 11 but humans can only reach state 10, you have no recovery path when the agent corrupts state 11.

```
DANGEROUS:
  Human can access: Forms 1-10
  Agent can access: Forms 1-11  ← Form 11 hidden from human review
  Agent corrupts Form 11 → Human CANNOT fix it → System stuck

SAFE:
  Human can access: Forms 1-10
  Agent can access: Forms 1-10 ONLY
  Agent corrupts anything → Human CAN review and fix it
```

### What Observability Covers for Agents

```
TRADITIONAL SOFTWARE OBSERVABILITY:
  • Request/response logging
  • Error rates
  • Latency metrics

AGENT OBSERVABILITY (MORE COMPLEX):
  • Every TOOL INVOCATION (name, args, result, timestamp, caller)
  • Every DECISION STEP (why did LLM choose this tool? this path?)
  • WORKFLOW EXECUTION (which agents were called, in what order?)
  • LLM INPUTS/OUTPUTS (full context sent to model, full response)
  • TOKEN USAGE (per agent, per session, cumulative)
  • TASK SUCCESS RATE (TSR) per tool per agent
  • LOOP DETECTION (agent calling same tool repeatedly without progress)
  • HUMAN-IN-THE-LOOP TRIGGERS (when should human be alerted?)
```

### Observability Stack

```
┌──────────────────────────────────────────────────────────────┐
│ LAYER 1: DATA INSTRUMENTATION                                │
│   • Log every tool call, decision, state change             │
│   • Structured format: JSON with trace_id, session_id       │
│   • Capture: timestamp, agent_id, tool, inputs, outputs     │
├──────────────────────────────────────────────────────────────┤
│ LAYER 2: TELEMETRY PIPELINE                                  │
│   • OpenTelemetry traces (distributed, across agents)        │
│   • Metrics pipeline (Prometheus / CloudWatch / Datadog)     │
│   • Log aggregation (ELK stack / Splunk / CloudWatch Logs)  │
├──────────────────────────────────────────────────────────────┤
│ LAYER 3: MONITORING & DASHBOARDS                             │
│   • Real-time TSR per tool                                   │
│   • Latency P50, P95, P99 per agent                         │
│   • Cost per session / per query type                        │
│   • Error rates by agent and tool                            │
├──────────────────────────────────────────────────────────────┤
│ LAYER 4: PROACTIVE ALERTS                                    │
│   • Agent in loop → alert human (after N iterations)        │
│   • Tool failure rate > threshold → alert                    │
│   • Cost spike → alert + optional circuit breaker           │
│   • Suspicious activity (unusual tool combinations) → alert  │
├──────────────────────────────────────────────────────────────┤
│ LAYER 5: HUMAN-IN-THE-LOOP                                   │
│   • Alerts route to human reviewer                           │
│   • Human can pause, correct, resume agent                  │
│   • Ideal: agent self-heals; fallback: human recovers        │
└──────────────────────────────────────────────────────────────┘
```

### Telemetry Implementation (MCP Server Side)

```python
import logging
import json
from datetime import datetime
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider

# Set up tracer
provider = TracerProvider()
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("mcp-weather-server")

# Wrap every MCP tool with observability
@mcp.tool()
async def get_weather(city: str) -> dict:
    """Get current weather for a city."""
    
    with tracer.start_as_current_span("tool.get_weather") as span:
        span.set_attribute("tool.name", "get_weather")
        span.set_attribute("tool.input.city", city)
        span.set_attribute("session.id", get_current_session_id())
        
        start_time = datetime.utcnow()
        
        try:
            result = await fetch_weather_api(city)
            
            span.set_attribute("tool.status", "success")
            span.set_attribute("tool.latency_ms", 
                (datetime.utcnow() - start_time).total_seconds() * 1000)
            
            # Log structured telemetry
            log_event({
                "event": "tool_invoked",
                "tool": "get_weather",
                "caller_id": get_caller_id(),
                "input": {"city": city},
                "output_size": len(str(result)),
                "status": "success",
                "timestamp": start_time.isoformat()
            })
            
            return result
            
        except Exception as e:
            span.set_attribute("tool.status", "error")
            span.record_exception(e)
            log_event({"event": "tool_failed", "tool": "get_weather", "error": str(e)})
            raise
```

### Debugging Tools

| Tool | Purpose | How |
|---|---|---|
| **MCP Inspector** | Intercept client↔server communication | `npx @modelcontextprotocol/inspector python server.py` |
| **LangSmith** | Trace full LLM + tool call chain | Auto-traces via env var |
| **OpenTelemetry** | Distributed traces across agents | Instrument each component |
| **Custom logging** | Tool-level telemetry | Log every call with structured JSON |

---

## 6. The Grand Comparison: Skills vs Prompts vs A2A vs MCP

```
╔══════════╦═══════════════════╦═══════════════════╦═════════════════╦════════════════╗
║          ║    SKILLS         ║    PROMPTS         ║    A2A          ║    MCP         ║
╠══════════╬═══════════════════╬═══════════════════╬═════════════════╬════════════════╣
║ ANALOGY  ║ Team playbook     ║ Today's briefing   ║ Inter-dept      ║ Hands /        ║
║          ║ / User manual     ║ / Meeting notes    ║ collaboration   ║ nervous system ║
╠══════════╬═══════════════════╬═══════════════════╬═════════════════╬════════════════╣
║ PURPOSE  ║ Give agent        ║ Moment-to-moment   ║ Task delegation ║ Tool           ║
║          ║ procedural        ║ instructions per   ║ between agents  ║ connectivity   ║
║          ║ knowledge         ║ conversation turn  ║                 ║ to ext systems ║
╠══════════╬═══════════════════╬═══════════════════╬═════════════════╬════════════════╣
║ SCOPE    ║ Superset of       ║ Single turn or     ║ Full agentic    ║ Tools,         ║
║          ║ prompts           ║ conversation       ║ logic           ║ resources,     ║
║          ║                   ║                    ║                 ║ prompts        ║
╠══════════╬═══════════════════╬═══════════════════╬═════════════════╬════════════════╣
║ CONTAINS ║ Instructions,     ║ Natural language   ║ Agent logic,    ║ API calls,     ║
║          ║ code scripts,     ║ only               ║ routing rules,  ║ DB queries,    ║
║          ║ assets, resources ║                    ║ coordination    ║ file I/O       ║
╠══════════╬═══════════════════╬═══════════════════╬═════════════════╬════════════════╣
║ LOADING  ║ Progressive:      ║ Loaded per turn    ║ Always present  ║ Always         ║
║          ║ metadata first,   ║ by human/app       ║ (protocol-      ║ available      ║
║          ║ full content on   ║                    ║ level)          ║ once server    ║
║          ║ demand            ║                    ║                 ║ connected      ║
╠══════════╬═══════════════════╬═══════════════════╬═════════════════╬════════════════╣
║ PERSIST  ║ Across sessions   ║ Single conversation║ Across sessions ║ Across         ║
║          ║ and conversations ║                    ║ (protocol-      ║ sessions       ║
║          ║                   ║                    ║ level)          ║                ║
╠══════════╬═══════════════════╬═══════════════════╬═════════════════╬════════════════╣
║ DEFINED  ║ Developer / team  ║ Developer / user   ║ Protocol spec   ║ Server         ║
║ BY       ║ once              ║ per conversation   ║ + implementer   ║ developer      ║
╠══════════╬═══════════════════╬═══════════════════╬═════════════════╬════════════════╣
║ WHEN TO  ║ Reusable domain   ║ One-off            ║ Multiple        ║ Need live      ║
║ USE      ║ expertise,        ║ instructions,      ║ agents must     ║ data, APIs,    ║
║          ║ governance,       ║ dynamic context    ║ collaborate     ║ file access    ║
║          ║ brand guidelines  ║                    ║                 ║                ║
╚══════════╩═══════════════════╩═══════════════════╩═════════════════╩════════════════╝
```

### The Analogy That Makes It Click

```
Think of building a COMPANY:

  SKILLS = Employee handbooks, SOPs, domain expertise
            "Here's how we handle marketing campaigns"
            "Here's our brand guidelines"
            Written once, referenced always

  PROMPTS = Daily stand-up meeting notes
            "Today, focus on the Q3 report"
            "This specific call, use formal tone"
            Ephemeral, per-session

  A2A = Inter-department collaboration protocol
        "How does Marketing talk to Finance?"
        "How does EU team collaborate with US team?"
        Standard rules for cross-team coordination

  MCP = Company's IT infrastructure
        Databases, APIs, file systems, cloud services
        The "hands" that let employees access data
```

### All Four Together in One System

```python
# A production multi-agent system uses ALL FOUR:

# 1. Skills: Agent knows HOW to analyze campaigns
#    (loaded from /skills/marketing-campaign/skill.md)

# 2. Prompts: User provides the specific request
user_prompt = "Analyze last week's EU campaign performance"

# 3. MCP: Agent fetches actual data via tools
# → Calls get_campaign_data("EU", "last_week") via MCP server

# 4. A2A: Main agent delegates to specialized agent
# → Main agent sends task to FinanceAgent via A2A
# → FinanceAgent calculates ROI independently
# → Returns artifact: {"roi": "127%", "recommendation": "Increase budget"}

# Result: Complete, accurate, governed response
```

---

## 7. Code Reference — A2A Protocol Implementation

### Agent Card (JSON)

```json
{
  "name": "WeatherAgent",
  "description": "Provides real-time weather data and forecasts",
  "url": "https://agents.mycompany.com/weather",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "push_notifications": false,
    "state_transition_history": true
  },
  "skills": [
    {
      "id": "weather-current",
      "name": "Current Weather",
      "description": "Get current weather conditions for any city",
      "tags": ["weather", "real-time"],
      "examples": ["What's the weather in Paris?", "Is it raining in London?"]
    },
    {
      "id": "weather-forecast",
      "name": "Weather Forecast",
      "description": "Get 7-day weather forecasts",
      "tags": ["weather", "forecast"],
      "examples": ["Forecast for Tokyo this week?"]
    }
  ],
  "auth": {
    "schemes": ["Bearer"]
  }
}
```

### A2A Protocol Core Classes

```python
# a2a_protocol.py

from dataclasses import dataclass
from typing import Optional, List, Union
from enum import Enum
import json

class TaskStatus(Enum):
    SUBMITTED = "submitted"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"

@dataclass
class TextPart:
    """Fundamental unit: text content"""
    type: str = "text"
    text: str = ""

@dataclass
class DataPart:
    """Fundamental unit: structured data"""
    type: str = "data"
    data: dict = None

@dataclass
class Message:
    """Single turn of communication"""
    role: str  # "user" or "agent"
    parts: List[Union[TextPart, DataPart]]
    
    def to_dict(self) -> dict:
        return {
            "role": self.role,
            "parts": [{"type": p.type, "text": p.text} 
                      if isinstance(p, TextPart) 
                      else {"type": p.type, "data": p.data}
                      for p in self.parts]
        }

@dataclass
class Task:
    """Unit of work for an agent"""
    id: str
    status: TaskStatus
    messages: List[Message]

@dataclass
class Artifact:
    """Tangible output produced by agent"""
    task_id: str
    type: str  # "text", "json", "file"
    content: Union[str, dict]


# Helper functions
def make_user_message(text: str) -> Message:
    return Message(role="user", parts=[TextPart(text=text)])

def make_agent_message(text: str = None, data: dict = None) -> Message:
    parts = []
    if text:
        parts.append(TextPart(text=text))
    if data:
        parts.append(DataPart(data=data))
    return Message(role="agent", parts=parts)

def extract_text(message: Message) -> str:
    texts = [p.text for p in message.parts if isinstance(p, TextPart)]
    return " ".join(texts)

def extract_data(message: Message) -> Optional[dict]:
    for part in message.parts:
        if isinstance(part, DataPart):
            return part.data
    return None
```

### A2A Server — Weather Agent

```python
# weather_a2a_server.py
from fastapi import FastAPI, HTTPException
from a2a_protocol import Task, TaskStatus, make_agent_message, Artifact
import uuid

app = FastAPI()

# Agent Card endpoint (discoverable at well-known URL)
@app.get("/.well-known/agent.json")
async def agent_card():
    return {
        "name": "WeatherAgent",
        "description": "Real-time weather data and forecasts",
        "url": "http://localhost:8001",
        "version": "1.0.0",
        "capabilities": {"streaming": False},
        "skills": [{"id": "weather", "name": "Weather", "description": "Weather data"}]
    }

# Task submission endpoint
@app.post("/tasks/send")
async def handle_task(request: dict):
    task_id = str(uuid.uuid4())
    user_text = request["messages"][0]["parts"][0]["text"]
    
    # Agent processes the task
    city = extract_city_from_query(user_text)  # NLP or simple parsing
    weather_data = await fetch_weather(city)
    
    # Return artifact
    return {
        "task": {
            "id": task_id,
            "status": "completed"
        },
        "artifacts": [{
            "task_id": task_id,
            "type": "json",
            "content": weather_data
        }]
    }
```

### A2A Client — Coordinator Agent

```python
# coordinator_a2a_client.py
import httpx
from a2a_protocol import make_user_message, extract_text

class A2AClient:
    """Client for communicating with remote agents via A2A protocol."""
    
    def __init__(self, base_url: str, api_key: str = None):
        self.base_url = base_url
        self.headers = {"Authorization": f"Bearer {api_key}"} if api_key else {}
    
    async def discover_capabilities(self) -> dict:
        """Fetch Agent Card from remote agent."""
        async with httpx.AsyncClient() as client:
            response = await client.get(
                f"{self.base_url}/.well-known/agent.json",
                headers=self.headers
            )
            return response.json()
    
    async def send_task(self, user_message: str) -> dict:
        """Send a task to the remote agent."""
        message = make_user_message(user_message)
        
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.base_url}/tasks/send",
                json={"messages": [message.to_dict()]},
                headers=self.headers,
                timeout=30.0  # Timeout guard
            )
            return response.json()


# Coordinator orchestrating multiple agents
class CoordinatorAgent:
    def __init__(self):
        self.weather_agent = A2AClient("http://localhost:8001", api_key="...")
        self.venue_agent = A2AClient("http://localhost:8002", api_key="...")
    
    async def plan_event(self, user_request: str) -> str:
        # Delegate to weather agent via A2A
        weather_result = await self.weather_agent.send_task(
            f"What's the weather for outdoor event this weekend? {user_request}"
        )
        
        # Delegate to venue agent via A2A
        venue_result = await self.venue_agent.send_task(
            f"Find available venues for: {user_request}"
        )
        
        # Coordinator synthesizes results
        return f"Weather: {weather_result}\nVenues: {venue_result}"
```

---

## 8. Demo Project Structure

From the Drive (Demos folder → MCP-Demo, Skills-Demo, A2A-Demo):

```
Demos/
├── MCP-Demo/
│   ├── README.md (MCP_steps.md)  ← Server+client setup guide
│   ├── weather_server.py          ← MCP Server exposing weather tools
│   ├── weather_client.py          ← MCP Client using OpenAI + tools
│   └── .env.example               ← OPENAI_API_KEY, etc.
│
├── Skills-Demo/
│   ├── .claude/
│   │   └── skills/
│   │       ├── marketing-campaign/
│   │       │   ├── skill.md
│   │       │   └── analyze.py
│   │       └── brand-guidelines/
│   │           └── skill.md
│   └── agent.py
│
└── A2A-Demo/
    ├── a2a_protocol.py            ← Agent Card, Task, Message, Part, Artifact
    ├── weather_agent.py           ← A2A Server (weather)
    ├── venue_agent.py             ← A2A Server with skills (no MCP)
    ├── coordinator_agent.py       ← A2A Client orchestrating both
    └── README.md
```

### Running the MCP Demo

```bash
# 1. Install Python package manager (uv)
pip install uv

# 2. Create virtual environment
uv venv && source .venv/bin/activate

# 3. Install dependencies
uv pip install -r requirements.txt

# 4. Set environment variables
cp .env.example .env
# Edit .env: add OPENAI_API_KEY=sk-...

# 5. Launch MCP Server (Terminal 1)
python weather_server.py

# 6. Launch MCP Client (Terminal 2)
python weather_client.py

# OR: Use MCP Inspector to debug
npx @modelcontextprotocol/inspector python weather_server.py
```

---

## 9. Key Principles & Takeaways

### MCP Takeaways

| # | Principle |
|---|---|
| 1 | MCP is a client-server protocol — server exposes tools, client embeds them into LLM calls |
| 2 | The LLM (not the client code) decides which tool to invoke — client code never says "call tool X" |
| 3 | Add timeout + loop guards on the client side to prevent infinite tool-calling |
| 4 | MCP Inspector for debugging; OpenTelemetry for production observability |
| 5 | Use stdio transport for local tools, HTTP+SSE for remote/cloud tools |

### Agent Skills Takeaways

| # | Principle |
|---|---|
| 6 | Skills = directory of organized files (skill.md + scripts + assets) |
| 7 | Start with skill.md — defines name, description, instructions, inputs, outputs |
| 8 | Progressive disclosure: metadata always loaded, full content only on demand |
| 9 | Skills persist across conversations — unlike prompts (single session) |
| 10 | Skills are a superset of prompts — think of them as persistent, reusable prompt+code |
| 11 | Skills are for team productivity and modular reuse, not just one-off customization |

### A2A Protocol Takeaways

| # | Principle |
|---|---|
| 12 | A2A = HTTP for agents — lets standalone agents from any framework collaborate |
| 13 | Three actors: User → A2A Client → A2A Server (which is another agent) |
| 14 | Five communication elements: Agent Card, Task, Artifact, Message, Part |
| 15 | Secure by default: HTTPS, OAuth 2.0, OpenID Connect — no new security primitives |
| 16 | Agent Card at /.well-known/agent.json — how the ecosystem discovers each agent |
| 17 | Secure + opaque: Agent B invokes Agent A without knowing A's internals |

### Observability Takeaways

| # | Principle |
|---|---|
| 18 | Ground rule: agents must never have access to anything a human cannot review/fix |
| 19 | Observe every decision step, not just inputs/outputs |
| 20 | Log every tool invocation: who called it, when, with what args, with what result |
| 21 | TSR (Task Success Rate) = key metric for tool health |
| 22 | Self-healing first, human-in-the-loop as fallback, never let system reach unrecoverable state |
| 23 | Observability is not optional — it's the foundation that makes everything else trustworthy |

### The One-Slide Summary

```
┌──────────────────────────────────────────────────────────────┐
│                   THE AGENTIC STACK                          │
├───────────────┬──────────────────────────────────────────────┤
│   WHAT        │   METAPHOR         │   USE WHEN              │
├───────────────┼────────────────────┼─────────────────────────┤
│   MCP         │ Hands / nervous    │ Access live data, APIs, │
│               │ system             │ files, cloud services   │
├───────────────┼────────────────────┼─────────────────────────┤
│   Skills      │ Team playbook /    │ Domain expertise,       │
│               │ employee handbook  │ governance, reusability │
├───────────────┼────────────────────┼─────────────────────────┤
│   A2A         │ Org chart /        │ Multiple agents must    │
│               │ collaboration proto│ delegate & coordinate   │
├───────────────┼────────────────────┼─────────────────────────┤
│   Prompts     │ Today's meeting    │ One-off, dynamic,       │
│               │ notes              │ per-conversation context│
├───────────────┼────────────────────┼─────────────────────────┤
│ Observability │ Flight recorder    │ ALWAYS — non-negotiable │
│               │ + control tower    │ for production systems  │
└───────────────┴────────────────────┴─────────────────────────┘

Production systems use ALL of these together.
There is no "use this OR that" — understand each and combine them.
```
