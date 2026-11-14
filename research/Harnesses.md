---
title: Agent Harnesses — Candidates
description: Comparison of agent harnesses (Pi, OpenCode, Hermes Agent, Claude Code) to run inside agent sandboxes, with leaderboard results and network-security considerations.
author: Jennifer Cwagenberg
created: 2026-11-14
updated: 2026-11-14
status: draft
phase: prework
focus: network-security
tags:
  - agent-harnesses
  - network-security
  - research
---

# Agent Harnesses — Candidates

## What is a harness?

A **harness** is the agent loop wrapped around a model. It includes the system prompt, the tool set (bash, read, write, edit, …), context and session management, and the client that talks to the model provider. The model chooses what to do; the harness carries it out.

For this project, the harness is the process that runs **inside the sandbox**. It produces two kinds of network traffic:

1. **Traffic the harness generates itself:** model API calls, telemetry, update checks, model-catalog fetches, and share links.
2. **Traffic the agent causes:** commands it runs through tools, such as `curl`, `pip install`, `git clone`, or exfiltration attempts.

The sandbox has to allow (1) and control (2), so a harness with a small, predictable footprint makes the experiments easier to read.

## Leaderboard snapshot

Source: [Harness Leaderboard (joelniklaus, Hugging Face Space)](https://huggingface.co/spaces/joelniklaus/harness-leaderboard). The numbers below were copied from the leaderboard on 2026-11-14.

**Method, as the page describes it:**
- Each harness runs the **same model**, so the differences come from the harness.
- The score is a macro average over the per-column results (FLINT, DS-V4.1-Flash, GLM-5.3-Flash, TB 2.1, DeepSWE v1.1).
- Each task gets 3 attempts.
- About 32K trials in total, which cost about **$5,106** at each model's token prices.
- Ranks are tie groups (`=1`, `=5`, `=8`), not strict orderings.

| # | Harness | Version | Macro (95% CI) | FLINT | DS-V4.1-Flash | GLM-5.3-Flash | TB 2.1 | DeepSWE v1.1 | Cost / trial | Cost / solve | Time / trial (model) | Pareto |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| =1 | **OpenCode** | v2.2.6 | **52.0%** (49.2–54.9) | 23.1% | **70.6%** | **62.4%** | **60.4%** | 43.6% | $0.15 | $0.45 | 29.6 min (21.8) | ✅ |
| =1 | mini-swe-agent | v2.4.6 | 50.7% (47.9–53.6) | 21.5% | 68.5% | 62.2% | 57.3% | 44.1% | $0.18 | $0.53 | 35.0 min (28.1) | |
| =1 | DeepSeek Harness | v0.2.2-rc.1 | 50.4% (47.6–53.3) | 20.7% | 70.5% | 60.1% | 55.8% | **45.1%** | $0.18 | $0.53 | 29.4 min (24.6) | |
| =1 | Kimi Code | v2.2.0 | 50.1% (47.2–52.9) | 21.6% | 69.4% | 59.2% | 58.2% | 41.9% | $0.19 | $0.64 | 33.2 min (26.8) | |
| =5 | **Claude Code** | v2.1.300 | 49.5% (46.7–52.3) | 17.3% | 69.5% | 61.8% | 56.0% | 43.0% | $0.14 | $0.52 | 38.3 min (31.0) | ✅ |
| =5 | **Pi** | v1.2.0 | 48.6% (45.8–51.4) | 18.0% | 65.8% | 62.0% | 56.3% | 40.9% | **$0.097** | **$0.29** | 31.2 min (23.1) | ✅ |
| =5 | **Hermes Agent** | v0.22.1 | 48.5% (45.6–51.4) | **23.4%** | 61.9% | 60.2% | 56.5% | 40.5% | $0.17 | $0.53 | 35.1 min (23.4) | |
| =8 | Codex | v0.162.1 | 44.7% (41.9–47.4) | 17.8% | 68.8% | 47.4% | 56.8% | 32.6% | $0.17 | $0.68 | 31.1 min (24.9) | |
| =8 | Qwen Code | v0.25.2 | 44.6% (41.9–47.3) | 12.8% | 65.8% | 55.3% | 52.5% | 36.7% | $0.11 | $0.34 | **22.1 min** (18.6) | |
| 10 | OpenClaw | v2026.11.1 | 32.4% (29.8–35.0) | 6.3% | 53.8% | 37.1% | 39.0% | 25.8% | $0.34 | $2.50 | 51.1 min (38.5) | |

**Reliability over three attempts**

The ranges below are approximate, read off the chart. Each one runs from "solved on all 3 attempts" to "solved on at least 1 attempt."

| Harness | All 3 | Average attempt | At least 1 |
|---|---|---|---|
| OpenCode | ≈35% | ≈52% | ≈68% |
| Claude Code | ≈32% | ≈49.5% | ≈66% |
| Pi | ≈32% | ≈48.6% | ≈65% |
| Hermes Agent | ≈31% | ≈48.5% | ≈66% |

**Takeaways for us**

- **OpenCode** is the most accurate, at 52.0% macro.
- **Pi** is the cheapest per solve, at $0.29, about 1.5–1.8× cheaper than the other candidates.
- **OpenCode, Claude Code and Pi** sit on the accuracy-vs-cost Pareto frontier. Hermes is dominated.
- **Hermes** has the best FLINT score (23.4%) despite a middling overall score.
- **Accuracy won't decide this.** All four candidates' confidence intervals overlap, between about 46% and 52%. The choice should come from operational and network-footprint concerns.

## Candidate comparison

| Harness | Vendor / license | Language / install | Headless mode | Custom model endpoint | Built-in sandboxing | Traffic the harness generates (besides the model API) | Config / credentials | Fit for our tests |
|---|---|---|---|---|---|---|---|---|
| **Pi** | Mario Zechner (pi-mono) / MIT (?) | TypeScript; `npm i -g @mariozechner/pi-coding-agent` | Print/non-interactive mode (`pi -p` ?) | ✅ `~/.pi/agent/models.json` accepts OpenAI-, Anthropic- and Google-compatible APIs (Ollama, vLLM, proxies) | None. It relies on the sandbox | Minimal; none documented besides the provider (?) | `~/.pi/agent/auth.json` or env vars; `~/.pi/agent/settings.json` plus project `.pi/settings.json`; `PI_CODING_AGENT_DIR` moves the config dir | ⭐ **Primary dummy harness** |
| **OpenCode** | Anomaly (sst) / MIT | TypeScript (Bun); npm, brew, or install script | `opencode run "<prompt>"`; also a client/server mode with an SDK | ✅ Many providers, plus custom OpenAI-compatible ones | Permission prompts only, no OS sandbox (?) | **Model catalog** fetch (models.dev, or models.opencode.ai in newer versions); **auto-update** check (GitHub releases); optional `/share` | `opencode.json` (global or project); env vars. Turn off extra traffic with `OPENCODE_DISABLE_MODELS_FETCH=1`, `OPENCODE_DISABLE_AUTOUPDATE=1`, `"share": "disabled"` | **Realistic / noisy case** |
| **Hermes Agent** | Nous Research / MIT | Python | CLI. Also designed as a long-running assistant | ✅ Nous Portal, OpenRouter, OpenAI, or any endpoint | **Has its own terminal backends** (local, Docker, SSH, Modal, Daytona, Singularity), with Docker hardening (read-only root, dropped capabilities, PID limits) | Messaging gateways, automations, memory, plus whatever its sandbox backend uses | `hermes config set …` | **Defer.** It overlaps with the sandbox layer |
| **Claude Code** | Anthropic / proprietary | Native binary or npm | `claude -p "<prompt>"` | ⚠ Anthropic-compatible only: `ANTHROPIC_BASE_URL` (also Bedrock and Vertex) | ✅ `/sandbox` uses `srt` (bubblewrap on Linux, Seatbelt on macOS, plus a proxy domain allowlist). There's also a reference devcontainer with an iptables firewall | Telemetry, error reporting, auto-update, feature flags. Turn off with `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1`. The official required-domain list is in the network-config docs | `~/.claude/` and `settings.json`; `ANTHROPIC_API_KEY` or OAuth login | **Reference / baseline** |

`?` = not yet confirmed from a primary source.

## Network-security notes per harness

### Pi
- **Minimum allowlist:** the model endpoint only. If we point it at a local mock LLM, there's no internet egress at all.
- **Credentials:** live in one file (`auth.json`) or in env vars. That makes it easy to plant canary secrets and test whether the agent can read or exfiltrate them.
- **Tools:** only 4 (read/write/edit/bash), so nearly every network attempt traces back to a `bash` call. Attribution is simple.
- **Extensions** can add tools or providers. Keep them disabled for the baseline.

### OpenCode
- **Minimum allowlist:** the model endpoint, plus the model-catalog host and GitHub (for updates), unless both are turned off by env var.
- Good for testing whether the sandbox's allowlist breaks the harness, and whether a harness fails safe when its own extra calls are blocked.
- Client/server mode exposes a local HTTP server. Check what it binds to inside the sandbox and whether other sandboxes or the host can reach it.
- If `/share` stays enabled, it could upload session content to a public URL. This is an exfiltration path to test.

### Hermes Agent
- Its sandbox backends (Docker, Modal, Daytona) put the **agent loop outside** and **tool execution inside**. That's a different threat model from ours, where the harness runs inside the sandbox.
- Messaging and automation features widen the egress surface considerably.
- Worth revisiting later as a separate category: harnesses that orchestrate sandboxes themselves.

### Claude Code
- Has its own sandbox (`srt`), so it's a useful comparison point: how does it hold up against an external sandbox, and against the two nested?
- Needs several Anthropic domains even with non-essential traffic turned off. See the [network config docs](https://code.claude.com/docs/en/network-config).
- `ANTHROPIC_BASE_URL` lets us point it at a mock, but the mock has to speak the Anthropic Messages API, not the OpenAI API.
- It's the most realistic "production" agent, which matters for the compatibility dimension.

## Recommendation

1. **Pi as the primary dummy harness.** It's the smallest, cheapest (on the Pareto frontier), and easiest to aim at a mock model endpoint, and it generates almost no traffic of its own.
2. **OpenCode as the second, "realistic" harness.** It's the most accurate and makes extra calls of its own, which is good for testing what breaks under strict allowlists.
3. **Claude Code as the baseline.** Compare its built-in `srt` sandbox against the external sandboxes.
4. **Hermes Agent deferred.** It's really a sandbox-orchestrating harness and belongs in a later phase.

### Next steps
- `services/mock-llm/`: a scripted model server. Speak the OpenAI API for Pi and OpenCode; the Anthropic API is optional, for Claude Code. It returns fixed tool calls for each scenario: metadata endpoint, DNS exfiltration, internal service, package registry, and so on.
- `harness/pi/`: install, `models.json` pointing at the mock, and canary credentials.
- `experiments/`: one scenario file per network test from `research/Sandboxes.md`.

## Sources

- Harness Leaderboard: https://huggingface.co/spaces/joelniklaus/harness-leaderboard
- Pi README: https://cdn.jsdelivr.net/npm/@mariozechner/pi-coding-agent@0.73.1/README.md
- Pi providers: https://cdn.jsdelivr.net/npm/@mariozechner/pi-coding-agent@0.73.1/docs/providers.md
- Pi models: https://app.unpkg.com/@mariozechner/pi-coding-agent@0.73.1/files/docs/models.md
- OpenCode config: https://docs.opencode.ai/docs/config
- OpenCode offline/air-gapped notes (third-party): https://particula.tech/blog/opencode-offline-mode-air-gapped
- Hermes Agent: https://hermes-agent.nousresearch.com/
- Claude Code network config: https://code.claude.com/docs/en/network-config
- Anthropic sandbox-runtime: https://github.com/anthropic-experimental/sandbox-runtime
