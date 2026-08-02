# pfSense Home Lab - Attack, Detect, Defend

A hands-on cybersecurity home lab built as a mentor-assigned co-op training project. The goal: build a small segmented network, generate realistic attacker traffic, and test how well different firewall/IDS detection mechanisms actually catch that traffic — including where they fail, and why.

This lab is fully local, built with VirtualBox on a personal laptop. It is separate from an unrelated VPS-based project also referenced in some notes.

## Architecture

Three VMs connected across isolated internal network segments:

| VM | Role | Network | IP |
|---|---|---|---|
| **pfSense (2.8.1 CE)** | Firewall / gateway | WAN (NAT), LAN, OPT1, OPT2 | 192.168.1.1 (LAN), 192.168.2.1 (OPT1), 192.168.131.4 (OPT2 mgmt) |
| **Kali Linux** | Attacker | atk-net (OPT1) | 192.168.2.x (DHCP) |
| **Ubuntu Server** | Target | tgt-net (LAN) | 192.168.1.x (DHCP) |

- **Adapter 1 (WAN, em0):** NAT — internet access for package downloads
- **Adapter 2 (LAN, em1):** `tgt-net`, 192.168.1.1/24, DHCP — Ubuntu target
- **Adapter 3 (OPT1, em2):** `atk-net`, 192.168.2.1/24, DHCP — Kali attacker
- **Adapter 4 (OPT2, em3):** Host-only, static 192.168.131.4 — management GUI access from host

## Tech stack

- **pfSense CE 2.8.1** — firewall/router
- **Kali Linux** — attacker tooling (nmap, hping3)
- **Ubuntu Server 26.04 LTS** — target host
- **Suricata 7.0.11** (ET Open ruleset) — IDS, installed as a pfSense package
- Diagnostics: `pfctl`, `journalctl`, `dmesg`

## What was built

1. **Segmented network** — full build with correct interface mapping, DHCP on both internal segments, verified cross-segment connectivity (Kali => pfSense => Ubuntu).
2. **Traffic generation** — multiple real attacker scan types run from Kali against Ubuntu: TCP connect scan, SYN scan (`-sS`), full 65,535-port scan, XMAS scan (`-sX`), NULL scan (`-sN`), FIN scan (`-sF`), and a sustained SYN flood via `hping3 --flood`.
3. **Working defense: port-based block rule** — a pfSense rule blocking inbound SSH (port 22) from the attacker segment. Verified working via two independent methods: `nc` connection timeout with visible TCP exponential backoff retransmission in the firewall log, and a follow-up scan showing the port reported as `filtered`.
4. **Tested defense: pf rate-limiting** (`max-src-conn-rate`) — configured, verified loaded at the kernel level via `pfctl -sr`, and tested against both a SYN scan and a 163,000-packet SYN flood. **Did not trigger in either case.**
5. **Tested defense: Suricata IDS** — installed, ET Open ruleset downloaded and confirmed loaded (726 signatures active, verified via `suricata.log`), tested against SYN, XMAS, NULL, and FIN scans at multiple port-count volumes. **No alerts generated for any scan type.**

## Key findings

### Finding 1 — `max-src-conn-rate` cannot detect handshake-avoiding traffic
pf's built-in connection-rate limiter only counts **fully established** TCP connections (completed 3-way handshake). Both nmap's SYN scan and hping3's flood are specifically designed to never complete that handshake — so this feature structurally cannot detect either, regardless of how it's tuned. Confirmed via kernel-level rule counters (`pfctl -vvsr`) showing 0 packets ever diverted to the rate-limit overload table, even under a sustained 44,855-connection-attempt flood. This is a documented, known limitation (see Netgate community forum), not a misconfiguration.

### Finding 2 — Default community IDS signatures did not catch evasive scans
With Suricata's ET Open `emerging-scan.rules` category confirmed active, none of four different scan techniques (including intentionally "loud"/abnormal flag combinations like XMAS and NULL scans) generated an alert. This suggests the default community ruleset is tuned toward known noisy tool fingerprints or multi-host sweep patterns rather than generic single-host scans — a real, useful limitation to understand rather than assume away.

### Overall takeaway
No single layer — stateful firewall rate-limiting or signature-based IDS — reliably caught evasive reconnaissance traffic on its own. The one thing that *did* work reliably was a simple, explicit, zone-based access rule (block SSH from an untrusted segment). This mirrors real-world defense-in-depth practice: perimeter access control, rate limiting, and signature detection each cover different threat patterns, and gaps in one layer are expected to be covered by another (e.g., behavioral/anomaly detection, which this lab did not implement).

## Repo structure

```
pfsense-homelab/
├── README.md
├── notes/
│   ├── day1.md   — network build, DHCP, cross-segment connectivity
│   ├── day2.md   — traffic generation, timezone/logging setup, host RAM incident, SSH block rule
│   ├── day3.md   — continued verification, protocol comparison, rate-limit root cause (hping3 flood)
│   └── day4.md   — Suricata install, ET Open ruleset, scan-type testing, final findings
└── screenshots/
    └── (firewall logs, rule configs, pfctl output, Suricata alerts page)
```

## Lessons learned

- pf evaluates rules top-to-bottom; a broad "allow" rule above a specific "block" rule silently defeats the block — a real, common misconfiguration pattern.
- pfSense only logs traffic a rule explicitly tells it to log; passed traffic is invisible by default.
- Config changes in web UIs don't always take effect on save — verify against a lower-level source of truth (kernel ruleset, engine logs) rather than trusting the UI state alone.
- A structurally negative result, properly diagnosed with evidence, is a legitimate and valuable security finding — not a failed test.
