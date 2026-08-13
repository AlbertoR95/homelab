# Issues Log

Problems encountered while building the lab, with the symptom as it actually presented, the diagnosis, and the fix. Recorded because the diagnostic path is usually more useful than the solution.

---

## WiFi bridge running in the wrong mode

**Symptom.** Laptop connected to the bridge by cable received an IP address and could reach the bridge's own web panel, but had no internet. Everything looked connected. Turning off the laptop's WiFi killed all connectivity, which revealed the cable had never been carrying traffic.

**Diagnosis.** The bridge's status page showed `Operation Mode: Access Point` on both radios. Access Point mode takes Ethernet in and produces WiFi out — the exact opposite of what was needed. The device was also running its own DHCP server, so the laptop had received an address and a default gateway pointing at the bridge itself, which does no routing in that mode.

**Fix.** Changed operation mode to Client. Note that on this hardware a physical slide switch on the side of the device limits which modes are selectable in the web interface.

**Lesson.** An IP address and a default route prove nothing about whether traffic actually reaches anywhere. "Connected but no internet" almost always means the gateway isn't a gateway.

---

## Wireless radio disabled after changing modes

**Symptom.** After switching to Client mode, the status page showed `Wireless Radio: Disabled` and an empty `Name (SSID) of Root AP` field.

**Diagnosis.** Changing operation mode clears the wireless configuration. The network scan and password entered under the previous mode no longer applied — the bridge was in the correct mode but had no network to associate with.

**Fix.** Enabled the radio, ran a fresh survey, selected the home network and re-entered the password.

**Lesson.** Configuration order matters. Set the operating mode first, then everything else. Configuring details before a mode change means redoing them afterwards.

---

## Two DHCP servers on the home network

**Symptom.** Clients on the home network intermittently received addresses that led nowhere. DNS lookups failed while direct IP connections worked.

**Diagnosis.** The bridge's DHCP server was still enabled and handing out addresses on `192.168.1.0/24` — the same subnet the home router serves. Two DHCP servers on one segment means whichever replies first wins, which is effectively random. Clients that got a lease from the bridge received a gateway and DNS server pointing at a device that neither routes nor resolves.

**Fix.** Disabled the DHCP server on the bridge. In client mode it should be transparent and serve nothing.

**Lesson.** This affected every device in the house, not just the lab. A misplaced DHCP server is one of the few lab mistakes that reaches beyond the lab.

---

## Bridge management interface unreachable

**Symptom.** The bridge stopped responding at its configured address. Pinging returned 100% loss.

**Diagnosis.** Two separate causes at different points. First, the bridge held a static address on `192.168.0.1` while the home network turned out to be `192.168.1.0/24` — different subnets, no route between them. Later, with LAN type set to Smart IP, the bridge took a DHCP lease from the router and moved to an address nobody had recorded.

**Fix.** Located it by MAC address:

```
sudo arp-scan --localnet
```

Then set a static address on the correct subnet, above the DHCP pool.

**Lesson.** A management address obtained by DHCP will move without warning. Anything you need to reach reliably gets a static address outside the pool, or a reservation on the router.

---

## Stale DNS after infrastructure changes

**Symptom.** `ping 8.8.8.8` succeeded with no packet loss. `ping google.com` failed with `Temporary failure in name resolution`. Browsing hung and then timed out.

**Diagnosis.** `/etc/resolv.conf` still listed a DNS server from an earlier configuration — a device that had since changed role and address. Routing was fine; only name resolution was broken. `nslookup google.com 8.8.8.8` succeeding while the system resolver failed confirmed it.

**Fix.** Renewed the DHCP lease:

```
nmcli device disconnect enp11s0 && nmcli device connect enp11s0
```

**Lesson.** This recurred three times across the build. Clients hold DNS settings from whatever served them last, and no amount of upstream reconfiguration updates them. Renew the lease after every infrastructure change.

---

## pfSense auto-detection reporting no link

**Symptom.** Console interface assignment returned `No link-up detected` despite a cable being connected and the port LED active.

**Diagnosis.** Auto-detection compares port states before and after. Pressing `a` snapshots which ports are currently up; connecting the cable then changes that state; pressing ENTER reports the difference. With the cable already connected at snapshot time, nothing changed.

**Fix.** Disconnected the cable, pressed `a`, waited for the prompt, then connected the cable and pressed ENTER.

**Lesson.** The mechanism is change detection, not state detection. The prompt is a genuine instruction about sequence, not a formality.

---

## pfSense LAN colliding with the home network

**Symptom.** Caught before it caused a failure. The console header showed `LAN (lan) -> igc1 -> v4: 192.168.1.1/24`.

**Diagnosis.** pfSense defaults its LAN to `192.168.1.1/24`, which is the same subnet as the home network on the WAN side. With the same subnet on both interfaces, pfSense cannot determine whether traffic for `192.168.1.x` should route upstream or stay local.

