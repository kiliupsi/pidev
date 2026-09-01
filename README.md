# PiDevHouse

A multi-agent AI software house built on [Pi](https://github.com/badlogic/pi-mono), a self-built dev harness. The system — internally named **Concentus** (Latin for "ensemble") — is a university master's project (PIDEV) accompanying an academic paper on failure modes and robustness in agentic systems.

The user submits a software request in natural language. A **Product Owner** agent decomposes it into user stories, and a **Developer → Reviewer → Tester** loop implements, reviews, and tests each story in sandboxed workspaces — producing working code, review/test scores, and detailed run telemetry.

## Architecture

```
┌──────────────────────────── TUI (OpenTUI + SolidJS) ────────────────────────────┐
│  composer ─ activity feed ─ header/footer (agent context, spinner, elapsed)     │
└──────────────────────────────────────┬──────────────────────────────────────────┘
                                       │ Message stream (bus + JSONL log)
┌──────────────────────────────────────▼──────────────────────────────────────────┐
│ Runtime                                                                          │
│   workflow.ts    run(): workspace setup, PO run, story loop, summary             │
│   storyLoop.ts   runStory(): developer → reviewer → tester iteration loop        │
└──────────────────────────────────────┬──────────────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼──────────────────────────────────────────┐
│ Modules                                                                          │
│   agents/       Product Owner · Developer · Reviewer · Tester (prompt templates) │
│   models/       Agent base class, llama provider, config, Zod story schema       │
│   repository/   In-memory + JSON-persisted stories, dependency-aware scheduling  │
│   services/     message bus, agent event bridge, summary collector               │
│   tools/        bwrap-sandboxed bash, browser, path scoping, context eviction    │
│                 story tools: createStories, getStory, updateStatus, updateScore  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### How a run works

1. A run directory `runs/<slug>/<timestamp>` is created with a fresh git repo (`src/`, `test/`, `log/`) and a llama-server preflight (verifies all slots have 64k context).
2. The **Product Owner** turns the request into a story plan (`createStories`).
3. For each ready story (all `blockedBy` dependencies tested), `runStory` iterates up to `maxIteration` times:
   - **Developer** implements the story and marks it `implemented`
   - **Reviewer** reviews and records a score (must be ≥ `minScore` and approve)
   - **Tester** tests and records a score (browser tool for UI stories)
   - Agents that fail to record their result are re-prompted before the iteration counts as failed.
4. Outcomes are classified: `completed | incomplete | infrastructure | no_ready | max_iterations | timeout | cancelled | error`, and a `summary.json` with per-agent token/TTFT/throughput metrics is always written.

### Robustness mechanics

- **Story state machine** — `todo → in_progress → implemented → approved → tested` (+`blocked`), enforced via Zod-validated transitions.
- **Sandboxing** — developer bash runs inside [bubblewrap](https://github.com/containers/bubblewrap) (`--unshare-all`, dropped caps, only `src/` + `test/` writable, Linux-only).
- **Path scoping & tool-call budget** — tools are blocked outside the workspace; steering warnings fire at 70%/85% of the budget.
- **Context eviction** — old tool results are deterministically elided (high/low water marks) to keep the KV prefix cache hot; replaces pi compaction for the tester.
- **Self-hosted inference** — Qwen3.8-27B GGUF via llama.cpp `llama-server` (see `scripts/serve-qwen3.8.sh` and `models/qwen3.8-mtp`), exposed over Tailscale with prefix-cache reuse and MTP speculative decoding.

## Layout

```
packages/core/    The Concentus agent system (Bun + TypeScript)
models/           Ollama-style model definition (Qwen3.8 MTP)
scripts/          Remote llama-server host setup
paper/            Typst paper (nested git repo, focus: F-06 failure modes & robustness)
```

`packages/core/src` is organized as `tui/` (terminal UI), `runtime/` (orchestration), and `modules/` (agents, models, repository, services, tools).

## Getting started

Requires Linux (bubblewrap sandbox) and [Nix](https://nixos.org) with flakes:

```bash
direnv allow      # .envrc: "use flake" — provides bun, bubblewrap, chromium, python3
bun install       # from repo root
bun run dev       # start the TUI
```

`LLAMA_SERVER` and `LLAMA_MODEL` must be set (see `packages/core/.env`, gitignored). Runs are written to `packages/core/runs/`.

### Tests

```bash
cd packages/core
bun test
```

### Experiments

Batch experiment harness used to generate the paper's data (task × variant × run trials, each in a detached worker process):

```bash
cd packages/core
bun run experiment [spec.json] [--dry-run]   # default spec: scripts/f06-experiment.json
```

## License

MIT — see [LICENSE](LICENSE).
