# pfSense on Proxmox – Troubleshooting

## Objective
Deploy pfSense on a dedicated micro-PC (Celeron J1900) to act as the primary firewall for a Proxmox server network.

**Hardware:**
- **pfSense device:** Dedicated micro-PC with 2 Ethernet ports (WAN, LAN).
- **Proxmox server:** Separate micro-PC 
- **Switch:** Used for connecting LAN devices.
- **Router:** Home Wi-Fi router providing Internet.

---

## Intended Network Design

**Planned flow:**

[ Internet / ISP ]
|
[ Router ]
|
(WAN port)
[ pfSense ]
(LAN port)
|
[ Switch ]
|
[ Proxmox Server + other LAN devices ]


- pfSense WAN → Connected to router (gets public/ISP-side IP).
- pfSense LAN → Connected to switch, providing DHCP to Proxmox and other devices.

---

## Issues Encountered

### 1. **No Internet when pfSense is in the middle**
- All devices worked fine directly connected to the router.
- When connecting `Router → pfSense → Proxmox`, Proxmox lost Internet.
- pfSense WAN successfully obtained IP from router, but LAN devices had no connectivity.

**Possible causes considered:**
- Incorrect LAN subnet (overlapping with WAN).
- DHCP conflicts.
- Firewall rule misconfiguration.

---

### 2. **Switch Was Not the Problem**
- Verified that `Router → Switch → Proxmox` worked fine.
- Indicates cabling and switch ports were functional.

---

### 3. **LAN/WAN Subnet Mismatch**
- pfSense LAN defaulted to `192.168.10.1/24` while home router used `192.168.1.1/24`.
- This required Proxmox to be in the same subnet as pfSense LAN to access its GUI and get DHCP leases — which initially wasn’t set.

---

## Workarounds Attempted
- Changed Proxmox `/etc/network/interfaces` to match pfSense LAN subnet.
- Tested direct connections:
  - pfSense LAN → Proxmox (bypassing switch).
  - pfSense LAN → Switch → Proxmox.
- Disabled "Block private networks" in pfSense WAN settings.
- Checked pfSense firewall rules (allow any on LAN).
- Confirmed WAN IP lease from router.
- Verified that Proxmox worked fine when bypassing pfSense.

---

## Why It Still Didn’t Work
- Despite the Proxmox server having multiple NICs and being configured with one port for WAN and another for LAN, connectivity issues persisted. WAN on pfSense consistently obtained an IP from the router, and LAN appeared to be active, but no traffic successfully passed through to the Proxmox server or connected devices.  
At this stage, all obvious causes — incorrect cabling, switch issues, firewall misconfiguration, DHCP conflicts, and subnet overlaps — had been investigated and either ruled out or corrected.  
The exact root cause remained undetermined, highlighting a gap in my current network troubleshooting expertise, particularly in complex multi-NIC routing scenarios. This became a clear learning opportunity for deeper study into pfSense routing logic and interface bridging in physical deployments.

---

## Lessons Learned
1. Ensure **pfSense LAN subnet** is compatible and reachable by all internal devices.
2. Always verify **LAN DHCP service** is working before connecting multiple devices.
3. Document each IP change — mismatched addressing can silently block everything.
4. In physical setups, simplify first: test pfSense LAN with a single laptop before inserting servers/switches.
5. Keep diagrams of the physical and logical network to speed up troubleshooting.