**Fix.** Moved the LAN to `10.10.10.1/24` from console option 2.

**Lesson.** Check the default LAN subnet against the upstream network before connecting anything. This is cheap to prevent and expensive to diagnose after the fact.

---

## DNS resolver ignoring LAN clients

**Symptom.** pfSense itself resolved and browsed normally. Diagnostics → DNS Lookup returned correct results. Clients on the LAN timed out on every query:

```
$ nslookup google.com 10.10.10.1
;; communications error to 10.10.10.1#53: timed out
```

Meanwhile `nslookup google.com 8.8.8.8` from the same client worked, and `ping 8.8.8.8` showed 0% loss — so routing and NAT were fine.

**Diagnosis.** The DNS Resolver's *Network Interfaces* setting determines which interfaces it listens on. Left at localhost only, it answers queries from pfSense and silently drops everything else. Timeouts rather than refusals are the signature: the service isn't rejecting queries, it isn't hearing them.

**Fix.** Set Network Interfaces to `All`. Note that selecting only `LAN` is rejected — pfSense uses its own resolver, so `Localhost` must always be included. Query forwarding was also enabled, since full recursion is unreliable behind multiple layers of NAT.

**Lesson.** A service working for the host it runs on says nothing about whether it works for the network. These are separate claims and need separate tests.

---

## Bridge subnet collision on the Proxmox host

**Symptom.** Caught before causing a failure, while reviewing `/etc/network/interfaces`.

**Diagnosis.** `vmbr1` was configured with `10.10.10.1/24`, left over from running pfSense as a VM. That address is now the physical firewall's LAN. The host would have had two routes to the same subnet — one through `vmbr0` toward the firewall, one to a local bridge with no physical port. The kernel prefers the local route, so traffic would have disappeared into a dead bridge.

**Fix.** Moved `vmbr1` to `10.10.99.0/24`, keeping it available as an isolated network with no internet access.

**Lesson.** Leftover configuration from abandoned approaches doesn't announce itself. Read the whole file before editing part of it.

---

## Forgotten Proxmox root password

**Symptom.** No access to the host, physically or over the network.

**Fix.** Reset from the console via single-user mode. At the GRUB menu, press `e`, append `init=/bin/bash` to the `linux` line, then `Ctrl+X`:

```
mount -o remount,rw /
passwd root
reboot -f
```

**Lesson.** Proxmox has two authentication realms. `Linux PAM standard authentication` uses the system root account; the Proxmox VE realm holds separately created users. Entering a correct password against the wrong realm looks identical to entering the wrong password.

---

## Firewall hardware failure

**Symptom.** The pfSense box stopped responding. Power LED off, Ethernet port LEDs still lit, case cold instead of its usual warm. Clients lost DHCP, DNS and routing; Proxmox still answered at its static address, confirming the rest of the network was intact.

**Diagnosis.** Lit Ethernet LEDs on a dead machine are expected — the network PHY draws power independently of the CPU, so it proves current reaches the board but says nothing about the system running. The cold case was the real signal: nothing was executing. A power drain changed nothing. Opening the case produced a faint burnt-plastic smell with no visible damage — no swollen capacitors, no scorch marks.

The cause was thermal. The unit was fanless with no heatsink touching the CPU; cooling was passive airflow through vents in the lid alone. It had run hot since day one. For a device running 24/7 that was never adequate, and the failure was cumulative rather than sudden.

**Recovered.** DDR3L SODIMM and mSATA SSD both appear undamaged. The SSD still holds `/cf/conf/config.xml` — the full pfSense configuration, readable with a USB adapter.

**Cost.** No configuration backup had been taken. The XML export from Diagnostics → Backup & Restore takes seconds and would have made this an inconvenience rather than a rebuild. The config is recoverable from the SSD, but that is luck, not planning.

**Lesson.** Hardware running persistently hot is a fault in progress, not a quirk to tolerate — the warning was there for months and was read as a characteristic of the device. Passive cooling with no heatsink contact is not sufficient for continuous operation; vents in a lid are not cooling. Take the backup as soon as a working state is reached, not when it feels significant enough to warrant one. 


---
## Recurring patterns

**Test three layers separately.** Gateway, external IP, domain name. Each isolates a different part of the chain, and which one fails narrows the cause immediately:

```
ping -c 4 <gateway>
ping -c 4 8.8.8.8
ping -c 4 google.com
```

**IP works, name doesn't — always DNS, never routing.** This came up three times in one session.

**Working locally does not mean working for clients.** The DNS resolver and the WiFi bridge both failed this way.

**Renew the client lease after any infrastructure change.** Clients cache DHCP and DNS settings and won't pick up changes on their own.

**Reduce variables before diagnosing.** Turn off the WiFi when testing a wired path. Connect one device at a time. Most of the confusion in this build came from having two working paths and not knowing which one was carrying traffic.
