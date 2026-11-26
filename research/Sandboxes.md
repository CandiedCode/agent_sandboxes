---
title: Agent Sandboxes — Landscape
description: Survey of AI agent sandboxes (self-hosted vs. managed), with an initial focus on network security controls.
author: Jennifer Cwagenberg
created: 2026-10-14
updated: 2026-11-19
status: draft
phase: prework
focus: network-security
tags:
  - agent-sandboxes
  - network-security
  - isolation
  - research
---

# Agent Sandboxes — Landscape

## What is an agent sandbox?

An **agent sandbox** is an execution boundary that lets an AI agent (Claude Code, Codex, a custom tool-calling loop, etc.) run shell commands, install packages, execute generated code and use tools **without inheriting the full authority of the host, the network, or the user's credentials**. The agent gets a disposable environment that looks like a normal machine; the sandbox decides what it can read, write, execute, and reach.

Sandboxes differ mainly in *where the boundary sits*. From weakest to strongest kernel isolation:

| Tier | Mechanism | Examples | Shares host kernel? |
|---|---|---|---|
| OS primitives | namespaces, seccomp, Landlock (Linux); Seatbelt (macOS) | bubblewrap, Anthropic `srt`, Codex sandbox | Yes |
| Containers | namespaces + cgroups + image | Dev Containers, plain Docker | Yes |
| User-space kernel | syscalls served by a userspace kernel | gVisor (`runsc`) | Partially (narrow syscall surface) |
| MicroVM | hardware virtualization, guest kernel | Firecracker, libkrun, Kata, Apple Virtualization.framework | No |

**Network is often the weak link.** A sandbox can isolate the filesystem and processes well and still allow unrestricted outbound traffic. That leaves exfiltration, lateral movement to internal services, and cloud metadata theft (`169.254.169.254`) wide open. The usual network controls are: no network at all, CIDR/IP allowlists, domain allowlists enforced via a proxy, SNI inspection or DNS filtering, and **credential injection at the boundary** so real secrets never enter the sandbox. Where these controls are enforced matters: inside the guest, which the agent may be able to tamper with, or on the host, which it can't reach.

## Landscape table

Deployment key: **Self-hosted** = you run it on your own machines/cluster · **Managed** = vendor-hosted API/SaaS · **Both** = OSS/self-host plus a hosted offering.

