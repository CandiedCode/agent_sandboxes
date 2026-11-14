---
title: Agent Sandbox Security — Literature Review
description: Published papers on AI agent sandbox security, and how much of that work covers network security (egress, exfiltration, DNS, metadata endpoints).
author: Jennifer Cwagenberg
created: 2026-11-14
updated: 2026-11-14
status: draft
phase: prework
focus: network-security
tags:
  - agent-sandboxes
  - network-security
  - literature-review
  - research
---

# Agent Sandbox Security — Literature Review

## Summary

- **Most of this literature is new.** Nearly everything listed here is a 2025–2026 arXiv preprint (mostly cs.CR), and most of it has not been peer reviewed. AgentDojo (NeurIPS 2024) is the main exception.
- **Isolation gets far more attention than the network.** Papers usually ask whether the agent can break out to the host (container escape, kernel attack surface, CVEs) or whether tool calls are authorized. Few treat the network boundary as the thing under test.
- **A survey confirms the gap.** Rashidi (2607.05743) organizes 39 execution-security papers into 17 categories. The *network egress control* category contains **one paper**, [Firewalls to Secure Dynamic LLM Agentic Networks](https://arxiv.org/abs/2502.01822). The survey says this is "either a sign these are genuinely nascent directions or a sign our search has not yet found the rest of a larger literature."
- **The few network results agree with each other:**
  - Prompt-layer defenses do not stop exfiltration.
  - Network controls (domain allowlists, redirect-chain analysis, default-deny proxies) do stop it.
  - Sharded exfiltration (splitting data across many requests) gets past naive data-loss prevention.
- **Some topics have almost no academic coverage:** DNS exfiltration, raw-IP and alternate-port bypass, cloud metadata (`169.254.169.254`), credential injection at the boundary, and comparing how different sandbox products enforce network policy. Practitioner blogs cover some of these; the academic papers mostly do not.

## Network coverage at a glance

✅ = network security is a primary subject · ◐ = covered in part, or as one control among many · ✗ = not covered (from the abstract or full text)

| Paper | arXiv | Date | Focus | Network? |
|---|---|---|---|---|
| Silent Egress | [2602.22450](https://arxiv.org/abs/2602.22450) | Feb 2026 | Implicit prompt injection → outbound leaks | ✅ |
| Permission Denied / Boundary-Bench | [2608.02670](https://arxiv.org/abs/2608.02670) | Aug 2026 | Coding agents under enterprise hardening policies | ✅ (detailed egress design) |
| Caging the Agents | [2603.17419](https://arxiv.org/abs/2603.17419) | Mar 2026 | Zero-trust production architecture (healthcare) | ✅ |
| ceLLMate | [2512.12594](https://arxiv.org/abs/2512.12594) | Dec 2025 | Browser agents sandboxed at the HTTP layer | ✅ (HTTP request policy) |
| Firewalls to Secure Dynamic LLM Agentic Networks | [2502.01822](https://arxiv.org/abs/2502.01822) | Feb 2025 | Input/output "firewalls" for agent-to-agent traffic | ◐ (information flow, not packets) |
| SandboxEscapeBench | [2603.02277](https://arxiv.org/abs/2603.02277) | Mar 2026 (v3 Aug) | Can frontier LLMs escape containers? | ◐ (1–2 network escape paths) |
| SandboxEval | [2504.00018](https://arxiv.org/abs/2504.00018) | Mar 2025 | Test suite for untrusted-code eval harnesses | ◐ ("external communication" tests) |
| A3S-Bench / ASEval | [2605.22321](https://arxiv.org/abs/2605.22321) | May 2026 | Trajectory-level agent security (20 threats) | ◐ (information leakage category) |
| Red-Teaming the Agentic Red-Team | [2606.24496](https://arxiv.org/abs/2606.24496) | Jun 2026 | Attacks on offensive-security agent tools | ◐ (API-key exfiltration, lateral movement) |
| Balkanization of Execution-Security Research | [2607.05743](https://arxiv.org/abs/2607.05743) | Jul 2026 | Survey of 39 papers | ◐ (identifies the egress gap) |
| AI Code Sandboxes: Comparative Study, Part 1 | [2606.08433](https://arxiv.org/abs/2606.08433) | Jun 2026 | Engine-level isolation across 5 products | ✗ (engine axes only; Part 2 unknown) |
| AI Sandboxes: Threat Model, Taxonomy, Measurement | [2606.18532](https://arxiv.org/abs/2606.18532) | Jun 2026 (v2 Nov) | General sandbox measurement framework | ✗ in abstract ("outward harm" only) |
| When the Agent Is the Adversary | [2604.23425](https://arxiv.org/abs/2604.23425) | Apr 2026 | Containment requirements after a frontier escape | ✗ in abstract |
| AgentDojo | [2406.13352](https://arxiv.org/abs/2406.13352) | NeurIPS 2024 | Prompt-injection benchmark | ◐ (exfiltration goals at the tool level, not network level) |

## Papers where the network is the main subject

### Silent Egress (2602.22450)
Lan, Kaul, Jones, Westrum. The attack is adversarial text in auto-generated **URL previews** (titles, metadata, snippets). It makes the agent send outbound requests that leak runtime context, while the reply the user sees looks harmless.
- Exfiltration succeeded with P(egress) = 0.89 over 480 runs, using a small local model (qwen2.5:7b).
- Output-based safety checks missed 95% of the successful attacks.
- **Sharded exfiltration** (spreading data over several requests) cut single-request leakage by 73% and got past simple data-loss prevention.
- Domain allowlisting and redirect-chain analysis worked much better than prompt-layer defenses.
- The authors conclude that network egress should be treated as "a first-class security outcome."

### Permission Denied (2608.02670) — closest to this project
Davidovich, Amar, Rozencwajg, Hiltch. The authors ran 12 coding agents on Terminal-Bench 2.1 under nested policy levels (control → non-root → NIST-derived high), in Daytona sandboxes. They release **Boundary-Bench**, an open-source hardening plugin. The strictest policy restricts egress like this (checked against the full text):
- **Proxy:** a loopback proxy that runs as root and denies by default.
- **Firewall:** a rule scoped to the agent's user that rejects any traffic bypassing the proxy, including from child processes.
- **Always denied:** cloud metadata endpoints and private ranges. Resolved IPs are rechecked so an allowed domain can't resolve into a denied range, which defends against DNS rebinding.
- **Allowlist:** 205 domains adapted from Claude Code's default web allowlist. The list was deliberately not tuned to the tasks.
- **Pre-flight probe suite:** direct, adversarial, and enablement probes. The adversarial probes cover nested shells, raw-IP egress, alternate ports, and child-process inheritance. If the probes fail, the run fails closed.
- **Result:** the strictest policy cut success by up to 18.3 points and raised cost by up to 167%.

This is the most reusable methodology for our experiments (see [Sandboxes.md](Sandboxes.md)).

### Caging the Agents (2603.17419)
Maiti (Commure). A production deployment of 9 agents behind four defense layers:
- gVisor on Kubernetes.
- **Credential-proxy sidecars**, so raw secrets never enter the agent container.
- **Per-agent egress allowlists.**
- A prompt-integrity framework.

Network exfiltration is one of six threat domains. The paper notes practical problems: CDN IP rotation breaks IP-based allowlists, so policy controllers need to be DNS-aware. It reports 90 days of operating data, and the configurations are open source.

### ceLLMate (2512.12594)
Meng, Feng, Shumailov, Fernandes. Sandboxes browser agents at the **HTTP request layer** rather than the UI layer. Every side effect ends up as a request to the backend, so the policy is enforced there, using an "agent sitemap" that maps requests to semantic actions. It is implemented as a Chrome extension and adds 7–15% latency on WASP.

## Papers that cover the network in part

- **SandboxEscapeBench (2603.02277)** (Marchand et al.): 18 escape scenarios on Docker/OCI and Kubernetes. The network-relevant ones are:
  - **route_ln** (CVE-2020-8558, a route_localnet bypass that reaches host localhost services).
  - **RBAC** (reaching the Kubernetes API).
  - **docker.sock**.

  The sandbox has open connectivity to the host and the internet, and the paper notes shortcuts through the Docker bridge gateway. It has **no** egress-filtering, host-network, or metadata scenarios.
- **SandboxEval (2504.00018)**: about 51 hand-written test cases, including "external communication," run against the Dyff evaluation harness. It is an early precedent for probing a sandbox's network properties.
- **A3S-Bench / ASEval (2605.22321)** (Ant Group): 2,254 trajectories, with OpenClaw running on 10 model backbones. The most exploitable categories were **sandbox escape and information leakage**. Attacks triggered 28.3% of the time with naive injection and 52.6% with evasion techniques.
- **Red-Teaming the Agentic Red-Team (2606.24496)**: offensive-security agent tools let an attacker exfiltrate API keys, gain persistence, and compromise the operator's machine **even inside containers**. The paper lays out a kill chain that includes lateral movement.
- **Balkanization survey (2607.05743)**: its single egress paper is the Firewalls paper (2502.01822). Its gap analysis notes that isolation architectures are rarely benchmarked against each other. It also lists Claude Code CVE-2026-21852, a data-exfiltration flaw that ran before trust confirmation.

## Gaps this project could fill

1. **Comparing network enforcement across sandbox products** (allowlist semantics, enforcement point inside or outside the guest, bypass resistance). No published paper does this. The Part 1 comparative study explicitly stops at engine-level properties.
2. **Covert channels:** DNS tunneling, raw IP, alternate ports, IPv6, and abuse of domains that are already allowed (e.g., GitHub gists, package registries). Permission Denied probes a few of these; nobody evaluates them adversarially at scale.
3. **Cloud metadata and internal-network SSRF** from inside agent sandboxes.
4. **Credential injection at the boundary**, measured as a defense. Today it is only described in architecture papers.
5. **Agent-driven bypass:** a capable agent actively trying to get around the network policy. SandboxEscapeBench does this for container escape, but no one has done it for egress.

## Caveats

- Every source is a preprint unless marked otherwise. Dates come from arXiv listings.
- I checked the full text for SandboxEscapeBench, Permission Denied, and the Balkanization survey. For the other papers I read only the abstract or abstract page, so "✗" means not visible in the abstract; the full text could still cover the network.
- Things I could not verify, and so left out of the table:
  - A search-engine claim that AgentBay (2512.04367) blocked C2 egress. Its abstract only discusses streaming bandwidth.
  - Details of *Provably Secure Agent Guardrail* (2605.29251), *Lingering Authority* (2606.22504), and *SoK: Attack Surface of Agentic AI* (2603.22928).
