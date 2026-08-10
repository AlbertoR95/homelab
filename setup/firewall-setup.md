# pfSense Firewall Setup

The firewall is a dedicated micro-PC running pfSense 2.7.2 with four Intel I226-V network ports. It sits between the WiFi client bridge and the lab network, providing NAT, DHCP, and DNS for everything behind it.

This document covers the working configuration and how it was reached — including two earlier attempts that didn't survive contact with the environment.

## Interface assignment

| Interface | Port | Configuration |
|---|---|---|
| WAN | igc0 | DHCP, currently 192.168.1.194 |
| LAN | igc1 | Static 10.10.10.1/24 |
| OPT1 | igc2 | Unassigned |
| OPT2 | igc3 | Unassigned |

Both interfaces were assigned through auto-detection from the console menu rather than by typing interface names. This matters more than it sounds: `igc0` through `igc3` give no indication of which physical socket they correspond to, and the mapping depends on how the manufacturer wired the board. Guessing wrong produces a firewall that looks correctly configured and passes no traffic.

Auto-detection has one behaviour worth knowing. Pressing `a` takes a snapshot of which ports are currently up; you then connect the cable and press ENTER, and pfSense reports what changed. If the cable is already connected when you press `a`, that port is already up in the snapshot, nothing changes, and it reports `No link-up detected`. The cable goes in *after* the prompt appears, not before.

## LAN configuration

Set from console option 2:

| Setting | Value |
|---|---|
| IPv4 address | 10.10.10.1/24 |
| Upstream gateway | none |
| IPv6 | disabled |
| DHCP server | enabled |
| DHCP range | 10.10.10.100 – 10.10.10.200 |

The upstream gateway field is left empty deliberately. It's tempting to enter the home router's address there, but the LAN has nothing upstream of it — pfSense *is* the gateway for that segment. Filling it in creates a routing loop that's awkward to unpick.

The DHCP range stops at `.200`, leaving `.2`–`.99` and `.201`–`.254` for static assignments. Proxmox uses `.10`, the switch uses `.250`. A static address inside the DHCP pool will eventually be handed to something else.

## Settings that differ from defaults

**Block RFC1918 private networks — disabled on WAN.** This option is enabled by default and normally correct: it stops private addresses arriving from the internet. Here the WAN address *is* private, because pfSense sits behind the home router. Leaving it enabled makes pfSense discard its own upstream traffic.

**DNS servers set explicitly, override disabled.** `1.1.1.1` and `8.8.8.8` are configured under System → General Setup, with *Allow DNS server list to be overridden by DHCP/PPP on WAN* unchecked. Without unchecking it, pfSense adopts whatever DNS the upstream hands it — in this case the WiFi bridge, which doesn't resolve anything.

**DNS Resolver listening on all interfaces.** Under Services → DNS Resolver, Network Interfaces is set to `All`. Restricting it to `LAN` is rejected outright: pfSense uses its own resolver, so `Localhost` must always be included. The more subtle failure is leaving it at the default of localhost only — pfSense then resolves perfectly for itself while every client on the LAN times out. Query forwarding is also enabled, which sends lookups to the configured upstream servers rather than performing full recursion. Recursion tends to be unreliable behind multiple layers of NAT.

## General settings

| Setting | Value |
|---|---|
| Hostname | homelab |
| Domain | homelab.lan |
| Timezone | Europe/London |

The timezone isn't cosmetic. Firewall logs are timestamped with it, and correlating an event against anything external is painful when the clock is in the wrong zone.

## Access

The web interface is at `https://10.10.10.1` over HTTPS with a self-signed certificate, so browsers warn on first connection. SSH is enabled from console option 14 as a fallback route in if the web interface ever becomes unreachable.

A configuration backup can be downloaded from Diagnostics → Backup & Restore as XML. Worth doing after any significant change — it restores the entire setup in minutes rather than repeating the process from scratch.

## Earlier attempts

### Physical, first attempt — abandoned

The original plan was the same as the current one: a dedicated firewall between the home router and the lab. It failed, and for a long time the failure looked like a configuration problem.

The symptoms were varied. Interfaces couldn't be reliably matched to physical ports. The pfSense LAN and the Proxmox host ended up on different subnets and couldn't communicate. Inserting the firewall into the path took the whole network offline. DHCP handed out addresses that led nowhere.

None of these were the actual problem. The environment had no wired uplink — the router was too far to reach with a cable — so a cheap WiFi bridge stood in for one, and it couldn't hold a stable connection against a WPA3 router. Every symptom above was downstream of an uplink that came and went.

Recognising that took longer than it should have, because each symptom looked like a self-contained problem with its own plausible fix. Chasing them individually meant never questioning the foundation they all rested on.

### Virtual — worked, then set aside

Moving pfSense inside Proxmox as a VM removed every hardware variable at once. Two virtual bridges carried WAN and LAN, pfSense saw clean virtual NICs, and the WiFi problems stayed at the hypervisor layer where Proxmox already handled them.

It worked, and it confirmed the network design itself was sound — the problem had always been physical, not logical.

It was set aside for one reason: a firewall inside the hypervisor can only protect what's inside the hypervisor. Physical devices can't sit behind it, and the spare ports that make real segmentation possible don't exist.

### Physical, second attempt — current build

After relocating to somewhere with a usable uplink, the original design became viable. Same hardware, same topology, and it came up cleanly.

The remaining problems were small and specific rather than structural: interfaces needed assigning, the LAN default of `192.168.1.1/24` collided with the home network and had to be moved, and the DNS resolver needed its listening interfaces corrected. Each took minutes once identified, because there was a stable foundation underneath them.

The difference between this attempt and the first wasn't skill. It was that the one blocking constraint had gone.

## What this build taught

**Symptoms cluster around a single cause.** Six apparently separate problems in the first attempt all traced back to an unstable uplink. Fixing them one at a time was never going to work.

**A service can work for itself and not for its clients.** The DNS resolver resolved perfectly for pfSense while silently ignoring the LAN. "It works on the box" and "it works for the network" are different claims.

**Test one layer at a time.** Pinging the gateway, then an external IP, then a domain name isolates local connectivity, routing, and name resolution as three separate results. A single test that fails tells you far less.

**Environmental constraints aren't configuration problems.** No amount of tuning fixes a design that the physical environment can't support. Recognising the difference early saves considerable time.
