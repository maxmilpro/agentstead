# Agentstead MVP: Product Requirements Document

**Version:** 0.1 (pre-alpha planning)
**Status:** Draft
**Scope:** MVP — the minimum viable homestead

---

## Overview

The Agentstead MVP is a locally-deployable agent runtime stack. It is the thing you install on your home server (or a cheap VPS, or a Raspberry Pi 5) to get a personal AI agent infrastructure that you own end-to-end.

The MVP is not feature-rich. It is foundation-solid. Every decision prioritizes: does this work reliably, locally, on hardware ordinary people own?

---

## Problem Statement

Running a personal AI agent today requires either:
1. Surrendering your data and agency to a hosted platform, or
2. Stitching together a fragile collection of open-source tools with no coherent operational model

Neither is acceptable. Agentstead solves this by providing a coherent, opinionated, self-hostable stack — with sensible defaults, good documentation, and a path to extensibility.

---

## Target Users (MVP)

**Primary:** Technical individuals — developers, sysadmins, data-literate professionals — who self-host services and want to add an owned agent stack to their infrastructure. Comfortable with Docker and a terminal. Not comfortable babysitting brittle integrations.

**Secondary:** Small teams (2–5 people, e.g. a small business or household) sharing an agent instance on shared infrastructure.

**Explicitly out of scope for MVP:** Non-technical users, enterprise deployments, multi-tenant SaaS.

---

## MVP Scope

### 1. Agent Harness (Core Runtime)

The agent harness is the execution environment for agents. It handles the core loop: perceive → plan → act → remember.

**Requirements:**

