# Agentstead

> Own your agent infrastructure. All of it.

Agentstead is an open, locally-deployable AI agent runtime. Run your own agents on your own hardware, with your own models, against your own data. No cloud required. No vendor lock-in. No surveillance.

---

## What it is

A composable stack for self-hosting personal AI agents:

- **Local-model-first** — runs on Ollama/llama.cpp; cloud APIs are optional, never required
- **Memory that's yours** — all agent memory (episodic + semantic) stored locally, portable, deletable
- **Tool-extensible** — add capabilities via a simple plugin interface
- **Docker Compose deployable** — runs on a Mac mini, a Linux box, or anything in between
- **Tailscale-native** — designed for home server / private network access

---

## Status

Early planning stage. See `docs/` for vision, product requirements, and architecture design.

- [Vision](docs/vision.md) — why this exists
- [MVP PRD](docs/prd-mvp.md) — what we're building first
- [Architecture](docs/architecture.md) — how it works

---

## Principles

1. Data stays on your hardware unless you explicitly send it somewhere
2. Every external call is logged
3. Agents are portable — export, import, migrate freely
4. Local models are good enough; the default proves it
5. Monetization serves users, not the other way around

---

## License

Apache 2.0
