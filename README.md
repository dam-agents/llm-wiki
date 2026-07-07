# LLM Wiki Agent

A Claude Code agent that builds and maintains AI-curated knowledge wikis for GitHub repositories.

It implements [Karpathy's LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f): the agent reads a source repo, distills its architecture and design decisions into durable, cited markdown, and keeps it current as the code evolves. The wiki is precomputed and maintained over time — not generated per query (unlike RAG). This repo is the agent's definition; the wiki it produces lives in a separate content repo.

## How it works

The agent runs headless and organizes everything into three layers:

- **Sources** — the GitHub repos being documented. Shallow-cloned into a gitignored `sources/`, read-only.
- **Wiki** — the product: `wiki/pages/`, `wiki/index.md`, `wiki/log.md`. `wiki/` is its own git repo, and the only thing pushed to the wiki remote, so the content outlives the pod.
- **Schema & tooling** — this repo: `CLAUDE.md` (the operating manual), `skills/`, `scripts/`, and the per-instance `wiki.config.json` + `THIS-WIKI.md`. Agent-owned — they sit on the pod's PVC beside `wiki/`, never inside it, so only content is ever pushed.

Four skills drive it, each logged in `wiki/log.md`:

- **onboard** — interactive, once: interview for purpose, sources, taxonomy, and cadence; schedule maintenance; run the first ingest.
- **ingest** — delta-ingest new commits, then commit and push silently (scheduled).
- **lint** — refresh stale pages, resolve contradictions, fix orphans and broken links (scheduled, after ingest).
- **query** — answer a question from the wiki with citations (on request, via Slack / Web UI).

Every page pins its provenance (`source`, `commit`, `files`) in frontmatter, and every claim is cited inline as `path:line @sha`.

Maintenance is scheduled through the `platform-outbound` MCP `create_schedule` tool — the only valid scheduler inside a Platform pod. Only `wiki/` survives pod deletion, via its remote; a fresh agent re-clones it and rebuilds `wiki.config.json` from each page's provenance.

## Setup

Bringing up a new agent takes one message. On the platform:

1. Create an agent from the `claude-code` template with a **GitHub** connection — `gh` uses it to clone this repo and the source repos, and to push the wiki. Add **Slack** for query delivery.
2. Send it this as the first message:

   > Here is a file, read it and set yourself up according to it: https://github.com/dam-agents/llm-wiki/blob/main/INSTALLATION.md

The agent then reaches steady state in two phases, both self-run:

- **Installation** ([`INSTALLATION.md`](INSTALLATION.md)) — one-shot, idempotent machine setup: makes this repo its working dir, surfaces the skills, protects the instance files. Does **not** interview, schedule, or ingest.
- **Onboarding** (the `onboard` skill) — the interview: purpose, sources, taxonomy, cadence, and the wiki remote (verified pushable, or it aborts and asks you to create it). Writes config, schedules maintenance, runs the first ingest.

Installation ends by offering onboarding, so the operator only ever sends one message. No environment variables are required — sources and the wiki remote come from the interview, not from config injected at create time.

## Files

- [`CLAUDE.md`](CLAUDE.md) — the full operating manual loaded by the agent.
- [`INSTALLATION.md`](INSTALLATION.md) — first-run machine-setup runbook.
- [`THIS-WIKI.md`](THIS-WIKI.md) — per-instance specialization, written at onboard.
- [`skills/`](skills/) — the four workflow skills (`onboard`, `ingest`, `lint`, `query`).
- [`scripts/`](scripts/) — thin deterministic helpers.
- [`wiki.config.json`](wiki.config.json) — per-instance configuration seed.
