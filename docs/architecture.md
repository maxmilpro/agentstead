# Agentstead Technical Architecture

**Version:** 0.1 (MVP design)
**Status:** Draft — subject to revision as implementation begins

---

## Design Philosophy

Every architectural decision is made against three constraints, in order:

1. **Runs locally.** If it requires an internet connection or a cloud service to function at all, it is not acceptable for the core stack.
2. **Data stays home.** All persistent state is under the operator's control by default. Egress is opt-in, logged, and reversible.
3. **Composable, not monolithic.** Components are replaceable. The harness does not own the model. The memory layer does not own the harness. The deployment tooling does not own either.

---

## System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Agentstead Homestead                    │
│                                                             │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────────┐  │
│  │  Agent       │   │  Memory      │   │  Tool          │  │
│  │  Harness     │◄──│  Store       │   │  Registry      │  │
│  │  (runtime)   │   │  (local)     │   │  (plugins)     │  │
│  └──────┬───────┘   └──────────────┘   └───────┬────────┘  │
│         │                                        │          │
│         └────────────────┬─────────────────────-┘          │
│                          │                                  │
│  ┌───────────────────────▼──────────────────────────────┐  │
│  │              Inference Layer                          │  │
│  │  ┌─────────────┐  ┌────────────┐  ┌──────────────┐  │  │
│  │  │  Ollama     │  │  llama.cpp │  │  Remote API  │  │  │
│  │  │  (default)  │  │  (direct)  │  │  (optional)  │  │  │
│  │  └─────────────┘  └────────────┘  └──────────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────────┐  │
│  │  API Server  │   │  Scheduler   │   │  Audit Log     │  │
│  │  (REST/WS)   │   │  (cron/event)│   │  (local)       │  │
│  └──────────────┘   └──────────────┘   └────────────────┘  │
│                                                             │
└──────────────────────────┬──────────────────────────────────┘
                           │ Tailscale / local network
              ┌────────────┴───────────┐
              │  Operator interfaces    │
              │  CLI │ API │ (UI later) │
              └────────────────────────┘
```

---

## Component Breakdown

### 1. Agent Harness (Runtime Core)

**Language:** Python 3.12+ (pragmatic: best ecosystem for LLM tooling, async-native)

**Execution model:**

The harness runs agents as stateful coroutines. Each agent instance is:
- A configuration (loaded from YAML/TOML at startup)
- A memory handle (connection to the agent's memory store partition)
- A tool set (resolved from the tool registry at startup)
- An inference client (abstracted behind a common interface)

The core agent loop:

```
while running:
    input = await receive_input()          # user message, scheduled trigger, event
    context = memory.retrieve(input)       # relevant memory chunks
    plan = await infer(input, context)     # LLM call: what to do
    actions = parse_tool_calls(plan)       # extract structured tool calls
    results = await execute_tools(actions) # run tools, collect results
    response = await infer(results)        # LLM call: synthesize response
    memory.store(input, response, results) # persist to memory
    await emit_response(response)          # return to caller
```

**Key abstractions:**

- `InferenceClient` — unified interface over Ollama, llama.cpp HTTP, OpenAI-compatible APIs. Implementations are swappable via config. Supports streaming.
- `MemoryStore` — interface for read/write/search operations on agent memory. Default implementation: SQLite (structured data) + LanceDB (vector search). Interface allows alternative backends.
- `ToolPlugin` — each tool is a Python class implementing a standard interface: `name`, `description`, `parameters` (JSON Schema), `execute(params) -> result`. Loaded via plugin discovery at startup.
- `AgentConfig` — Pydantic model representing the full agent specification. Versioned schema with migration support.

---

### 2. Inference Layer

**Primary path: Ollama**

Ollama is the default inference backend. It handles model management (pull, quantize, serve) and exposes an OpenAI-compatible API locally. The Agentstead model CLI wraps Ollama with additional metadata for capability-based model selection.

**Model selection strategy:**

Agentstead ships with a model capability registry — a YAML file mapping model names to their tested capability profile:

```yaml
models:
  llama3.2:3b:
    capabilities: [chat, tool_use, summarization]
    context_window: 128000
    min_ram_gb: 4
    quality_tier: fast
  qwen2.5:14b:
    capabilities: [chat, tool_use, coding, reasoning]
    context_window: 128000
    min_ram_gb: 10
    quality_tier: balanced
  qwen2.5:72b:
    capabilities: [chat, tool_use, coding, reasoning, analysis]
    context_window: 128000
    min_ram_gb: 48
    quality_tier: high
