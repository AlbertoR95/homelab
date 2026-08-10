# Cybersecurity Homelab

A segmented home network built around a dedicated pfSense firewall, a Proxmox virtualisation host, and a managed switch. The goal is hands-on experience with networking, virtualisation, and defensive security — and to document the process honestly, including the parts that took several attempts to get right.

## Current architecture

```
Home router (192.168.1.254)
        |  WiFi
TP-Link TL-WR902AC — client mode (192.168.1.250)
        |  Ethernet
pfSense — WAN igc0 (192.168.1.194, DHCP)
        |
pfSense — LAN igc1 (10.10.10.1/24)
        |
HORACO 2.5GbE managed switch (10.10.10.250)
        |
   +----+----+
Proxmox    Workstation
10.10.10.10   DHCP
```

The uplink is the one part of this setup that isn't ideal. The router is too far from the lab to run a cable, so a WiFi client bridge sits between the two. It works, but it's the weakest link in the chain and worth remembering when something behaves oddly.

## Hardware

| Component | Model | Role |
|---|---|---|
| Firewall | Micro-PC, 4x Intel I226-V | pfSense 2.7.2, physical |
| Hypervisor | Lenovo M710q | Proxmox VE |
| Switch | HORACO 2.5GbE, 8 port + SFP+ | Managed, VLAN-capable |
| WiFi bridge | TP-Link TL-WR902AC v4 | Client mode, WiFi to Ethernet |

## Addressing

The lab runs on `10.10.10.0/24`, deliberately separate from the home network on `192.168.1.0/24`. Keeping them apart matters: if pfSense had the same subnet on both sides, routing breaks in ways that are genuinely difficult to diagnose.

| Host | Address | Notes |
|---|---|---|
| pfSense LAN | 10.10.10.1 | Gateway, DHCP and DNS for the lab |
| Proxmox | 10.10.10.10 | Static, outside the DHCP pool |
| Switch management | 10.10.10.250 | Static |
| DHCP pool | 10.10.10.100–200 | Clients and VMs |

Static addresses sit below `.100` and above `.200` so they can never collide with a lease.

## pfSense interfaces

| Interface | Port | Assignment |
|---|---|---|
| WAN | igc0 | DHCP from the WiFi bridge |
| LAN | igc1 | 10.10.10.1/24, DHCP server enabled |
| OPT1 | igc2 | Free — planned for an isolated workstation segment |
| OPT2 | igc3 | Free |

Both interfaces were assigned using pfSense's auto-detection rather than by name. Interface names don't tell you which physical socket they correspond to, and guessing wrong was one of the things that derailed the first attempt at this build.

Two settings that aren't defaults and matter here:

- **Block RFC1918 private networks is disabled on WAN.** The WAN address is itself a private address, so leaving the block enabled makes pfSense discard its own upstream traffic.
- **DNS Resolver listens on all interfaces, with forwarding enabled.** Restricted to localhost it resolves for pfSense but silently ignores every client on the LAN.

## Proxmox

The host has a single physical NIC (`enp0s31f6`) bridged to `vmbr0`, which carries the static lab address. A second bridge, `vmbr1`, has no physical port and exists as a fully isolated network for experiments that shouldn't reach the internet at all.

`vmbr1` originally used `10.10.10.1/24`, left over from an earlier attempt at running pfSense as a VM. That address is now the physical firewall's LAN, so the two collided — the host would have had two routes to the same subnet, one of them a dead end. It now sits on `10.10.99.0/24`.

## Build history

**First attempt — physical pfSense.** The intended design put a dedicated firewall between the home router and the lab. It never worked. The real cause wasn't configuration: there was no wired uplink available, and the cheap WiFi bridge standing in for one couldn't hold a stable connection against a WPA3 router. Everything downstream of that — interface confusion, subnet mismatches, DHCP conflicts — was a symptom.

**Second attempt — pfSense as a VM.** Moving the firewall inside Proxmox removed every hardware variable at once. It worked, and proved the network design was sound. But it also meant the firewall could only protect virtual machines, not physical devices.

**Current build — physical pfSense, second time.** After moving to a location with a usable uplink, the original design became viable. The same hardware, the same topology, and this time it came up cleanly. What changed wasn't skill — it was that the one blocking constraint had gone.

A firewall on real hardware is worth the extra effort here: physical machines can sit behind it, spare ports allow real segmentation, and it behaves like the equipment it's modelled on.

## Status

Working: WiFi bridge, pfSense WAN and LAN, DHCP, DNS resolution for clients, Proxmox on the lab network, managed switch with static management address.

Not yet done: virtual machines are still on the old network configuration and need reconnecting. Firewall rules are at defaults. VLANs, IDS, and remote access are planned but untouched.

## Roadmap

- Reconnect Kali, Windows 11 and Debian to the lab network
- Split the workstation onto OPT1 as a separate segment with rules between it and the lab
- VLAN segmentation across the managed switch
- Firewall rules and traffic logging
- Suricata for intrusion detection, with traffic generated from Kali as a test
- Forward pfSense logs to a SIEM
- WireGuard for remote access

## Notes on measurement

The WiFi uplink measures roughly 4% packet loss with 2.8 ms average latency. Low latency alongside scattered drops points to interference rather than weak signal. It's recorded here as a baseline: if something misbehaves later and the loss reads 15%, that's a change worth investigating rather than a constant to work around.

## Documentation

- `setup/firewall-setup.md` — pfSense configuration and the full history of getting there
- `setup/network-bridge.md` — WiFi client bridge setup
- `setup/proxmox-setup.md` — host network configuration
- `notes/troubleshooting.md` — problems encountered, with symptoms and diagnosis

## References

- [Proxmox VE Documentation](https://pve.proxmox.com/pve-docs/)
- [pfSense Documentation](https://docs.netgate.com/pfsense/en/latest/)
