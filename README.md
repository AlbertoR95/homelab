# 🔐 Cybersecurity Homelab

> A self-hosted cybersecurity and networking lab built on Proxmox VE, pfSense, and multiple virtual machines — documenting both the successes and the real-world challenges of building a lab from scratch.

---

## 🗺️ Architecture Overview

```
[Home WiFi Router]
        │
        │ (WAN - single Ethernet port)
        ▼
[Proxmox VE Host — Lenovo M710q]
        │
   ┌────┴─────────────────┐
   │                      │
[vmbr0]               [vmbr1]
(WAN bridge)       (LAN bridge — isolated)
   │                      │
[pfSense VM]──────────────┤
   │ WAN        LAN       │
   └──────────────────────┤
                          │
              ┌───────────┼───────────┐
              │           │           │
          [Kali VM]  [Windows 11]  [Debian VM]
         (attacker)  (target)      (server)
```

---

## 🛠️ Hardware

| Component | Details |
|-----------|---------|
| **Hypervisor host** | Lenovo M710q Mini PC |
| **Hypervisor** | Proxmox VE |
| **Firewall** | pfSense (VM inside Proxmox) |
| **Physical NICs** | 1x Ethernet (single-port constraint) |
| **Internet source** | Home WiFi → Ethernet bridge |

---

## 💻 Virtual Machines

| VM | OS | Role | Network |
|----|-----|------|---------|
| `pfsense` | pfSense CE | Firewall / Router | vmbr0 (WAN) + vmbr1 (LAN) |
| `kali` | Kali Linux | Attacker / Pentesting | vmbr1 |
| `win11` | Windows 11 | Target machine | vmbr1 |
| `debian` | Debian / Linux Mint | Server / general use | vmbr1 |

---

## 🎯 Objectives

- Learn practical networking: subnets, routing, VLANs, firewall rules
- Practice pentesting techniques in a safe, isolated environment
- Understand pfSense configuration from scratch
- Simulate real-world SOC and attacker/defender scenarios
- Document everything for learning and portfolio

---

## 📁 Folder Structure

```
homelab/
├── setup/          # Step-by-step installation guides
│   ├── proxmox.md
│   ├── pfsense-vm.md
│   └── vm-setup.md
├── systems/        # Per-OS notes and configurations
│   ├── kali.md
│   ├── windows11.md
│   └── debian.md
├── notes/          # Concepts, issues, troubleshooting logs
└── README.md
```

---

## 🧱 Build Journey & Key Decisions

This lab wasn't built in a straight line. Here are the real decisions and pivots made along the way — which turned out to be the most valuable learning.

### Phase 1 — Initial Proxmox Setup ✅
- Installed Proxmox VE on Lenovo M710q
- Assigned static IP `192.168.1.150`
- Created base VMs: Kali, Windows 11, Debian/Mint
- Basic networking via `vmbr0`

### Phase 2 — Physical pfSense Attempt ❌

**Goal:** insert a physical pfSense box between the router and Proxmox.

**Problems encountered:**
- WAN/LAN interface misassignment (`igc0`/`igc1` confusion)
- IP/subnet mismatches between pfSense LAN and Proxmox
- WiFi dependency: pfSense needs a wired WAN, but the environment relies on WiFi
- WPA3 incompatibility with cheap bridge devices (TP-Link TL-WR902AC)
- DHCP conflicts from manual IP configurations
- Complete connectivity loss when pfSense was inserted into the path

**Root cause:** The single Ethernet port + WiFi-only uplink made a physical firewall insertion architecturally impossible without additional hardware.

### Phase 3 — Virtual pfSense (Breakthrough) ✅

**Decision:** move pfSense entirely inside Proxmox as a VM.

**Why it worked:**
- Proxmox handles all physical networking
- pfSense only sees clean virtual NICs — no driver or hardware issues
- WPA3/WiFi problems disappear at the hypervisor layer
- Full isolation between WAN (vmbr0) and LAN (vmbr1) achieved virtually

**Key insight:** Virtualization doesn't just simplify labs — it removes entire classes of physical hardware problems.

---

## ⚙️ Network Configuration

### Proxmox Bridges

| Bridge | Role | Connected to |
|--------|------|-------------|
| `vmbr0` | WAN / uplink | Physical NIC → home router |
| `vmbr1` | Internal LAN | pfSense LAN, all lab VMs |

### pfSense VM

| Interface | Assignment | IP |
|-----------|-----------|-----|
| WAN | vmbr0 | DHCP (from home router) |
| LAN | vmbr1 | `10.10.10.1/24` (static) |

### DHCP Range (pfSense LAN)
`10.10.10.100` → `10.10.10.200`

---

## ✅ Current Status

| Component | Status |
|-----------|--------|
| Proxmox VE | ✅ Running |
| Kali Linux VM | ✅ Running |
| Windows 11 VM | ✅ Running |
| Debian VM | ✅ Running |
| pfSense VM | 🔧 Configuring |
| vmbr0 / vmbr1 bridges | ✅ Configured |
| Internet via WiFi→Ethernet | ⚠️ In progress |
| DHCP on LAN | 🔧 Configuring |
| VLANs | 📋 Planned |
| IDS (Snort/Suricata) | 📋 Planned |

---

## 🚀 Roadmap

- [ ] Stable internet connection (WiFi 6 router with WISP/client mode)
- [ ] Complete pfSense LAN/WAN/DHCP configuration
- [ ] Validate full VM internet access through pfSense
- [ ] Implement VLAN segmentation on managed switch
- [ ] Configure firewall rules and traffic logging
- [ ] Deploy IDS/IPS (Snort or Suricata on pfSense)
- [ ] First pentesting exercises: Kali → Windows 11
- [ ] VPN setup for remote access

---

## 🧠 Lessons Learned

1. **WiFi ≠ Ethernet** — especially for firewall setups. pfSense is designed for wired WAN.
2. **Virtualisation solves hardware problems** — moving pfSense into a VM eliminated an entire category of physical compatibility issues.
3. **Reduce variables when debugging** — isolate one layer at a time.
4. **Same subnet first** — always verify IP/subnet alignment before debugging routing.
5. **Consumer hardware has limits** — cheap WiFi bridges and WPA3 don't mix well.
6. **Document failures** — they're often more instructive than successes.

---

## 🔗 References & Tools

- [Proxmox VE Documentation](https://pve.proxmox.com/pve-docs/)
- [pfSense Documentation](https://docs.netgate.com/pfsense/en/latest/)
- [Kali Linux](https://www.kali.org/)

---

*This is an active learning project. The goal is not a perfect setup — it's understanding how every piece works.*
