# Day 1 — Network Build

## Goal
Build a segmented lab network in VirtualBox: pfSense (firewall/gateway), Kali (attacker), Ubuntu (target).

## What was done

- Installed pfSense 2.8.1 CE with 4 network adapters:
  - Adapter 1 => NAT (WAN, em0) — internet access
  - Adapter 2 => Internal Network `tgt-net` (LAN, em1) => 192.168.1.1/24, DHCP (.100–.199)
  - Adapter 3 => Internal Network `atk-net` (OPT1, em2) => 192.168.2.1/24, DHCP (.100–.199)
  - Adapter 4 => Host-only Adapter (OPT2, em3) => 192.168.131.4, static — used for GUI access from host
- Added explicit "Pass any/any" rules on OPT1 and OPT2 — both interfaces required manual rules since only LAN gets an automatic allow-all rule by default.
- Installed Ubuntu Server on `tgt-net`, confirmed DHCP-assigned IP in the 192.168.1.x range, confirmed ping to pfSense (192.168.1.1).
- Imported a pre-built Kali Linux (OffSec image) on `atk-net`, confirmed DHCP-assigned IP in the 192.168.2.x range, confirmed ping to pfSense (192.168.2.1).
- **Confirmed full cross-segment connectivity:** Kali successfully pinged Ubuntu (192.168.1.100) through pfSense.

## Key gotchas

- **pfSense default-deny behavior:** all non-LAN interfaces (OPT1, OPT2) block everything until a Pass rule is manually added. LAN is the only interface that gets an automatic allow-all rule out of the box.
- **VirtualBox adapter numbers =/= pfSense interface names.** In this build: LAN = em1 = Adapter 2, OPT1 = em2 = Adapter 3. Easy to cross-wire if not tracked carefully.

## Status at end of day
All three VMs operational. Core infrastructure confirmed working: pfSense GUI accessible via host-only adapter, DHCP active on both LAN and OPT1, full cross-segment connectivity verified.
