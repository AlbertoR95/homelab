# pfSense Firewall Setup

This document covers the full journey of setting up pfSense as the lab firewall — including the physical attempt that failed, why it failed, and the virtual solution that worked.

---

## 🎯 Objective

Deploy pfSense as the primary firewall for the homelab, sitting between the internet uplink and all lab VMs, providing NAT, DHCP, and traffic segmentation.

---

## Phase 1 — Physical pfSense (Attempted)

### Intended Architecture

```
[ Internet / ISP ]
        │
   [ Home Router ]
        │
     (WAN port)
   [ pfSense box ]   ← dedicated micro-PC (Celeron J1900), dual NIC
     (LAN port)
        │
    [ Switch ]
        │
[ Proxmox Server + LAN devices ]
```

**Hardware used:**
| Component | Details |
|-----------|---------|
| pfSense box | Dedicated micro-PC, 2x Ethernet (WAN + LAN) |
| Proxmox host | Separate micro-PC |
| Switch | Managed switch |
| Uplink | Home WiFi router |

---

### Problems Encountered

#### 1. No internet when pfSense inserted into the path
- Direct connection `Router → Proxmox` worked fine
- As soon as pfSense was in the middle: no connectivity on LAN side
- pfSense WAN correctly got an IP from the router via DHCP
- LAN side was active but no traffic passed through

#### 2. LAN/WAN subnet mismatch
- pfSense LAN defaulted to `192.168.10.1/24`
- Home router was on `192.168.1.1/24`
- Proxmox was configured for the router subnet, not the pfSense LAN subnet
- Required manual reconfiguration of `/etc/network/interfaces` on Proxmox

#### 3. Interface assignment confusion
- `igc0` / `igc1` were not consistently assigned to WAN/LAN
- Swapping interfaces without a monitor connected made it hard to verify
- Required multiple resets via pfSense console

#### 4. WiFi uplink incompatibility
- The environment relies on WiFi → Ethernet bridging (TP-Link TL-WR902AC)
- The bridge device had WPA3 incompatibility issues with the home router
- Resulted in unstable or no uplink, making WAN configuration unreliable

#### 5. DHCP conflicts
- Multiple manual IP assignments created address conflicts
- Difficult to isolate which device had which IP at any point

---

### Workarounds Attempted

- Changed Proxmox network config to match pfSense LAN subnet
- Tested direct connection: pfSense LAN → Proxmox (bypassing switch)
- Disabled "Block private networks" on pfSense WAN interface
- Verified pfSense firewall rules (allow any on LAN)
- Confirmed WAN DHCP lease from router
- Factory-reset pfSense multiple times

---

### Why It Still Didn't Work — Root Cause Analysis

After eliminating obvious causes (cabling, switch, DHCP, subnet overlap), the core issue became clear:

> **The environment was fundamentally incompatible with a physical pfSense deployment.**

Specific reasons:
1. **Single Ethernet port on Proxmox host** — no clean way to separate WAN and LAN physically
2. **WiFi-only uplink** — pfSense expects a stable wired WAN; the WiFi bridge introduced instability
3. **WPA3 incompatibility** — cheap bridge hardware couldn't reliably connect to the home router
4. **Too many layers** — WiFi bridge → switch → pfSense → server created too many failure points to debug reliably

This was not a configuration error that could be fixed with more tweaking — it was an architectural mismatch.

---

## Phase 2 — Virtual pfSense on Proxmox ✅

### Decision

Move pfSense entirely inside Proxmox as a VM, using virtual network bridges instead of physical interfaces.

### New Architecture

```
[ Home Router ]
       │
[ Proxmox Host ] ── physical NIC
       │
  ┌────┴──────────────────┐
  │                       │
[vmbr0]               [vmbr1]
(WAN bridge)       (LAN bridge — isolated)
  │                       │
[pfSense VM] ─────────────┤
  WAN → vmbr0             │
  LAN → vmbr1             │
                ┌──────────┼──────────┐
                │          │          │
           [Kali VM]  [Win11 VM]  [Debian VM]
```

### pfSense VM Configuration

| Setting | Value |
|---------|-------|
| VM type | Other (FreeBSD) |
| RAM | 1–2 GB |
| Disk | 16 GB |
| NIC 1 (WAN) | vmbr0 — gets DHCP from home router |
| NIC 2 (LAN) | vmbr1 — static `10.10.10.1/24` |

### Proxmox Bridge Setup (`/etc/network/interfaces`)

```bash
# WAN bridge — connects to physical NIC and home router
auto vmbr0
iface vmbr0 inet dhcp
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0

# LAN bridge — isolated internal network, no physical port
auto vmbr1
iface vmbr1 inet static
    address 10.10.10.254/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
```

### pfSense LAN Settings

| Setting | Value |
|---------|-------|
| LAN IP | `10.10.10.1/24` |
| DHCP range | `10.10.10.100` – `10.10.10.200` |
| DNS | `1.1.1.1`, `8.8.8.8` |
| NAT | Automatic outbound NAT enabled |

### VM Network Settings (Kali, Windows, Debian)

- **Bridge:** `vmbr1`
- **IP config:** DHCP (from pfSense)
- **Gateway:** `10.10.10.1` (pfSense LAN)

---

### Why This Worked

| Problem (Physical) | Solution (Virtual) |
|--------------------|--------------------|
| WiFi WPA3 incompatibility | Proxmox handles WiFi — pfSense never touches it |
| Physical NIC driver issues | Virtual NICs — no driver problems |
| Interface assignment confusion | Clean vmbr0/vmbr1 separation, no physical ambiguity |
| Single Ethernet port | Irrelevant — segmentation is virtual |
| DHCP conflicts | Clean isolated vmbr1 network |

> **Key insight:** Virtualisation doesn't just simplify the setup — it removes entire categories of physical hardware problems.

---

## Current Status

| Task | Status |
|------|--------|
| pfSense VM created in Proxmox | ✅ Done |
| vmbr0 (WAN) configured | ✅ Done |
| vmbr1 (LAN) configured | ✅ Done |
| pfSense WAN → DHCP from router | 🔧 In progress (WiFi uplink issue) |
| pfSense LAN → DHCP to VMs | 🔧 In progress |
| VMs receiving IP on vmbr1 | 🔧 In progress |
| Internet access through pfSense | 📋 Pending stable uplink |
| Firewall rules configured | 📋 Planned |
| VLANs | 📋 Planned |
| IDS/IPS (Snort or Suricata) | 📋 Planned |

---

## Lessons Learned

1. **WiFi ≠ Ethernet** — pfSense is designed for wired WAN; don't fight this constraint physically.
2. **Virtualisation solves hardware problems** — moving to a VM eliminated the entire physical layer of issues.
3. **Reduce variables when debugging** — test one segment at a time (LAN only, then add WAN).
4. **Subnet alignment first** — before debugging routing, make sure all devices are in the correct subnet.
5. **Document failures** — the physical attempt failure led directly to a better architectural understanding.
6. **Root cause vs. symptoms** — many symptoms (DHCP issues, no internet) had one root cause (WiFi uplink incompatibility).
