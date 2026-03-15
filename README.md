# Multi-Protocol AI Agent Notebooks

A collection of Google Colab notebooks demonstrating three different protocols for building AI agent systems — all using the **same travel research use case** for easy comparison. Each notebook builds a multi-agent travel advisory that combines weather forecasts, country profiles, and public holiday data from free public APIs.

Built for the **UT Austin McCombs / Great Learning** _Generative AI for Business Applications_ course.

---

## Notebooks

### 1. `MCP World Explorer.ipynb` — Model Context Protocol

**Protocol:** [MCP (Model Context Protocol)](https://modelcontextprotocol.io/) by Anthropic

**What it teaches:**
MCP is a **vertical integration** protocol — it connects a single LLM to external tools and data sources. The LLM decides which tools to call, sends structured parameters, and receives structured results. Think of it as giving an LLM hands to reach into databases, APIs, and file systems.

**Architecture:**
```
┌──────────────┐         MCP          ┌──────────────┐
│     LLM      │ ───────────────────► │    Tools     │ ──► Real APIs
│  (one agent) │   stdio / SSE        │  (functions) │
└──────────────┘                      └──────────────┘
```

**How it works:**
- Defines **tool functions** (e.g., `get_weather()`, `get_country_info()`, `get_holidays()`) that wrap free public APIs
- The LLM reads tool schemas, decides which to call, provides parameters, and receives structured JSON back
- A single LLM orchestrates everything — there are no autonomous agents, just an LLM with tool access
- Uses MCP's tool-calling interface over stdio or SSE transport

**Key concepts covered:**
- MCP tool schemas and descriptions
- LLM tool selection and parameter generation
- Structured tool responses
- Single-agent orchestration with multiple tools

**APIs used:**
| Tool | API | Purpose |
|------|-----|---------|
| Weather | [Open-Meteo](https://open-meteo.com/) | Current conditions + 7-day forecast |
| Country | [REST Countries](https://restcountries.com/) | Country profiles (capital, languages, currency) |
| Holidays | [Nager.Date](https://date.nager.at/) | Public holidays by country and year |

---

### 2. `ACP Agents.ipynb` — Agent Communication Protocol

**Protocol:** [ACP (Agent Communication Protocol)](https://agentcommunicationprotocol.dev/) by BeeAI

**What it teaches:**
ACP is a **horizontal integration** protocol — it connects autonomous agents to each other over standard HTTP. Unlike MCP where one LLM calls tools, ACP lets agents delegate work to peer agents using natural language. Each agent is independently deployable and decides internally how to fulfill requests.

**Architecture:**
```
┌──────────────┐     ACP/HTTP      ┌──────────────┐     ACP/HTTP     ┌──────────────┐
│   Notebook   │ ───────────────►  │   Advisor    │ ────────────────► │   Data       │ ──► Real APIs
│   (Client)   │   port 8001       │    Agent     │   port 8000      │   Agents     │
└──────────────┘                   └──────────────┘                   └──────────────┘
```

**How it works:**
- **Data Agent Server** (`data_agent.py`, written via `%%writefile`) hosts three autonomous agents on port 8000:
  - `weather_agent` — geocodes a city via Open-Meteo, returns weather report
  - `country_agent` — queries REST Countries, returns country profile
  - `holiday_agent` — queries Nager.Date, returns public holidays
- **Advisor Agent Server** (`advisor_agent.py`, written via `%%writefile`) runs on port 8001:
  1. **Discovers** available data agents via `GET /agents`
  2. **Plans** which agents to call using LLM (outputs JSON plan)
  3. **Executes** the plan by sending ACP messages to data agents in parallel
  4. **Synthesizes** results into a travel advisory using LLM
- **Notebook Client** sends a single message to the Advisor and displays the response

**Key concepts covered:**
- Agent discovery via `GET /agents` endpoint
- ACP's `@server.agent` decorator pattern
- LLM-driven planning and delegation
- Parallel agent invocation
- Natural language inter-agent communication

**Dependencies:** `acp-sdk==1.0.3`, `langchain-openai`, `httpx`, `uvicorn`

**Files generated at runtime (via `%%writefile`):**
| File | Lines | Port | LLM? | Purpose |
|------|-------|------|------|---------|
| `data_agent.py` | ~180 | 8000 | No | Three data agents wrapping free APIs |
| `advisor_agent.py` | ~150 | 8001 | Yes | LLM orchestrator that delegates to data agents |

---

### 3. `A2A Agents.ipynb` — Agent2Agent Protocol

**Protocol:** [A2A (Agent2Agent)](https://google.github.io/A2A/) by Google / Linux Foundation

**What it teaches:**
A2A is the **industry-standard** protocol for agent-to-agent communication, backed by 50+ partners including Google, AWS, Microsoft, Salesforce, and SAP under the Linux Foundation. It uses JSON-RPC 2.0 as the wire protocol and introduces **Agent Cards** — JSON documents that describe an agent's capabilities, analogous to OpenAPI specs for agents.

**Architecture:**
```
┌──────────────┐     A2A/HTTP      ┌──────────────┐     A2A/HTTP     ┌──────────────┐
│   Notebook   │ ───────────────►  │   Advisor    │ ────────────────► │   Data       │ ──► Real APIs
│   (Client)   │   port 8001       │    Agent     │   port 8000      │   Agents     │
└──────────────┘                   └──────────────┘                   └──────────────┘
```

**How it works:**
- **Data Agent Server** (`data_agent_a2a.py`, written via `%%writefile`) runs on port 8000:
  - Publishes an **Agent Card** at `/.well-known/agent.json` listing three skills
  - A single `DataAgentExecutor` routes requests by message prefix (`weather:`, `country:`, `holiday:`)
  - Each skill calls its respective free API and returns results via the event queue
- **Advisor Agent Server** (`advisor_agent_a2a.py`, written via `%%writefile`) runs on port 8001:
  1. **Discovers** data agent skills by fetching the Agent Card (`GET /.well-known/agent.json`)
  2. **Plans** which skills to invoke using LLM (outputs JSON plan)
  3. **Executes** the plan by sending JSON-RPC `message/send` requests to the data agent in parallel
  4. **Synthesizes** results into a travel advisory using LLM
- **Notebook Client** constructs raw JSON-RPC 2.0 requests to the Advisor, showing the actual wire protocol

**Key concepts covered:**
- **Agent Cards** — the A2A discovery mechanism (`/.well-known/agent.json`)
- **Skills** — capabilities listed on Agent Cards
- **AgentExecutor** pattern — implement `execute()` and `cancel()` methods
- **JSON-RPC 2.0** wire protocol (`message/send`)
- **A2AStarletteApplication** — server wiring with `DefaultRequestHandler` and `InMemoryTaskStore`
- Raw protocol inspection (students see exact JSON-RPC request/response format)

**Dependencies:** `a2a-sdk[http-server]==0.3.25`, `langchain-openai`, `httpx`, `uvicorn`

**Files generated at runtime (via `%%writefile`):**
| File | Lines | Port | LLM? | Purpose |
|------|-------|------|------|---------|
| `data_agent_a2a.py` | ~217 | 8000 | No | Three skills (weather/country/holiday) on one AgentExecutor |
| `advisor_agent_a2a.py` | ~244 | 8001 | Yes | LLM orchestrator using A2A discovery + JSON-RPC delegation |

---

## Protocol Comparison

| | **MCP** | **ACP** | **A2A** |
|---|---------|---------|---------|
| **Direction** | Vertical (LLM → tools) | Horizontal (agent → agent) | Horizontal (agent → agent) |
| **Discovery** | Tool schemas | `GET /agents` | Agent Card at `/.well-known/agent.json` |
| **Transport** | stdio / SSE | REST HTTP | JSON-RPC 2.0 over HTTP |
| **Server pattern** | Tool functions | `@server.agent` decorator | `AgentExecutor` class |
| **Unit of work** | Function call | Full agent run | Full agent run |
| **Who decides how?** | The LLM picks tools + params | The receiving agent | The receiving agent |
| **Governance** | Anthropic | BeeAI (open-source) | Linux Foundation (50+ partners) |
| **Best for** | Giving one LLM access to data | Quick Python-native agent composition | Industry-standard interoperability |

All three are **complementary**: use MCP inside agents for tool access, A2A or ACP between agents for collaboration.

---

## Demo Queries

Each notebook runs the same four progressive demos for easy comparison:

| # | Query | Skills/Tools Used |
|---|-------|-------------------|
| 1 | "What's the weather in Tokyo?" | Weather only |
| 2 | "Tell me about France — languages, currency, weather in Paris?" | Country + Weather |
| 3 | "Complete travel brief for Japan — country, Tokyo weather, holidays 2026" | Country + Weather + Holidays |
| 4 | "Brazil vs Thailand for a beach holiday — compare both" | Country x2 + Weather x2 (LLM picks cities) |

---

## Prerequisites

- **Google Colab** (recommended) or Jupyter Notebook
- **OpenAI API key** — stored in `config.json`, Colab secrets, or entered at runtime
- All data APIs are **free and require no API keys** (Open-Meteo, REST Countries, Nager.Date)

### Running a Notebook

1. Open the `.ipynb` file in Google Colab
2. Run the `pip install` cell, then **restart the runtime**
3. Run all remaining cells from top to bottom
4. When prompted, provide your OpenAI API key (or pre-configure it in Colab secrets as `OPENAI_API_KEY`)

The `config.json` format (if providing credentials via file):
```json
{
  "OPENAI_API_KEY": "your-key-here",
  "OPENAI_API_BASE": "https://api.openai.com/v1"
}
```

---

## LLM Configuration

All notebooks use `langchain-openai` with `ChatOpenAI` configured for `gpt-4o-mini`. The `load_openai_config()` function searches for credentials in this order:

1. `config.json` in the working directory
2. Environment variables (`OPENAI_API_KEY`, `OPENAI_API_BASE`)
3. Google Colab secrets (`userdata.get()`)
4. Interactive prompt (`getpass`)

To use a custom OpenAI-compatible endpoint, set `OPENAI_API_BASE` in any of the above locations.

---

## Support

If you enjoy this project, buy me some tokens: [buymeacoffee.com/amircs](https://buymeacoffee.com/amircs)
