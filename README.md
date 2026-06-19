# llm-wiki

A Claude Code agent that maintains an **AI-curated, interlinked markdown wiki**
about one or more GitHub repositories. It implements the LLM-Wiki
pattern: distil source repos into durable, cited markdown that is precomputed and
kept current, rather than chunk-retrieved per query (RAG). The value is the
maintenance that runs when nobody is asking.

This repo is the agent's **definition**. There is no custom image: the agent runs
on the generic `claude-code` template and installs itself from this repo at
runtime (see **Setup**). Edit the files here and the running agent picks them up on
its next definition pull.

## The three layers

1. **Sources** — the GitHub repos being documented. Shallow-cloned into a
   gitignored `sources/`, read-only.
2. **Wiki** — the maintained markdown under `wiki/`: `wiki/pages/`,
   `wiki/index.md`, `wiki/log.md`. This is the product. `wiki/` is its own git
   repo and the only thing pushed to the wiki remote, so the remote stays a pure
   content database that outlives the pod.
3. **Schema & tooling** — `CLAUDE.md` (the operating manual), the `skills/`, the
   `scripts/`, and the per-instance files `wiki.config.json` + `THIS-WIKI.md`.
   Agent-owned: they live on the pod's PVC beside `wiki/`, never inside it.

## Two phases: installation, then onboarding

The agent reaches steady state in two distinct phases, both run by the agent
itself, off a single first message from the operator:

1. **Installation** ([`INSTALLATION.md`](INSTALLATION.md)) — machine setup. Makes
   `/home/agent/work` this repo, surfaces the skills, protects the instance files,
   writes a sentinel. One-shot and idempotent. Does **not** interview, schedule,
   or ingest.
2. **Onboarding** (the `onboard` skill) — the human interview. Asks for the wiki's
   purpose, source repos, taxonomy, cadence, and the git remote to push the wiki
   to; verifies the remote; writes `wiki.config.json` and `THIS-WIKI.md`; schedules
   recurring maintenance; runs the first ingest.

Installation ends by offering onboarding, so the operator only ever sends one
message.

## Setup

Bringing up a new llm-wiki agent takes three steps:

1. **Create the agent** on the platform from the generic `claude-code` template,
   with a **GitHub** connection granted (used by `gh` to clone this repo, the
   source repos, and to push the wiki). Grant **Slack** too if you want query
   delivery to a channel.
2. **Grab the link to [`INSTALLATION.md`](INSTALLATION.md)** — it is:
   `https://github.com/dam-agents/llm-wiki/blob/main/INSTALLATION.md`
3. **Tell the agent**, in its first message:

   > Here is a file, read it and set yourself up according to it: https://github.com/dam-agents/llm-wiki/blob/main/INSTALLATION.md

The agent reads the runbook, installs itself, then offers to onboard. Answer its
interview and it points itself at the repos you want documented, schedules its own
maintenance, and runs the first ingest. From then on it runs autonomously.

llm-wiki needs **no environment variables**: the sources and the wiki remote come
from the onboarding interview, not from config injected at create time.

## How the agent runs after onboarding

- **onboard** (manual, once) — interview, write config + `THIS-WIKI.md`,
  self-schedule maintenance, run the first ingest.
- **ingest + lint** (scheduled) — delta-ingest new commits, then lint; commit and
  push the wiki silently.
- **query** (Slack / Web UI) — answer from the wiki with citations.

Maintenance is scheduled through the `platform-outbound` MCP `create_schedule`
tool — the only valid scheduler inside a Platform pod. Across agent deletion only
the `wiki/` content survives, via its git remote configured at onboarding; a fresh
agent clones it and reconstructs `wiki.config.json` from page provenance.

## Files

- [`CLAUDE.md`](CLAUDE.md) — full operating manual loaded by the agent.
- [`INSTALLATION.md`](INSTALLATION.md) — first-run machine-setup runbook.
- [`THIS-WIKI.md`](THIS-WIKI.md) — per-instance specialisation, written at onboard
  (seed reads _Not yet onboarded_).
- [`skills/`](skills/) — the four wiki workflow skills (`onboard`, `ingest`,
  `query`, `lint`).
- [`scripts/`](scripts/) — thin deterministic helpers ([`scripts/README.md`](scripts/README.md)).
- [`wiki.config.json`](wiki.config.json) — per-instance configuration seed.
