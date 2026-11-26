# Research Report: Network Security and Firewalls for AI Agent Sandboxes

We found **8 key research papers and technical analyses** focusing on firewalls, network isolation, and sandbox security for autonomous LLM agents, organized by category below.

---

## 📄 Academic Research Papers

| # | Title | Source | Year |
|---|-------|--------|------|
| 1 | [Firewalls to Secure Dynamic LLM Agentic Networks](https://arxiv.org/abs/2502.01822) | arXiv:2502.01822 | 2025 |
| 2 | [Rethinking Agent Security as a Networking Problem](https://arxiv.org/abs/2608.12172) | arXiv:2608.12172 | 2026 |
| 3 | [Token-Flow Firewall: Semantic Runtime Auditing for Persistent AI Agents](https://arxiv.org/abs/2607.08395) | arXiv:2607.08395 | 2026 |
| 4 | [LLM Agents Should Employ Security Principles](https://arxiv.org/abs/2505.24019) | arXiv:2505.24019 | 2025 |
| 5 | [The Balkanization of Execution-Security Research for AI Coding Agents: Isolation, Access Control, and TOCTOU Vulnerabilities](https://arxiv.org/abs/2607.05743) | arXiv:2607.05743 | 2026 |
| 6 | [Model-as-a-Trojan: Self-Activating Covert DNS Exfiltration Baked Into the Weights of an LLM Agent](https://arxiv.org/abs/2610.22578) | arXiv:2610.22578 | 2026 |
| 7 | [Securing AI Agent Execution](https://arxiv.org/abs/2510.21236) | arXiv:2510.21236 | 2025 |
| 8 | [ce llm ate: Sandboxing Browser AI Agents](https://arxiv.org/abs/2512.12594) | arXiv:2512.12594 | 2025 |

**(1)** Proposes a dual-firewall architecture (Language Converter Firewall for inputs and Data Abstraction Firewall for outputs) operating in an isolated trusted environment to project agent-to-agent communication onto strict contextual integrity constraints.  
**(2)** Advocates moving security enforcement out-of-reach of nondeterministic agent prompts to the network layer via sidecar proxies and a centralized SDN-like control plane combining deterministic rules and semantic policy filtering.  
**(3)** Introduces TokenWall, a runtime semantic firewall that inspects and intercepts hazardous token flows (memory updates, tool arguments, retrieved files, egress calls) before reaching privileged sinks.  
**(4)** Introduces the AgentSandbox framework with an explicit I/O Firewall mediating interactions between ephemeral task agents and external services based on least-privilege principles.  
**(5)** Systematizes sandbox isolation, network egress control, and execution-security literature across 39 papers, identifying systemic blind spots in contemporary agent harnesses.  
**(6)** Demonstrates how trojaned model weights can bypass traditional egress-locked sandboxes by covertly tunneling exfiltrated data through DNS queries, outlining defenses like DNS allowlisting and resolver-side firewalls.  
**(7)** Outlines the AgentBox architecture enforcing container sandboxing via dynamic iptables egress rules, environment whitelisting, and scoped tool manifests.  
**(8)** Evaluates host firewall, network proxy, and browser-level interception mechanisms to sandbox autonomous web-browsing agents without disrupting legitimate traffic.

---

## Key Themes Across Sources

- **Moving Firewalls from Prompts to Network Boundaries**: A recurring consensus across recent papers (such as Tran et al. and Abdelnabi et al.) is that in-prompt guardrails are fundamentally inadequate against prompt injection and memory corruption. Security enforcement must be decoupled into independent sidecars, network proxies, and kernel/container firewalls.
- **Semantic & Token-Level Firewalls**: Traditional 5-tuple packet filters cannot understand agent intent or encoded exfiltration payloads. Frameworks like TokenWall and Language Converter Firewalls enforce contextual integrity directly over semantic token flows and API parameters.
- **Covert Egress & Sandbox Escape Vectors**: Recent research highlights subtle egress channels—such as covert DNS exfiltration and TOCTOU race conditions—demonstrating that basic containerization without strict outbound resolver filtering leaves viable attack paths.
- **Ephemeral Sandbox Architectures**: Several works recommend deploying ephemeral, task-specific worker agents mediated by dedicated I/O firewalls, preventing persistent privilege escalation and data spillage.

---

*Research conducted on November 23, 2026.*