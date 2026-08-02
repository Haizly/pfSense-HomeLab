# Day 2 — Traffic Generation, Logging, and First Block Rule

## Goal
Generate attacker-style traffic from Kali toward Ubuntu, get it properly logged on pfSense, and write a working firewall rule to block it.

## Traffic generation
Ran several nmap scan types from Kali against Ubuntu (192.168.1.100), including:
- `nmap -sT` — TCP connect scan (full handshake)
- `nmap -sS -p 1-65535` — SYN scan across all ports (found port 22/SSH open)
- Lighter scans (`--top-ports 100`, `-T4`) used later to reduce host load

## Setup fixes required before traffic was useful

**Timezone mismatch:** pfSense and Kali were logging ~7 hours apart, making log correlation confusing. Fixed via **System => General Setup => Timezone** (standardized to match Kali's zone).

**Logging not enabled by default:** pfSense only logs traffic a rule explicitly tells it to log — passed traffic is invisible otherwise. Enabled "Log packets that are handled by this rule" on the OPT1 pass rule.

## Host resource incident

Running all three VMs at 2048MB RAM each pushed the host to ~14/15.7GB used during a full 65,535-port scan. Ubuntu froze (flashing caps lock LED, initially suspected kernel panic). Recovery:
- Pausing Kali freed host RAM but did not recover Ubuntu.
- Forced ACPI shutdown => hard power-off on Ubuntu.
- Checked `journalctl -b -1 -p err` and `dmesg | grep -i oom` after reboot — **no kernel panic, no OOM kill found.**
- Conclusion: likely host-level RAM pressure across all VMs simultaneously, not a genuine Ubuntu crash.
- Lesson: budget host RAM carefully before heavy scans with 3 VMs open; prefer lighter scans (`--top-ports`) over full-range scans when resources are tight.

## First working block rule: SSH

Added a rule on OPT1: **Block TCP, source any, destination 192.168.1.100, port 22**, with logging enabled.

**First attempt failed** — `nc -zv 192.168.1.100 22` still showed "open." Root cause: **rule ordering**. The block rule was added *below* the existing "Allow atk-net traffic" pass-any rule. pf evaluates top-to-bottom and stops at the first match, so the pass rule caught traffic before the block rule was ever evaluated.

**Fixed by reordering** the block rule above the pass rule.

**Re-tested and confirmed working:**
- `nc -zv 192.168.1.100 22` => "Connection timed out" (silent drop, correct Block behavior)
- `nc -zv 192.168.1.100 80` => "Connection refused" (passed through pfSense; Ubuntu itself refused since nothing listens on port 80 — correct pass behavior)
- Firewall log showed clean, repeated block entries for port 22, with visible TCP retransmission backoff (1s, 1s, 1s, 2s, 4s, 8s, 16s, 33s...) — a textbook signature of a silently-dropped SYN.

## Key lessons

1. **pf rule order matters — first match wins.** A broad allow rule above a specific block rule silently defeats the block. Common real-world misconfiguration, not just a lab quirk.
2. **pfSense only logs what a rule tells it to log.**
3. **Timezone mismatches between systems make log correlation confusing** — worth standardizing (UTC is a common convention).
4. **`pfctl -sr` and `pfctl -s Sources`** are useful for confirming actual kernel-level rule state rather than trusting the GUI alone.
5. Running 3 VMs at 2048MB each is tight on a 16GB host under scan load — monitor RAM before heavy tests.