- **Local-model-first execution.** The default runtime uses locally-running models via [Ollama](https://ollama.com/) or a compatible inference server. Cloud model APIs (OpenAI, Anthropic, etc.) are supported as optional drop-in providers — never the default.
- **Structured tool use.** Agents can use tools (file I/O, web search, code execution, API calls) via a standard tool-call interface. Tools are registered as plugins; the core harness has no hardcoded capabilities.
- **Persistent memory.** Each agent has a memory store: short-term (conversation context), medium-term (session summaries), and long-term (vector-indexed knowledge). Memory is stored locally by default (SQLite + a local vector store like Chroma or LanceDB). Memory is portable — exportable and importable as a standard format.
- **Multi-agent coordination (basic).** MVP supports a simple orchestrator pattern: one coordinator agent can spawn and direct sub-agents for subtasks. Full multi-agent mesh is post-MVP.
- **Configuration as code.** Agents are defined in YAML/TOML configuration files. No GUI required for core operation (GUI is a later milestone). An agent config specifies: model, tools, memory settings, identity, and behavioral guidelines.

**Out of scope for MVP:** Agent-to-agent networking across homesteads, real-time streaming UI, voice interface.

---

### 2. Deployment and Infrastructure Tooling

The stack must be trivially deployable on the hardware the target users already own.

**Requirements:**

- **Docker Compose-based deployment.** The full stack (agent runtime, memory stores, inference server, optional UI) ships as a composable Docker Compose configuration. Single command to start: `docker compose up`.
- **Hardware profiles.** Pre-configured compose profiles for common hardware targets:
  - `profile: mac-mini` — Apple Silicon, Metal acceleration, high memory
  - `profile: linux-gpu` — NVIDIA/AMD GPU acceleration
  - `profile: cpu-only` — x86/ARM CPU-only, optimized model selection
- **Tailscale integration.** First-class support for Tailscale-based access: the stack can expose its API and optional UI over a Tailscale network without additional configuration. This is the recommended access pattern for home server deployments.
- **Model management CLI.** A CLI tool (`agentstead model`) for pulling, quantizing, and managing local model weights. Wraps Ollama and adds Agentstead-specific metadata (capability tags, memory requirements, recommended use cases).
- **Health and observability.** A `/health` endpoint and local-only metrics dashboard (no external telemetry by default). Logs are structured JSON, locally stored, rotated automatically.
- **Upgrade path.** `agentstead update` pulls new images and migrates configuration/data with no manual intervention. Migrations are versioned and reversible.

---

### 3. Data Ownership Primitives

Data ownership is not a feature. It is an architectural invariant. These primitives enforce it structurally.

**Requirements:**

- **Local-first data storage.** All agent data — memory, conversation history, outputs, configuration — is stored in the local filesystem by default. Nothing leaves the homestead unless explicitly configured to do so.
- **Explicit egress consent.** Any operation that would send data to an external service (cloud model API, web search, webhook) requires explicit configuration. The runtime logs all egress events to a local audit trail.
- **Portable agent snapshots.** An agent's full state — memory, configuration, history — can be exported to a self-contained archive (`.agentstead` format, which is a tar of structured data + metadata). This snapshot can be imported on any other Agentstead instance. Agents are migratable.
- **Identity layer (basic AT Protocol integration).** Each agent has an identity anchored to a DID (Decentralized Identifier). In MVP, DIDs are `did:key` (locally generated). Post-MVP: `did:plc` integration with AT Protocol for federated agent identity. This investment means agent identities are portable and not tied to any Agentstead instance.
- **Data access control.** Each data store (memory, documents, conversation history) has an access policy: who/what can read it, who/what can write it. In MVP, policies are simple (owner-only or shared with named Tailscale peers). Post-MVP: richer policy language.

---

### 4. The SDLC Automation Angle: Agents that Build Agents

The dogfood use case: Agentstead is built and maintained, in part, by agents running on Agentstead.

**MVP Requirements:**

- **Dev agent persona.** A pre-configured agent persona (`dev`) that has tools for: reading/writing files, running shell commands (sandboxed), searching the web, and calling the GitHub API. This is the "coding assistant" persona for personal use.
- **Spec-to-scaffold workflow.** Given a plain-language description of a new agent or tool, the dev agent can generate: a configuration file, a tool plugin stub, and a basic test. Not magic — structured prompting with templates — but functional.
- **Self-improvement loop (basic).** The dev agent can be tasked with: reading its own configuration, identifying improvement areas (via a structured review prompt), and proposing changes as a diff for human approval. The human approves; the agent applies the change. This is the seed of a self-improving agent system.

This is not the main story for MVP, but it is the proof of concept for the long-term direction.

---

## Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| Time-to-first-agent | < 30 minutes from `git clone` to running agent on Mac mini |
| Idle resource usage | < 2GB RAM (without model loaded) |
| Startup time | < 60 seconds (cold start, model already pulled) |
| Data portability | Full agent snapshot export/import in < 5 minutes |
| External dependencies | Zero required cloud dependencies for core function |
| License | Apache 2.0 (copyleft-compatible, commercially usable) |

---

## Monetization Direction (MVP-Compatible)

Agentstead is open source. The monetization model does not compromise that. Candidate approaches, in order of alignment with the project values:

1. **Capability marketplace.** A community marketplace for agent personas, tool plugins, and workflow templates. Creators publish; users pay creators directly (or for free). Agentstead takes a small platform fee on paid listings. Open listings are always free.

2. **Managed homestead hosting.** For users who want ownership semantics without self-hosting: a hosted Agentstead instance where the user retains full data export rights, no vendor lock-in, and portable snapshots at any time. This is "your data, our ops" — not a proprietary platform.

3. **Compute sharing network (post-MVP).** Homestead operators with spare GPU capacity can contribute to a peer-to-peer inference network and earn credits. Users who need more compute than their hardware provides can spend credits. No central intermediary owns the compute.

4. **Enterprise/team features.** Multi-user homestead administration, audit logging, policy management, SSO. Available as a paid add-on to the open-source core.

**Anti-patterns we will not pursue:** Data monetization. API metering that penalizes usage. Features that degrade when you don't pay. Licensing that restricts self-hosting.

---

## Success Metrics (MVP)

- 100 self-hosted deployments within 90 days of public release
- Mean time-to-working-agent (first meaningful agent task completion) < 45 minutes
- Community contributions: at least 5 externally-contributed tool plugins within 60 days
- Zero reported incidents of unexpected data egress
- At least one person uses Agentstead to build something they couldn't have built without it

---

## Out of Scope (MVP)

- Web UI (CLI and API only for MVP)
- Mobile clients
- Voice interface
- Agent-to-agent federation across homesteads
- Windows support (macOS and Linux only for MVP)
- Fine-tuning or training pipelines
- Commercial hosted offering (designed for, not built yet)