| Name | Type / isolation tech | Deployment | Network controls (headline) | License / vendor | Link | Notes |
|---|---|---|---|---|---|---|
| **Shuru** | MicroVM CLI. Apple Virtualization.framework on macOS, KVM on Linux (experimental) | Self-hosted (local) | Network off unless `--allow-net`. Host allowlist via `--allow-host` / `network.allow`; an empty list allows all hosts. Host-side proxy substitutes secrets on HTTPS requests to bound hosts | Apache-2.0 / superhq-ai | [github](https://github.com/superhq-ai/shuru) | Ephemeral rootfs. Read-only mounts by default, with an overlay layer. Mainly for macOS on Apple Silicon. SuperHQ also has a separate cloud product |
| **microsandbox** | MicroVM library/runtime. Credits libkrun; hypervisor not confirmed | Self-hosted (local) | `allowed_hosts` / `allowed_ports` allowlist. Host-scoped secrets that "never enter the VM" | Apache-2.0 / superradcompany | [github](https://github.com/superradcompany/microsandbox) | Beta. Runs on macOS (Apple Silicon), Linux (KVM) and Windows (WHP). No daemon: the SDK boots the VM as a child process |
| **Docker Sandboxes** (`docker sandbox` / `sbx`) | MicroVM per sandbox, via Docker Desktop | Self-hosted (local) | All HTTP/S goes through a host-side proxy. Policy can be allow-all-except or deny-all-except. CIDR blocks. UDP/ICMP are restricted. Changes apply live | Proprietary (Docker Desktop) / Docker | [docs](https://docs.docker.com/ai/sandboxes/network-policies) | ⚠ Default rules include broad wildcards (e.g. `*.googleapis.com`). The "allowlist" label is used inconsistently. CLI syntax is in flux (`docker sandbox` vs `sbx`) |
| **NVIDIA OpenShell** | Policy-driven runtime. Landlock + seccomp + network namespace + egress proxy | Self-hosted | **Default-deny**, including model API traffic. Rules are per-binary (host:port bound to the executable). `network_policies` can be hot-reloaded with `openshell policy set` | OSS (license ?) / NVIDIA | [docs](https://docs.nvidia.com/openshell/latest/about/overview.html) | Filesystem, Landlock and process policy are fixed at creation; only network policy is dynamic. NemoClaw builds on it |
| **Dev Containers** | Container (OCI) defined by `devcontainer.json` | Both: local Docker/Podman, or managed via GitHub Codespaces | Nothing built in. Typically an iptables/ipset firewall in a post-create script (Claude Code's reference devcontainer works this way) | Spec: CC-BY / MIT tooling / Microsoft | [containers.dev](https://containers.dev) | The firewall runs *inside* the container, so an agent with root/NET_ADMIN could undo it. Shares the host kernel |
| **Bubblewrap** (`bwrap`) | Linux namespaces + seccomp (unprivileged) | Self-hosted | Only `--unshare-net`, which is all or nothing. Selective egress needs an external proxy | LGPL-2.0+ / containers project | [github](https://github.com/containers/bubblewrap) | A building block: used by Flatpak, Anthropic `srt`, and the Codex Linux sandbox (?) |
| **gVisor** (`runsc`) | User-space kernel (Sentry) as an OCI runtime | Self-hosted (and runs underneath Modal, GKE Sandbox, and others) | Network modes `sandbox` (netstack) / `host` / `none`. No L7 allowlist on its own; pair it with NetworkPolicy or a proxy | Apache-2.0 / Google | [gvisor.dev](https://gvisor.dev) | Building block. The netstack itself is an attack surface worth testing |
| **Modal Sandboxes** | gVisor (default) or KVM VMs | Managed | `block_network`, `outbound_cidr_allowlist` (any protocol), `outbound_domain_allowlist` (beta, port 443 only, SNI-based). Runtime policy updates are alpha | Proprietary / Modal | [docs](https://modal.com/docs/guide/sandbox-networking) | ⚠ DNS is unrestricted unless the network is fully blocked. Domain fronting and ECH can bypass the domain allowlist (per Modal's own docs). No default access to workspace resources |
| **Vercel Sandbox** | MicroVM (Firecracker ?) | Managed | `deny-all`, or domain and CIDR rules that deny unmatched traffic. Host-side firewall with transparent TCP and DNS redirect, SNI matching, and DNS filtered by the same policy. **Credential injection** at the boundary. Live `sandbox.update()` | Proprietary / Vercel | [blog](https://vercel.com/blog/a-sandbox-without-a-network-boundary-is-only-half-a-sandbox) | Can forward selected HTTPS requests to your own proxy for auditing |
| **Cloudflare Sandbox SDK** | Cloudflare Containers (VM-backed ?), driven from Workers / Durable Objects | Managed | "Outbound Workers" act as programmable egress proxies. `allowedHosts` enables deny-by-default, with glob patterns. TLS interception and credential injection. Per-instance policy changes at runtime (`@cloudflare/sandbox` ≥ 0.8.9) | Proprietary platform; SDK OSS / Cloudflare | [docs](https://developers.cloudflare.com/sandbox/guides/outbound-traffic/) | Egress logic is written as code (a Worker), which makes it very flexible to author |
| **Kubernetes Agent Sandbox** | K8s CRDs + controller (`Sandbox`, `SandboxTemplate`, warm pools). Isolation via RuntimeClass (gVisor or Kata) | Both: OSS on any cluster; managed on GKE | Standard K8s **NetworkPolicy**, one per SandboxTemplate. GKE's managed offering enforces gVisor and default-deny | Apache-2.0 / kubernetes-sigs (SIG Apps) | [github](https://github.com/kubernetes-sigs/agent-sandbox) | NetworkPolicy is L3/L4 only, so a domain allowlist needs Cilium/FQDN policies or a proxy. Kata needs bare metal on EKS |
| **Agent Substrate** | K8s runtime that schedules "actors" onto warm worker pools, with snapshot/resume. Each actor runs in gVisor (default) or a Cloud Hypervisor microVM (Kata guest). Plain containers are not supported | Self-hosted (K8s; local kind for dev, GKE install path) | **Default-deny** ingress, egress and actor-to-actor traffic. Egress goes through a shared gateway outside the guest (`atenet-egress`) with host/wildcard rules (`http`, `https`, `tls_passthrough`). DNS answers only allowed names (NXDOMAIN otherwise). CONNECT, WebSocket, other TCP and UDP are blocked. HTTPS inspection uses TLS termination. **Credential injection** swaps placeholder headers for secrets (K8s Secrets, GCP Secret Manager, or a gRPC plugin). DNS policy changes apply live (~5s) | Apache-2.0 / agent-substrate (CNCF-adjacent, Google-linked ?) | [github](https://github.com/agent-substrate/substrate) | Claims 1.0 but calls itself "VERY young". No IP/CIDR allowlist documented. Worker pods run as root with SYS_ADMIN + NET_ADMIN. Egress gateway is shared and multi-tenant; per-actor proxies are not supported yet. Google's AX (below) is built on it |
| **AX** (Agent Executor) | Orchestration control plane (`ax` CLI, `ax-server`, PostgreSQL state) that runs agent `Task`s on Agent Substrate. Isolation comes from Substrate; AX does not name a runtime itself | Self-hosted (K8s 1.36+ with Substrate installed; `ax local start` creates a local kind cluster) | **Default-deny**: a task with no `Gateway` has no external network access. `Gateway.spec.egress` holds Substrate `EgressRule`s (`http`, `https`, `tlsPassthrough`) with `hostnames` (a `*` wildcard covers only the leftmost label) and `ports`. Bare `*`, IP addresses and `localhost` are rejected at apply. `https` rules are TLS-intercepted; the task runner points the usual CA variables at the gateway's trust bundle. **Credential injection** via `effects.replaceHeaders` + `credentialUri: ate-secret://…`; the bundled K8s Secrets provider denies by default until an admin grants namespaces | Apache-2.0 / Google | [github](https://github.com/google/ax) | "Heavy development"; API is `ax.io/v1alpha1` and you build your own images for now. ⚠ `ax-server` API has no authentication or authorization. Atespaces are not a tenant boundary. `spec.debug: true` gives arbitrary exec inside the sandbox to anyone who can create a Task. Rules are copied at task creation, so editing a Gateway does not affect running tasks. Rules may name in-cluster services (`*.svc.cluster.local`), so test internal reach. DNS and cloud-metadata handling are not documented in AX (inherited from Substrate ?) |
| **OpenSandbox** | Sandbox platform (server + execd + ingress/egress components) on Docker or K8s. Secure runtimes (gVisor/Kata/Firecracker) are optional (?) | Self-hosted | Per-sandbox egress component (DNS + nftables). The credential vault/proxy requires a default-deny policy. ⚠ Declarative domain allow/deny was on the roadmap; check whether it has shipped | Apache-2.0 / Alibaba | [github](https://github.com/alibaba/OpenSandbox) | Multi-language SDKs. Alibaba Cloud has a separate managed "Agent Sandbox" product |
| **Anthropic sandbox-runtime** (`srt`) | bubblewrap (Linux) / Seatbelt (macOS) + HTTP and SOCKS proxies | Self-hosted (local) | Domain allowlist and denylist enforced by proxies outside the sandbox. Network namespace blocks direct egress | Apache-2.0 (?) / Anthropic | [github](https://github.com/anthropic-experimental/sandbox-runtime) | Powers Claude Code's `/sandbox`. Lightweight, so a good baseline for comparison |
| **OpenAI Codex CLI sandbox** | Seatbelt (macOS) / Landlock + seccomp (Linux) | Self-hosted (local) | Network off by default in `workspace-write`. Toggled by config; mostly all-or-nothing (?) | Apache-2.0 / OpenAI | [github](https://github.com/openai/codex) | Built into the agent rather than a standalone product |
| **E2B** | Firecracker microVM per sandbox | Both: E2B Cloud, BYOC, or self-hosted OSS infra | Per-sandbox allow/deny egress rules, updatable on a running sandbox (`PUT /sandboxes/{id}/network`). Bring-your-own proxy (cloud/BYOC) | Apache-2.0 (infra) / E2B | [docs](https://e2b.dev/docs) | Egress IPs rotate, so allowlisting at the destination needs your own proxy |
| **Daytona** | Containers (Docker; gVisor ?) | Both (?): managed; self-host status unclear | Domain firewall with per-sandbox domain and IP allowlists, deny-by-default (per Daytona docs) | ⚠ License is disputed (AGPL earlier; reports of a move to closed source in 2026) / Daytona | [daytona.io](https://www.daytona.io) | Sources conflict; check before shortlisting |
| **Firecracker** / **Kata Containers** | MicroVM VMM / VM-backed OCI runtime | Self-hosted | None built in; they rely on tap/CNI and host firewalling | Apache-2.0 / AWS; OpenInfra | [firecracker](https://firecracker-microvm.github.io) · [kata](https://katacontainers.io) | Building blocks underneath E2B, Vercel (?) and K8s Agent Sandbox. Need KVM (bare metal or nested virt) |

`?` = not confirmed from a primary source yet; check before relying on it.

### Deployment summary

- **Self-hosted only:** Shuru, microsandbox, Docker Sandboxes, NVIDIA OpenShell, Bubblewrap, gVisor, Agent Substrate, AX, OpenSandbox, Anthropic `srt`, Codex sandbox, Firecracker, Kata
- **Managed only:** Modal, Vercel Sandbox, Cloudflare Sandbox SDK
- **Both:** Dev Containers (Codespaces), Kubernetes Agent Sandbox (GKE), E2B, Daytona (?)

## Complementary egress and MCP layers (not sandboxes)

These don't isolate processes on their own. They sit at the network or tool boundary and are meant to be paired with a sandbox from the table above.

| Name | What it is | Deployment | Network controls (headline) | License / vendor | Link | Notes |
|---|---|---|---|---|---|---|
| **Pipelock** | Go egress proxy and MCP wrapper ("AI agent firewall"). One port (default `127.0.0.1:8888`) serves fetch (`/fetch?url=`), a forward proxy (`HTTPS_PROXY` / CONNECT), WebSocket (`/ws?url=`) and MCP (`pipelock mcp proxy`). Optional containment: `pipelock sandbox` (Landlock + netns + seccomp on Linux; `sandbox-exec` on macOS) and `pipelock contain` (Linux; separate operator/proxy/agent UIDs, agent in a private netns) | Self-hosted (binary/Homebrew, Docker, Helm chart, generated docker-compose) | Presets: `strict` (allowlist-only), `balanced` (default; blocklists + SSRF floors), `audit`, `claude-code`, `hostile-model`, and others. DLP on outbound requests (74 patterns, checksum validation, custom `dlp.secrets_file`), with hostname checks before DNS resolution. Prompt-injection scanning on responses (34 patterns; `block` / `strip` / `warn` / `ask`). Optional TLS interception (`tls_interception.enabled`, `passthrough_domains`). MCP tool policy, tool-chain detection, and hashing of tool descriptions to catch rug pulls. Ed25519-signed action receipts. **No credential injection**: it detects, redacts (`<pl:…>` placeholders) and blocks | Apache-2.0 core; `enterprise/` under Elastic License 2.0 / pipelab-org | [github](https://github.com/pipelab-org/pipelock) | Self-described as "a network proxy, not a sandbox": processes the agent spawns can bypass it unless contained. Without TLS interception it sees only the hostname of CONNECT traffic. Not a DNS filter. Detection is "syntactic, not semantic". The `X-Pipelock-Agent` identity header is spoofable. Prebuilt binaries and images include key-gated paid code; source builds are Community-only. Claude Code: `pipelock claude setup` writes `PreToolUse` hooks into `settings.json`; `pipelock exec -- claude` only sets env vars and is not enforcement. Benchmarks are vendor-run. Repo moved from `luckyPipewrench/pipelock` |

## Evaluation dimensions

| Evaluation dimension | What to measure |
|---|---|
| Isolation | Host filesystem access, process escape attempts, cross-sandbox access |
| **Network security** ← _phase 1 focus_ | Internet allowlists, DNS restrictions, internal service access, metadata endpoint blocking |
| Credential security | Whether an agent can read, reuse, or exfiltrate secrets |
| Performance | Cold start, CPU overhead, memory usage, I/O throughput |
| Compatibility | Running Claude Code, Codex, package managers, compilers, tests, Git |
| Resource governance | CPU, RAM, disk, process count, timeout enforcement |
| Auditability | Denied operations, network attempts, policy changes |
| Developer experience | Setup complexity, policy authoring, debugging |
| Scalability | Concurrent sandboxes, creation throughput, cleanup reliability |

### Network security test ideas (draft)

- **Egress allowlist:** check that disallowed domains and IPs fail. Try raw IP, IPv6, non-443 ports, and UDP/QUIC.
- **Bypasses:** domain fronting, SNI/Host mismatch, ECH, `CONNECT` tunneling through an allowed proxy, wildcard abuse (e.g. writable buckets under `*.googleapis.com`).
- **DNS:** resolution of arbitrary names, DNS-tunnel exfiltration, DoH to an allowed resolver.
- **Internal reach:** RFC1918 ranges, link-local, the host gateway, the Docker socket, K8s API and service DNS, other sandboxes.
- **Metadata:** `169.254.169.254` / `fd00:ec2::254` / `metadata.google.internal`.
- **Tamper resistance:** can the agent change the policy itself? Is enforcement inside or outside the guest?
- **Credential injection:** can injected secrets be echoed back via an allowed endpoint the agent controls?
- **Audit:** are denied connections logged, with destination and process attribution?

## Next steps: shortlisting

Pick about 4–5 that cover different tiers and enforcement models:

1. OS primitives: **Anthropic `srt`** or **NVIDIA OpenShell** (per-binary policy)
2. Container: **Dev Containers + firewall script** (the common baseline)
3. User-space kernel / orchestrated: **Kubernetes Agent Sandbox (gVisor)**
4. Local microVM: **Docker Sandboxes**, **Shuru**, or **microsandbox**
5. Managed: **Vercel Sandbox** or **Modal** (the two have contrasting DNS/SNI enforcement)

## Sources

- Shuru: https://github.com/superhq-ai/shuru
- microsandbox: https://github.com/superradcompany/microsandbox
- Docker Sandboxes: https://docs.docker.com/ai/sandboxes/network-policies · https://docs.docker.com/ai/sandboxes/security/
- NVIDIA OpenShell: https://docs.nvidia.com/openshell/latest/about/overview.html · https://docs.nvidia.com/openshell/reference/policy-schema
- Modal: https://modal.com/docs/guide/sandbox-networking
- Vercel: https://vercel.com/blog/a-sandbox-without-a-network-boundary-is-only-half-a-sandbox
- Cloudflare: https://developers.cloudflare.com/sandbox/guides/outbound-traffic/ · https://developers.cloudflare.com/changelog/post/2026-04-13-sandbox-outbound-workers-tls-auth/
- K8s Agent Sandbox: https://github.com/kubernetes-sigs/agent-sandbox · https://opensource.googleblog.com/2025/11/unleashing-autonomous-ai-agents-why-kubernetes-needs-a-new-standard-for-agent-execution.html
- Agent Substrate: https://github.com/agent-substrate/substrate · https://github.com/agent-substrate/substrate/blob/main/docs/guides/egress-traffic.md · https://github.com/agent-substrate/substrate/blob/main/docs/guides/egress-credential-injection.md · https://github.com/agent-substrate/substrate/blob/main/docs/security/security.md
- AX: https://github.com/google/ax · https://github.com/google/ax/blob/main/docs/manifests.md · https://github.com/google/ax/blob/main/docs/sandbox.md · https://github.com/google/ax/blob/main/SECURITY.md · https://github.com/google/ax/blob/main/DESIGN.md · https://github.com/google/ax/blob/main/docs/roadmap.md
- OpenSandbox: https://github.com/alibaba/OpenSandbox
- E2B: https://e2b.dev/docs · https://docs.e2b.dev/network/byop
- Daytona: https://www.daytona.io/dotfiles/domain-firewall
- Pipelock: https://github.com/pipelab-org/pipelock · https://github.com/pipelab-org/pipelock/blob/main/docs/bypass-resistance.md · https://github.com/pipelab-org/pipelock/blob/main/docs/guides/claude-code.md