```

At startup, Agentstead probes available hardware (RAM, VRAM, GPU type) and recommends a model tier. The operator can override.

**Apple Silicon optimization:**

For Mac mini deployments (a primary target), Agentstead configures Ollama with:
- Metal backend enabled (GPU acceleration via MLX path)
- Appropriate context length for available unified memory
- Batch size tuned for Metal's memory model

**Fallback to cloud APIs:**

When configured, cloud APIs (Anthropic, OpenAI, Groq, etc.) are available as inference backends. This is never the default. When used, all API calls are logged to the audit trail. The operator can set per-agent policies: "only use local models", "allow cloud for tasks requiring > X capability tier".

---

### 3. Memory System

Memory is structured in three tiers, each with different persistence and retrieval semantics:

**Tier 1: Working memory (in-process)**
- The current conversation context window
- Ephemeral — lives only for the duration of a session
- Managed by the harness directly

**Tier 2: Episodic memory (session store)**
- Summaries of completed sessions
- Structured records of actions taken and their outcomes
- Storage: SQLite (append-only, local file)
- Retrieval: recency-weighted, SQL query

**Tier 3: Semantic memory (vector store)**
- Long-term knowledge about the operator, their projects, their preferences
- Indexed as embeddings for semantic search
- Storage: LanceDB (embedded, local file, no separate server)
- Retrieval: cosine similarity search, filtered by recency/relevance

**Memory isolation:**

Each agent has its own memory partition — a separate SQLite database file and a separate LanceDB table. Agents cannot read each other's memory unless explicitly granted access via the data access policy. The orchestrator agent (if used) has read-only access to sub-agent episodic summaries.

**Embedding model:**

Embeddings are generated locally using a small dedicated embedding model (default: `nomic-embed-text` via Ollama). No cloud embedding API.

**Portable snapshot format:**

```
agent-snapshot-<name>-<timestamp>.agentstead
├── manifest.json          # schema version, agent identity, export timestamp
├── config.toml            # full agent configuration
├── memory/
│   ├── episodic.sqlite    # episodic memory database
│   └── semantic.lance/    # LanceDB vector store directory
└── identity/
    └── did.json           # DID document (did:key)
