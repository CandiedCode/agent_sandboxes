---
title: Agent Sandboxes — Landscape
description: Survey of AI agent sandboxes (self-hosted vs. managed), with an initial focus on network security controls.
author: Jennifer Cwagenberg
created: 2026-10-14
updated: 2026-11-14
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
| **OpenSandbox** | Sandbox platform (server + execd + ingress/egress components) on Docker or K8s. Secure runtimes (gVisor/Kata/Firecracker) are optional (?) | Self-hosted | Per-sandbox egress component (DNS + nftables). The credential vault/proxy requires a default-deny policy. ⚠ Declarative domain allow/deny was on the roadmap; check whether it has shipped | Apache-2.0 / Alibaba | [github](https://github.com/alibaba/OpenSandbox) | Multi-language SDKs. Alibaba Cloud has a separate managed "Agent Sandbox" product |
| **Anthropic sandbox-runtime** (`srt`) | bubblewrap (Linux) / Seatbelt (macOS) + HTTP and SOCKS proxies | Self-hosted (local) | Domain allowlist and denylist enforced by proxies outside the sandbox. Network namespace blocks direct egress | Apache-2.0 (?) / Anthropic | [github](https://github.com/anthropic-experimental/sandbox-runtime) | Powers Claude Code's `/sandbox`. Lightweight, so a good baseline for comparison |
| **OpenAI Codex CLI sandbox** | Seatbelt (macOS) / Landlock + seccomp (Linux) | Self-hosted (local) | Network off by default in `workspace-write`. Toggled by config; mostly all-or-nothing (?) | Apache-2.0 / OpenAI | [github](https://github.com/openai/codex) | Built into the agent rather than a standalone product |
| **E2B** | Firecracker microVM per sandbox | Both: E2B Cloud, BYOC, or self-hosted OSS infra | Per-sandbox allow/deny egress rules, updatable on a running sandbox (`PUT /sandboxes/{id}/network`). Bring-your-own proxy (cloud/BYOC) | Apache-2.0 (infra) / E2B | [docs](https://e2b.dev/docs) | Egress IPs rotate, so allowlisting at the destination needs your own proxy |
| **Daytona** | Containers (Docker; gVisor ?) | Both (?): managed; self-host status unclear | Domain firewall with per-sandbox domain and IP allowlists, deny-by-default (per Daytona docs) | ⚠ License is disputed (AGPL earlier; reports of a move to closed source in 2026) / Daytona | [daytona.io](https://www.daytona.io) | Sources conflict; check before shortlisting |
| **Firecracker** / **Kata Containers** | MicroVM VMM / VM-backed OCI runtime | Self-hosted | None built in; they rely on tap/CNI and host firewalling | Apache-2.0 / AWS; OpenInfra | [firecracker](https://firecracker-microvm.github.io) · [kata](https://katacontainers.io) | Building blocks underneath E2B, Vercel (?) and K8s Agent Sandbox. Need KVM (bare metal or nested virt) |

`?` = not confirmed from a primary source yet; check before relying on it.

### Deployment summary

- **Self-hosted only:** Shuru, microsandbox, Docker Sandboxes, NVIDIA OpenShell, Bubblewrap, gVisor, OpenSandbox, Anthropic `srt`, Codex sandbox, Firecracker, Kata
- **Managed only:** Modal, Vercel Sandbox, Cloudflare Sandbox SDK
- **Both:** Dev Containers (Codespaces), Kubernetes Agent Sandbox (GKE), E2B, Daytona (?)

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
- OpenSandbox: https://github.com/alibaba/OpenSandbox
- E2B: https://e2b.dev/docs · https://docs.e2b.dev/network/byop
- Daytona: https://www.daytona.io/dotfiles/domain-firewall
