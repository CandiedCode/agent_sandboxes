# Researcher: Academic Papers and Analyses Referencing NVIDIA OpenShell

We found **7 research papers and academic preprints** that evaluate, benchmark, or integrate **NVIDIA OpenShell** in the context of AI agent sandboxing, network security, and runtime containment.

---

## 📄 Academic Research Papers (7)

| # | Title | Venue / Source | Year |
|---|-------|----------------|------|
| 1 | [When the Agent Is the Adversary: Architectural Requirements for Agentic AI Containment After the April 2026 Frontier Model Escape](https://arxiv.org/abs/2604.23425) | arXiv:2604.23425 | 2026 |
| 2 | [Before the Tool Call: Deterministic Pre-Action Authorization for Autonomous AI Agents](https://arxiv.org/abs/2603.20953) | arXiv:2603.20953 | 2026 |
| 3 | [Tracekit Box: Running AI Agents in Micro-VMs Where Every Way Out Is Decided and Signed](https://arxiv.org/abs/2610.38458) | arXiv:2610.38458 | 2026 |
| 4 | [Beyond Static Sandboxing: Learned Capability Governance for Autonomous AI Agents](https://arxiv.org/abs/2604.11839) | arXiv:2604.11839 | 2026 |
| 5 | [Agent-Native Applications: A New Application Paradigm](https://arxiv.org/abs/2611.23243) | arXiv:2611.23243 | 2026 |
| 6 | [Towards an Agent Operating System - Lessons from Classical and Cloud OS](https://arxiv.org/abs/2607.25076) | arXiv:2607.25076 | 2026 |
| 7 | [Tether: Verified Stateful Policy Engine for System-Level Monitoring of Agents](https://arxiv.org/abs/2611.25599) | arXiv:2611.25599 | 2026 |

**(1)** Evaluates four categories of agent containment (alignment, environment sandboxes, tool-call interception, audit systems) and critically benchmarks NVIDIA OpenShell, Microsoft AGT, and AEGIS against adversarial escape vectors following real-world frontier model escapes.
**(2)** Examines NVIDIA OpenShell and NemoClaw architectures, highlighting out-of-process policy engines, kernel-level network allowlisting, and filesystem write restrictions as key layers in multi-tier agent safety.
**(3)** Contrasts Micro-VM cryptographic egress signing (Tracekit Box) with runtime domain filtering proxies and declarative network policies implemented in Anthropic's sandbox and NVIDIA OpenShell.
**(4)** Investigates capability over-provisioning in agent runtimes, analyzing OpenShell's container-level isolation and declarative YAML policies alongside adaptive governance systems.
**(5)** Systematizes the shift from prompt-level guardrails to external runtime mediation, characterizing OpenShell's policy-as-code enforcement over local file access, credentials, external networks, and inference routing.
**(6)** Explores Agent-OS architectural primitives (identity, authorization, sandboxing, observability), assessing OpenShell as a dedicated sandbox management layer for agent harnesses.
**(7)** Compares stateless policy allowlists (as found in OpenShell and Cedar) against stateful information flow control (IFC) to prevent multistage staging and data exfiltration.

---

## 🔧 Technical Documentation & Reference Implementations (2)

| # | Title | Source | Year |
|---|-------|--------|------|
| 1 | [OpenShell Architecture Documentation](https://docs.nvidia.com/openshell/about/architecture) | NVIDIA | 2026 |
| 2 | [How to Secure AI Coding Agents in NVIDIA OpenShell with Evo](https://snyk.io/blog/secure-ai-coding-agents-nvidia-openshell/) | Snyk Security | 2026 |

**(1)** Official architectural documentation detailing OpenShell's supervisor-sandbox split, Landlock/seccomp enforcement, credential brokering, and the OpenShell Sandbox Protocol.
**(2)** Technical deep dive into per-binary network egress filtering, secret proxying, and dependency firewall integration within OpenShell container sandboxes.

---

## Key Themes Across Sources

- **Decoupled Architecture (Supervisor vs. Sandbox)**: OpenShell places policy enforcement outside the untrusted agent environment via an external supervisor, preventing prompt-injected or compromised agents from altering their own constraints.
- **Granular Egress & Network Filtering**: Rather than basic host-level network blocking, papers study OpenShell's ability to enforce destination allowlists and per-binary networking rules, preventing tools from loaning approved network access to unauthorized binaries.
- **Stateless vs. Stateful Containment**: Recent preprints contrast OpenShell's declarative stateless YAML allowlists with newer stateful information-flow tracking (IFC) and cryptographic egress logging to mitigate persistent multi-stage escapes.

---
*Research conducted on November 25, 2026.*