```

---

### 4. Tool System

Tools are the agent's hands. The tool system is intentionally simple and extensible.

**Plugin discovery:**

At startup, the harness scans `~/.agentstead/tools/` and the built-in tool directory for Python files matching the tool plugin interface. Plugins are loaded dynamically. No restart required to add a tool (hot reload supported).

**Built-in tools (MVP):**

| Tool | Description | Egress? |
|------|-------------|---------|
| `read_file` | Read file from local filesystem | No |
| `write_file` | Write file to local filesystem | No |
| `run_command` | Execute shell command (sandboxed) | No |
| `web_search` | Search the web via SearXNG (self-hosted) | Yes (to SearXNG instance) |
| `fetch_url` | Fetch and parse a URL | Yes |
| `github_api` | GitHub API operations | Yes |
| `calendar_read` | Read local calendar (ical) | No |
| `memory_search` | Search agent's own memory | No |

**Sandboxing:**

`run_command` executes in a restricted subprocess with:
- No network access (via `unshare` on Linux, restricted on macOS via `sandbox-exec`)
- No write access outside a designated scratch directory
- CPU/memory resource limits
- Execution timeout (default: 30s, configurable per-agent)

**Tool result auditing:**

Every tool call (name, parameters, result, latency) is logged to the agent's audit trail. The operator can query: "what did my agent do in the last 24 hours?"

---

### 5. Identity Layer (AT Protocol foundation)

**MVP: did:key**

Each agent is provisioned with a `did:key` DID at initialization. This is a locally-generated keypair; the public key is the identifier. No external service required.

The DID is used to:
- Sign agent outputs (outputs include a detached signature for authenticity verification)
- Identify agents in multi-agent interactions
- Anchor the portable snapshot (the DID is stable across migrations)

**Post-MVP: did:plc**

The DID will be upgraded to `did:plc` (AT Protocol's resolvable DID method) for cross-homestead federation. This enables:
- Agent-to-agent trust establishment across homesteads
- Agent marketplace reputation anchored to a stable identity
- Integration with AT Protocol's data portability layer (your agent's public outputs as a PDS record)

The investment in DID-based identity in MVP is specifically to make this upgrade non-breaking.

---

### 6. API Server

**Framework:** FastAPI (async Python, OpenAPI schema auto-generated)

**Endpoints (MVP):**

```
POST   /v1/agents/{agent_id}/chat          # send a message, get a response
POST   /v1/agents/{agent_id}/task          # submit an async task
GET    /v1/agents/{agent_id}/tasks/{id}    # poll task status/result
GET    /v1/agents/{agent_id}/memory        # query agent memory
DELETE /v1/agents/{agent_id}/memory/{id}  # delete a memory entry (right to forget)
GET    /v1/agents                          # list configured agents
GET    /v1/health                          # health check
GET    /v1/audit                           # query audit log
```

The API is OpenAI chat-compatible on the `/v1/agents/{agent_id}/chat` endpoint, meaning existing OpenAI-compatible clients work with Agentstead agents without modification.

**Authentication:**

MVP uses simple bearer token auth (token generated at setup, stored locally). Tokens are scoped: a token can be scoped to a specific agent, or to read-only operations. Post-MVP: mTLS for service-to-service, proper OIDC for user-facing clients.

**Tailscale ACL integration:**

In the recommended deployment, the API server is only exposed on the Tailscale interface (not the public internet). Tailscale ACLs provide additional access control at the network layer.

---

### 7. Deployment Stack

**docker-compose.yml (core services):**

```yaml
services:
  harness:          # agent runtime (Python)
  ollama:           # local inference server
  lancedb:          # embedded in harness, no separate service
  searxng:          # self-hosted web search (optional)
  scheduler:        # cron/event trigger service
  audit:            # audit log aggregator + query API
```

**Configuration hierarchy:**

```
~/.agentstead/
├── config.toml           # global config (ports, paths, hardware profile)
├── agents/
│   ├── personal.toml     # personal assistant agent config
│   └── dev.toml          # dev agent config
├── tools/                # user-installed tool plugins
├── models/               # model capability registry (local override)
└── memory/               # all agent memory stores (partitioned by agent)
    ├── personal/
    └── dev/
