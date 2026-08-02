# Day 4 — Suricata Install and Signature Detection Testing

## Goal
Install Suricata IDS on pfSense to properly attempt detection of the SYN scan/flood that pf's rate-limiting could not catch.

## Suricata install and configuration

- Installed via **System => Package Manager**.
- Added an instance on **OPT1** (the interface facing Kali/`atk-net`) — correct placement to inspect attacker traffic before it reaches Ubuntu.
- Enabled "Send Alerts to System Log" logging option.
- Initial rule set only included Suricata's built-in "Default Rules" (protocol-anomaly detectors like `app-layer-events.rules`, `decoder-events.rules`) — not actual threat signatures.

## Enabling the ET Open ruleset

- Enabled "Install ETOpen Emerging Threats rules" under **Global Settings**.
- First download attempt via **Updates => Force** failed (unstable host wifi causing a timeout mid-download). Succeeded on retry — MD5 hash generated, confirming successful download.
- Enabled `emerging-scan.rules` category under **OPT1 Categories**.
- **First save silently failed to take effect** — confirmed later by revisiting the page and finding the box unchecked despite believing it was saved. Re-checked and saved properly the second time.
- Verified the fix via `suricata.log`: rule count went from **381 rules loaded** (Default Rules only) → **726 rules loaded** after the properly-saved category was picked up on restart — confirming ET scan signatures were genuinely active.

## Detection testing

Tested Suricata's `emerging-scan.rules` against four different scan techniques, at multiple port-count volumes, with the ruleset confirmed loaded:

| Scan type | Command | Result |
|---|---|---|
| SYN scan | `nmap -sS --top-ports 100` | No alert |
| SYN scan | `nmap -sS -p 1-1000` | No alert |
| XMAS scan | `nmap -sX --top-ports 100` | No alert |
| NULL scan | `nmap -sN --top-ports 100` | No alert |
| FIN scan | `nmap -sF --top-ports 100` | No alert |

Checked **Services => Suricata => Alerts** after every test — consistently empty across all scan types.

## Troubleshooting note: transient network failure

Mid-testing, all connectivity between Kali/Ubuntu/pfSense briefly failed ("failed to determine route", pings to gateways failing on both sides). Diagnosed as a host-level network interface change (switched from wifi to a phone hotspot), which affected the VirtualBox NAT adapter (pfSense WAN) and caused a brief hang. Confirmed **not** a lab misconfiguration — traffic resumed normally once the host network stabilized. Ubuntu itself was checked directly and confirmed healthy throughout (13% memory usage, correct IP, interface up) — ruling it out as the cause.

## Final finding

Two independent detection mechanisms — pf's `max-src-conn-rate` (Day 3) and Suricata's ET Open `emerging-scan.rules` (Day 4) — **both failed to detect evasive nmap scan traffic**, for two different but related reasons:

- pf structurally cannot see connections that never complete a handshake.
- Suricata's default community scan signatures appear tuned toward noisier tool fingerprints or multi-host sweep patterns rather than a single clean scan against one already-known host — nmap's default techniques are deliberately among the least detectable, which is exactly why they're popular with real penetration testers.

## Lessons learned

1. Config checkboxes in web UIs don't always save on the first attempt — verify against a lower-level source of truth (e.g., engine log rule-load counts) rather than trusting the UI state alone.
2. A "textbook clean" scan is specifically hard for common signature-based detection to catch — a real, well-documented limitation, not a lab failure.
3. Two consecutive negative results, properly diagnosed with logs and evidence, demonstrate a genuine security principle: **no single tool catches everything.** This is the practical justification for defense-in-depth — layering perimeter access control, rate limiting, signature-based detection, and behavioral/anomaly detection, since each layer has blind spots the others can cover.

## Overall lab conclusion

Across four days: a fully working segmented network was built, real attacker traffic was generated using multiple techniques, and one detection/defense mechanism (a zone-based port-block rule) was implemented and verified working end-to-end with clear evidence. Two more advanced detection mechanisms (pf rate-limiting, default-ruleset IDS) were tested rigorously and found to have genuine, well-documented limitations against evasive scanning techniques — a result that illustrates, with direct hands-on evidence, why real security architectures don't rely on any single defensive layer.
