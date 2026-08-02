# Day 3 — Verification, Protocol Comparison, and the Rate-Limit Root Cause

## Goal
Confirm the SSH block rule's before/after behavior cleanly, then resolve why pf's rate-limiting feature wasn't working.

## Port 80 / port 22 verification
- `nc -zv 192.168.1.100 80` => "Connection refused" (pass rule working correctly)
- `nc -zv 192.168.1.100 22` => "Connection timed out" (block rule working correctly)
- Firewall log showed the full contrast in one tight window: DNS lookup noise (irrelevant — `nc`'s reverse-DNS attempt, blocked by default-deny since pfSense doesn't answer DNS on OPT1) => port 80 passed => port 22 blocked repeatedly with visible retransmission backoff.

## Protocol comparison: Telnet vs SSH vs RDP

| | Telnet (23) | SSH (22) | RDP (3389) |
|---|---|---|---|
| Access type | Text shell | Text shell | Full GUI desktop |
| Encryption | None (plaintext) | Yes, by default | Yes, but historically vulnerability-prone |
| Status today | Deprecated/legacy only | Standard; still restrict exposure | Common but high-risk if exposed directly |

Connected to real-world practice: companies commonly block SSH/RDP exposure by network zone (public internet vs. internal/VPN) — same underlying principle as this lab's OPT1 block rule.

## Rate-limiting root cause investigation

**Setup:** OPT1 pass rule configured with `max-src-conn-rate 3/1` (later tested at various thresholds), tied to the `<virusprot>` overload table.

**Problem:** Ran SYN scans at various thresholds (3/1, 10/1) — 100% of traffic passed every time, `<virusprot>` table stayed at 0 entries.

**Diagnostic steps:**
1. Verified via `pfctl -sr` that the rule was correctly loaded into the kernel ruleset (config was not the issue).
2. Verified via `pfctl -vvsr` rule counters that `<virusprot:0>` had zero entries ever added — the rate condition genuinely never triggered.
3. Ran `sudo hping3 -S -p 80 --flood 192.168.1.100` — generated **163,332 packets** in ~10 seconds, a far larger and more sustained burst than any nmap scan.
4. Checked `pfctl -s Sources` immediately after — showed 30,727 states created, but rate had already decayed to `0.0/1s` (rate is a live, continuously-decaying measurement; must be checked *during* the burst, not after it ends).
5. Checked `pfctl -vvsr` on the actual pass rule (`ridentifier 1785148707`) — **58,988 packets, 44,855 state creations, 100% passed, zero diverted to the overload table.**

**Root cause (confirmed via web search, Netgate community forum):** `max-src-conn-rate` only counts **fully established connections** (completed TCP 3-way handshake). Both nmap's SYN scan (`-sS`, sends RST instead of completing) and hping3's flood (`--flood`, never processes replies) are specifically designed to avoid completing the handshake — so this pf feature structurally cannot detect either technique, regardless of tuning. Confirmed by a pfSense community admin's own comment in the referenced thread: "enjoy your time with Snort or Suricata."

## Comparison to prior tools (UFW, Cloudflare WAF)

- **UFW** — host-based, same core stateful-filtering model as pf; similar rate-limiting caveats apply.
- **Cloudflare WAF** — operates at Layer 7 (inspects actual HTTP request content), fundamentally different from pf/UFW's Layer 3/4 packet-header filtering. Has huge aggregate visibility across many sites that a standalone firewall doesn't have.
- Takeaway: stateful firewalls (pf, UFW) are good at "who can talk to whom" — not "is this traffic behaviorally malicious." That's the job of a dedicated IDS/IPS or WAF layer.

## Key lessons

1. **`max-src-conn-rate` is not a scan/flood detector** — it's designed for limiting abuse of *established* connections (e.g., excessive legitimate SMTP connections), not catching SYN scans or floods.
2. **`pfctl -s Sources` rate values are live/decaying** — must be observed during the traffic burst, not after.
3. **A negative result, properly diagnosed with evidence, is a valid and valuable finding** — not a failed lab.