```

**CLI:**

```bash
agentstead init                    # initialize a new homestead
agentstead agent create <name>     # create a new agent
agentstead agent start <name>      # start an agent
agentstead model pull <name>       # pull and register a model
agentstead model recommend         # recommend models for this hardware
agentstead snapshot export <agent> # export portable snapshot
agentstead snapshot import <file>  # import snapshot
agentstead audit <agent> --since 24h  # query audit log
agentstead update                  # update all components
```

---

## Tech Stack Summary

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Runtime language | Python 3.12+ | Best LLM ecosystem; async support |
| Inference server | Ollama | Best local model management; OpenAI-compatible |
| Vector store | LanceDB | Embedded (no server), fast, Rust core |
| Structured store | SQLite | Zero-ops, reliable, portable |
| API framework | FastAPI | Async, auto-schema, production-grade |
| Deployment | Docker Compose | Ubiquitous, reproducible, understood |
| Identity | did:key → did:plc | Standards-compliant, portable, AT Protocol-aligned |
| CLI | Typer (Python) | Consistent with runtime language, good UX |
| Config format | TOML | Human-readable, widely supported |
| Embedding | nomic-embed-text | Open, local, strong multilingual performance |
| Network access | Tailscale | Zero-config mesh VPN, ideal for home server |

---

## Local Model Optimization Strategy

The core thesis is that local models are good enough for the primary use cases of personal agents. The optimization strategy operates at three levels:

**1. Model selection:** Match model capability to task requirements. A scheduling assistant does not need a 70B model. Agentstead's capability registry enables automatic right-sizing.

**2. Context management:** Long-context models are more expensive per token. Agentstead's memory tiering ensures only relevant context is included in each inference call — semantic search retrieves the most relevant memories rather than dumping the full history.

**3. Hardware utilization:** On Apple Silicon, Metal acceleration via Ollama/llama.cpp achieves near-GPU performance on unified memory. On NVIDIA GPUs, CUDA acceleration is automatically configured. CPU-only deployments use quantized models (Q4_K_M or Q5_K_M) that trade minimal quality for significant speed and memory gains.

**Benchmark targets (Mac mini M4 Pro, 24GB RAM):**

| Model | Quantization | Tokens/sec | Notes |
|-------|-------------|-----------|-------|
| llama3.2:3b | Q8 | ~80 t/s | Fast assistant tasks |
| qwen2.5:14b | Q4_K_M | ~25 t/s | Balanced; primary workhorse |
| qwen2.5:32b | Q4_K_M | ~10 t/s | Complex reasoning tasks |

These targets are sufficient for interactive agent use (user-facing response < 3 seconds for typical queries at the 14B tier).

---

## Data Sovereignty Architecture

Data sovereignty is enforced architecturally, not just by policy:

**No default outbound connections.** The harness initializes with all egress disabled. Each egress capability (web search, API calls, cloud inference) is explicitly enabled in configuration and logged.

**Filesystem isolation.** All agent data lives under `~/.agentstead/memory/`. Backups, exports, and migrations are operator-controlled operations on this directory.

**Audit trail as first-class feature.** Every action the agent takes — every inference call, every tool invocation, every file touched — is recorded in a local, queryable audit log. The operator can always answer: "what did my agent do, and with what data?"

**Right to forget.** The API exposes a DELETE endpoint for individual memory entries. The CLI provides `agentstead memory purge <agent>` for complete memory deletion. Deletion is permanent and immediate.

**Zero telemetry by default.** Agentstead collects no usage metrics, no crash reports, no model usage statistics. If a future hosted service is offered, telemetry will be strictly opt-in and scoped to operational data needed for that service only.

---

## Post-MVP Roadmap (Architecture Implications)

These are not MVP scope, but the MVP architecture explicitly accommodates them:

- **Web UI:** FastAPI serves static files; a React/SvelteKit frontend is a natural addition without architectural change.
- **Agent federation:** The DID identity layer and API design support inter-homestead agent communication. The primary design constraint is the authentication model, which is left extensible.
- **Compute sharing network:** A peer-to-peer inference sharing network can be layered over the existing inference client abstraction. Homesteads register available capacity; the client routes requests to available peers with appropriate trust policies.
- **AT Protocol PDS integration:** Agents publishing outputs to a user's AT Protocol PDS (personal data server) is a natural extension of the identity layer. The agent's DID links its outputs to its identity.
- **Fine-tuning pipelines:** The audit log and memory store provide the data foundation for generating fine-tuning datasets from real agent interactions. This is the "your agent learns from your data, not theirs" story.
