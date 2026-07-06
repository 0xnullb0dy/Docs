# Homelab Network Build — Full Documentation

> [!info] Purpose
> This document covers the complete build of a UniFi-based homelab network: VLAN design, the UCG-Ultra setup, Zone-Based Firewall configuration, server migration, and every piece of real troubleshooting encountered along the way. Built for daily use **and** as a cybersecurity practice environment.

---

## 1. Project Goals

- Replace a TP-Link consumer router with a UniFi Cloud Gateway Ultra (UCG-Ultra) as the single firewall/routing boundary
- Segment the network with VLANs for security isolation (servers, gaming/entertainment, IoT, management)
- Preserve gaming performance and daily usability
- Build toward a future dedicated cybersecurity lab VLAN
- Document everything for future reference and as a personal runbook

---

## 2. Final Network Topology

```
[ATT ONT]
     |
[ATT BGW302-505]  (IP Passthrough mode, admin UI at 192.168.1.254)
     |
     | WAN — Public IP passed through directly
     |
[UCG-Ultra]
     |
     |── LAN 1 (Trunk, all VLANs) ──→ [Netgear Managed Switch — Desk]
     |                                       |
     |                                       |── VLAN 1  (Untagged) → PC, Laptop
     |                                       |── VLAN 40 (Untagged) → Minimox
     |                                       |── VLAN 20 (Untagged) → AX21 (WAP)
     |                                       |── VLAN 99 (Untagged) → Raspberry Pi
     |                                       └── Uplink (Tagged: 1,20,40,60,99)
     |
     |── LAN 2 (VLAN 40 Access) ──→ Bigmox (Proxmox, direct connection)
     |
     |── LAN 3 (VLAN 60 Access) ──→ [Living Room Switch — Unmanaged]
     |                                       |── PS5
     |                                       |── Nintendo Switch
     |                                       └── Apple TV
     |
     └── LAN 4 — Reserved for future Cyber Lab VLAN (50)

Smart TV: permanently disconnected from the network (HDMI-only display)
```

### Hardware Inventory

| Device | Role | Connection |
|---|---|---|
| UCG-Ultra | Gateway / Firewall / DHCP / DNS | WAN → AT&T BGW302-505 |
| AT&T BGW302-505 | ONT modem/gateway, IP Passthrough mode | WAN |
| Netgear Smart Switch | Managed switch (Advanced 802.1Q VLAN mode) | LAN 1 (trunk) |
| TP-Link Archer AX21 | Wireless Access Point (AP mode only) | Netgear switch, VLAN 20 port |
| Bigmox | Proxmox VE host | UCG-Ultra LAN 2 (direct) |
| Minimox | Proxmox VE host | Netgear switch, VLAN 40 port |
| Raspberry Pi | Tailscale subnet router (remote access) | Netgear switch, VLAN 99 port |
| Living Room Switch | Unmanaged switch | UCG-Ultra LAN 3 |
| PS5 / Switch / Apple TV | Gaming & entertainment | Living Room Switch |
| Smart TV | Display only | Disconnected from network entirely |

---

## 3. IP Addressing & VLAN Scheme

Standard used: `10.0.0.0/8` private range, with the VLAN ID mirrored in the third octet for readability (`10.[VLAN].[VLAN].0/24`). This avoids the overused `192.168.x.x` range, which conflicts with most consumer routers, hotel/public networks, and VPN split-tunnel scenarios.

| VLAN ID | Name | Subnet | Gateway | Zone | Purpose |
|---|---|---|---|---|---|
| 1 (Default) | Main LAN | 10.10.10.0/24 | 10.10.10.1 | Internal | PC, Laptop — trusted daily drivers |
| 20 | IoT | 10.20.20.0/24 | 10.20.20.1 | IoT (custom zone) | Alexa, coffee maker, iPhone, iPad (via AX21) |
| 40 | Servers | 10.40.40.0/24 | 10.40.40.1 | Internal | Bigmox, Minimox (Proxmox hosts) |
| 60 | Gaming | 10.60.60.0/24 | 10.60.60.1 | Internal | PS5, Switch, Apple TV |
| 99 | Management | 10.99.99.0/24 | 10.99.99.1 | Internal | Switch mgmt IP, Raspberry Pi (Tailscale) |
| 30 *(reserved)* | Guest | 10.30.30.0/24 | 10.30.30.1 | Hotspot | Future Guest SSID |
| 50 *(reserved)* | Cyber Lab | 10.50.50.0/24 | 10.50.50.1 | Internal | Future isolated pentest lab |

> [!note] VLAN 1 is technically the "Default" network
> UniFi will not allow the Default network's VLAN ID to be changed away from 1. It was renamed and re-addressed to `10.10.10.0/24` but is still VLAN 1 (native/untagged) under the hood — this matters when configuring trunk ports on third-party switches (see Netgear section below).

**Domain name used network-wide:** `home.arpa` (per [RFC 8375](https://datatracker.ietf.org/doc/html/rfc8375), the IETF-reserved domain specifically for home networks — avoids conflicts with mDNS's reserved `.local` domain).

---

## 4. DHCP Reservations

| Device                | MAC-based Reservation | IP          | VLAN |
| --------------------- | --------------------- | ----------- | ---- |
| Netgear Switch (mgmt) | —                     | 10.99.99.11 | 99   |
| Raspberry Pi          | Yes                   | 10.99.99.3  | 99   |
| Minimox               | Yes                   | 10.40.40.10 | 40   |
| Bigmox                | Yes                   | 10.40.40.11 | 40   |
| iPhone                | Yes                   | 10.20.20.10 | 20   |
| iPad                  | Yes                   | 10.20.20.11 | 20   |

> [!warning] Mini/Big swap
> Originally planned as Bigmox=.10 / Minimox=.11, but was deliberately swapped during setup to Minimox=.10 / Bigmox=.11. Both the DHCP reservation **and** the Local DNS Record must match — a mismatch between the two was the cause of one of the troubleshooting issues below.

### Local DNS Records (manual, in addition to dynamic DHCP-hostname registration)

| Hostname | IP |
|---|---|
| `minimox.home.arpa` | 10.40.40.10 |
| `bigmox.home.arpa` | 10.40.40.11 |
| `pi.home.arpa` | 10.99.99.3 |

---

## 5. UCG-Ultra Initial Setup

### Pre-configuration (offline/local)

1. Connect laptop directly to a UCG-Ultra **LAN** port (not the WAN port — identifiable by the globe icon, Port 1 on standard UCG-Ultra units)
2. Browse to `https://192.168.1.1` to reach the setup wizard
3. **Critical first step:** change the Default network from `192.168.1.0/24` to `10.10.10.0/24` before anything else, since AT&T's gateway also lives on `192.168.1.x`

```
Gateway IP/Subnet field must use the device's own usable
host address with CIDR — not the network address:

Correct:    10.10.10.1/24
Incorrect:  10.10.10.0/24  (network address — not assignable)
```

4. Domain Name field, set on **every** network created: `home.arpa`

### Temporary WAN connection for initial setup

To complete UI account setup/firmware updates, the UCG-Ultra's WAN was temporarily connected to the AT&T gateway in normal routing mode (double-NAT, harmless and temporary) before switching to IP Passthrough.

---

## 6. Creating the VLANs (UniFi Network App)

```
Settings → Networks → Create New Virtual Network
Turn OFF "Auto-Scale Network" for manual control over subnet/VLAN ID
```

Networks created: **Servers (40)**, **Gaming (60)**, **Management (99)**, **IoT (20)** — each with Purpose set to Corporate/Standard (Guest VLAN 30 will use the "Guest" purpose specifically when built, which auto-applies client isolation).

---

## 7. Port Manager (UCG-Ultra physical ports)

| Port | Mode | Native/Untagged VLAN | Tagged VLANs |
|---|---|---|---|
| LAN 1 | Trunk | Default (VLAN 1) | 20, 40, 60, 99 |
| LAN 2 | Access | VLAN 40 (Servers) | — |
| LAN 3 | Access | VLAN 60 (Gaming) | — |
| LAN 4 | Reserved | — | (future VLAN 50) |

> [!tip] "Trunk" isn't a literal toggle in UniFi
> It's achieved by setting **Native VLAN** + **Tagged VLAN Management: All** (or a specific list) together. There's no single "Trunk" dropdown — the combination of these two fields *is* the trunk configuration.

---

## 8. Zone-Based Firewall (UniFi Network 10.x)

UniFi replaced the old per-rule firewall model with **Zone-Based Firewall (ZBF)** starting in Network 9.0. Networks are grouped into zones; a zone cannot reach any other zone except the Gateway/Internet by default. **Important nuance discovered during setup:** networks that remain together in the *same* zone (e.g. "Internal") can freely reach each other by default — zone separation only blocks *cross-zone* traffic, not traffic between networks inside the same zone.

### Zones

- **Internal** (default zone): VLAN 1 (Main), 40 (Servers), 60 (Gaming), 99 (Management)
- **IoT** (custom zone): VLAN 20

### Policies Built

| Policy Name | Source | Destination | Action | Connection State |
|---|---|---|---|---|
| *(automatic)* | IoT zone | Internal zone | Block | All |
| iPhone/iPad exception | IoT (10.20.20.10/.11) | Internal (VLAN 60, 40) | Allow | All |
| Block Gaming to Internal | VLAN 60 | VLAN 1, 40, 99 | Block | All |
| Block Servers to Main | VLAN 40 | VLAN 1 (Default) | Block | **New** (see note below) |
| Block Servers to Management | VLAN 40 | VLAN 99 | Block | All |

> [!danger] The Connection State lesson (most important firewall concept in this build)
> A firewall doesn't just match source/destination IP — it tracks whether a packet is **New** (the first packet of a connection) or **Established** (a reply/continuation of a connection already permitted in the other direction).
>
> **"Block Servers to Main" originally had Connection State = All.** This meant it blocked not only new connections *from* Servers *to* Main, but also legitimate **replies** from Minimox to anything the Main LAN had itself initiated (e.g., a ping or Proxmox web request from the desktop). The fix: set Connection State to **New** only. This still blocks Minimox from *initiating* anything toward Main, but lets it freely reply to things Main reaches out for — exactly the one-directional trust relationship intended.
>
> **Gotcha encountered:** the first attempt to change this setting did not actually save. Always close and reopen a policy after editing to confirm the change actually persisted before assuming it's fixed.

### Multicast DNS (mDNS) Proxy

Set to **Auto** at the gateway level — automatically reflects AirPlay/AirPrint/Chromecast/HomeKit discovery traffic across all VLANs. This is separate from firewall *traffic* rules — discovery (seeing a device in a list) and actual data transfer both need to be allowed independently. Enabled by default on new networks.

---

## 9. Netgear Managed Switch Configuration

### VLAN Mode

The switch initially appeared limited to VLAN IDs 1–8. This was **not a hardware limit** — it was the switch sitting in **Port-based VLAN mode** (a simple, non-tagging mode limited to a small number of groups) rather than **802.1Q VLAN mode** (real tagged VLANs, ID range 1–4094). Switched to **Advanced 802.1Q VLAN** mode specifically (not "Basic," which doesn't support port tagging needed for a trunk link).

### Per-VLAN Table Entries

| VLAN | Ports (Untagged/Access) | PVID | Uplink Port 8 |
|---|---|---|---|
| 1 (Main) | Ports 1, 4 (PC, Laptop) | 1 | **Untagged** *(matches UCG native VLAN)* |
| 20 (IoT) | Port 5 (AX21) | 20 | Tagged |
| 40 (Servers) | Port 2 (Minimox) | 40 | Tagged |
| 99 (Management) | Port 3 (Raspberry Pi) | 99 | Tagged |
| 60 (Gaming) | *(unused on this switch)* | — | Tagged *(unused capacity)* |

> [!warning] Three critical mistakes caught before they caused outages
> 1. **VLAN 99 was initially left off the uplink's tagged list** — this would have made the switch's own management interface unreachable.
> 2. **The Main LAN entry was initially tagged on the uplink instead of untagged.** Since UniFi's Default network is genuinely VLAN 1 (untagged/native on the UCG-Ultra's trunk), tagging it as "VLAN 10" on the switch would never have matched anything in UniFi — Main LAN devices would have had zero connectivity. The switch's "Main LAN" entry must use **VLAN ID 1**, untagged on the uplink, not a literal "10."
> 3. Forgot to configure the management VLAN section of the configuration. After configuring that the switch was accessible across the network per firewall rules.

### Switch's Own Management IP

```
System/IP Settings (separate from the per-port VLAN table):
   IP Address: 10.99.99.11/24
   Gateway:    10.99.99.1   (must be in the same subnet as the IP)
```

### Core VLAN Concept (worth remembering)

The **switch port's VLAN assignment** is the actual security boundary — not the IP address a device happens to have. A device's IP only works if it matches the subnet of whatever VLAN the **switch port** has placed it on. Manually assigning an IP from a different VLAN's range to a device plugged into the wrong port does not grant access to that VLAN — it just breaks the device (no reachable gateway, no connectivity). The router only ever sees traffic *after* the switch has already decided which VLAN it belongs to.

---

## 10. AT&T BGW302-505 — IP Passthrough

```
Admin UI: https://192.168.1.254  (note: NOT .1.1 on this model)
Firewall → IP Passthrough
Allocation Mode: Passthrough
Passthrough Mode: DHCPS-fixed
Passthrough MAC: [UCG-Ultra's WAN MAC address]
```

> [!note] One public IP, one Passthrough target — always
> AT&T (like virtually all residential ISPs) only ever provides a single public IP. IP Passthrough hands that one IP to exactly one designated MAC address. There is no way to passthrough to two devices simultaneously regardless of plan tier. The Archer was the original Passthrough target (left over from before the UCG-Ultra); this had to be explicitly repointed to the UCG-Ultra's WAN MAC during cutover.

**Accessing the AT&T admin page once your laptop is on the UCG-Ultra's LAN:** you cannot reach `192.168.1.254` directly, since it's on a separate interface from your LAN. Either temporarily plug directly into the AT&T gateway, or use a phone on the AT&T WiFi if still broadcasting.

---

## 11. Server Migration — Bigmox & Minimox (Proxmox)

### Switching from static IP to DHCP + Reservation

Original static config (commented out, left as reference):

```bash
auto lo
iface lo inet loopback
iface eno1 inet manual
auto vmbr0
iface vmbr0 inet dhcp
        bridge-ports eno1
        bridge-stp off
        bridge-fd 0
iface enp4s0 inet manual
iface wlp2s0 inet manual
source /etc/network/interfaces.d/*
```

> [!danger] Bug caught before it caused an outage
> When commenting out the static `address`/`gateway` lines, **`bridge-ports`, `bridge-stp`, and `bridge-fd` were accidentally commented out too.** Without `bridge-ports`, the virtual bridge (`vmbr0`) has no link to the physical NIC at all — DHCP requests never reach the network, and the host gets no IP whatsoever (not even a fallback). Only `address` and `gateway` should ever be removed; the bridge structure lines must stay active regardless of static vs. DHCP.

### Missing `dhclient` (isc-dhcp-client)

Recent Debian releases stopped shipping `isc-dhcp-client` by default, but Proxmox's `ifupdown2` still depends on it to bring up DHCP interfaces. Without it, DHCP silently never even attempts a request.

```bash
apt update
apt install isc-dhcp-client
ifreload -a
```

(Bigmox happened to already have it installed — Minimox did not. Worth checking on any Proxmox host before assuming DHCP will "just work.")

### Chicken-and-egg: installing a package with no internet

Since DHCP wasn't working yet, `apt update` had nothing to reach. Fixed with a temporary, non-persistent IP assignment:

```bash
ip addr add 10.40.40.50/24 dev vmbr0
ip route add default via 10.40.40.1
# (run apt update / apt install isc-dhcp-client here)
ip addr del 10.40.40.50/24 dev vmbr0
ip route del default via 10.40.40.1
ifreload -a
```

### Forcing a fresh DHCP lease (if needed)

```bash
dhclient -r vmbr0      # release current lease
dhclient vmbr0          # request a new one
ip a show vmbr0         # verify
```

### Confirming DHCP vs. static in `ip a` output

```
inet 10.40.40.10/24 ... dynamic noprefixroute eth0
   valid_lft 86234sec preferred_lft 86234sec
```

Look for the word **dynamic** and a real countdown number for `valid_lft`/`preferred_lft`. `forever` with no `dynamic` flag normally indicates a static address — **but not always** (see note below).

> [!note] Edge case: infinite DHCP leases can look static
> On the Raspberry Pi, `ip a` showed `forever`/no `dynamic` flag even though `journalctl -u NetworkManager -b | grep eth0` proved a real DHCP transaction had succeeded. Likely explanation: the upstream DHCP server (the Archer) was issuing an infinite lease time for this MAC (a common behavior for router-side "reservations"), which some systems display identically to a manually-static address. **When in doubt, check the actual NetworkManager/dhclient logs, not just `ip a`.**

---

## 12. Raspberry Pi — Tailscale Subnet Router

### Placement Decision

Initially considered VLAN 40 (Servers), but rejected: the "Block Servers to Main" rule would have prevented the Pi from reaching the Main LAN, defeating its purpose as a general remote-access gateway. **Placed on VLAN 99 (Management) instead** — inherits strong inbound protection (only reachable from VLAN 10) while retaining outbound reach to VLAN 10/40/60 (the default Internal-zone behavior), which is exactly what a "way in from anywhere" device needs.

### Switching to DHCP (NetworkManager, Raspberry Pi OS Bookworm+)

```bash
# Find the active connection name
nmcli con show

# Switch to DHCP, clear static fields
nmcli con mod "Wired connection 1" ipv4.method auto
nmcli con mod "Wired connection 1" ipv4.addresses ""
nmcli con mod "Wired connection 1" ipv4.gateway ""
nmcli con mod "Wired connection 1" ipv4.dns ""

# Apply
nmcli con down "Wired connection 1"
nmcli con up "Wired connection 1"

# Verify
ip a show eth0
nmcli device show eth0   # look for DHCP4.OPTION fields as definitive proof
```

### Tailscale Subnet Route Advertisement

```bash
# On the Pi, enable IP forwarding (/etc/sysctl.conf):
net.ipv4.ip_forward=1

# Advertise the local subnet to the tailnet:
tailscale up --advertise-routes=10.99.99.0/24
```

> [!note] Tailscale doesn't bypass the UniFi firewall
> The advertised route gets traffic to the Pi's "front door," but whatever the Pi can reach locally is still entirely governed by the same Zone-Based Firewall rules covering VLAN 99 — Tailscale itself adds no additional access beyond what the local network already allows.

### Side effect discovered during DNS troubleshooting

Tailscale's MagicDNS (`100.100.100.100`) had been configured at some point (abandoned mid-setup) to handle `home.arpa` lookups, overwriting `/etc/resolv.conf` on **Minimox** as well (Minimox also runs Tailscale). This only affects a machine's own outbound DNS lookups — it has no bearing on whether *other* devices can resolve that machine's hostname. Important distinction that cost some debugging time (see troubleshooting log).

---

## 13. Troubleshooting Log (Chronological)

A real record of every issue actually hit during this build, in the order encountered — useful both as a personal reference and as an example of methodical network debugging.

### Issue 1 — Double NAT after connecting UCG-Ultra to AT&T
**Symptom:** UCG-Ultra WAN showed a private IP instead of the public one.
**Cause:** UCG-Ultra was wired behind the TP-Link Archer rather than directly to the AT&T gateway.
**Fix:** Re-cabled UCG-Ultra's WAN port directly to an AT&T LAN port.

### Issue 2 — Still double-NATed after direct connection
**Cause:** AT&T's IP Passthrough was still pointed at the Archer's MAC address (leftover from before the UCG-Ultra existed) — Passthrough only ever targets one specific MAC.
**Fix:** Repointed Passthrough to the UCG-Ultra's WAN MAC in the AT&T admin UI (`192.168.1.254`).

### Issue 3 — UniFi wouldn't accept `10.10.10.0/24` as the LAN subnet
**Cause:** Entered the network address instead of a usable host address.
**Fix:** Used `10.10.10.1/24` (the gateway's own address) instead of `10.10.10.0/24`.

### Issue 4 — TP-Link AX21 couldn't provide separate IoT/Guest wireless networks
**Cause:** Consumer-grade router; confirmed via TP-Link's own support forum that the AX21 doesn't support per-SSID VLAN tagging — that capability is reserved for TP-Link's business-class Omada APs.
**Resolution:** Accepted as a known limitation until a UniFi AP is purchased; all AX21 wireless clients placed on a single VLAN (20/IoT).

### Issue 5 — Netgear switch "only allows VLAN IDs 1–8"
**Cause:** Switch was in **Port-based VLAN mode**, not true 802.1Q tagging mode.
**Fix:** Switched to **Advanced 802.1Q VLAN** mode (Basic 802.1Q doesn't support per-port tagging needed for a trunk either).

### Issue 6 — VLAN 99 missing from the switch uplink's tagged list
**Cause:** Overlooked during initial VLAN table setup.
**Fix:** Added VLAN 99 as Tagged on the uplink port — otherwise the switch's own management interface would have been unreachable.

### Issue 7 — Main LAN entry tagged on the uplink instead of untagged
**Cause:** Misunderstanding that "VLAN 10" (the colloquial name) is actually VLAN 1 (UniFi's Default network, which can never be reassigned). Since the UCG-Ultra's trunk treats this VLAN as the **untagged native VLAN**, tagging it as a literal "10" on the switch meant it would never have matched anything in UniFi.
**Fix:** Set the Main LAN entry to VLAN ID 1, **untagged** on the uplink port specifically (the one exception among all the VLANs on that trunk).

### Issue 8 — Minimox: bridge-ports accidentally removed
**Cause:** Commenting out the old static IP block also commented out `bridge-ports`/`bridge-stp`/`bridge-fd`, severing the bridge's link to the physical NIC.
**Fix:** Restored those three lines (uncommented), left only `address`/`gateway` removed.

### Issue 9 — Minimox: DHCP request never even attempted
**Cause:** `isc-dhcp-client` (`dhclient`) not installed by default on this Debian/Proxmox version — `ifupdown2` has no DHCP client to call.
**Fix:** `apt install isc-dhcp-client` (required a temporary manual IP just to reach the internet for the install — see Section 11).

### Issue 10 — Ping to Minimox worked one direction, failed for the reply
**Symptom:** Could ping/curl Minimox's gateway and other VLANs fine, but Minimox itself was unreachable from Main LAN.
**Cause:** "Block Servers to Main" firewall policy had **Connection State = All**, which blocked legitimate *reply* traffic (Minimox responding to a connection Main itself initiated), not just new connection attempts from Servers.
**Fix:** Changed Connection State to **New** only — and confirmed the change actually saved by closing and reopening the policy (the first edit attempt silently failed to persist).

### Issue 11 — DNS resolved `minimox.home.arpa` to the wrong IP
**Cause:** Local DNS Record pointed to Bigmox's address instead of Minimox's, after the deliberate Mini=.10/Big=.11 swap wasn't fully propagated to the DNS record.
**Fix:** Corrected the Local DNS Record to match the actual DHCP reservation.

### Issue 12 — DNS testing was confusing because it was run on the wrong machine
**Cause:** Ran `nslookup` directly on Minimox, which uses Tailscale's MagicDNS (`100.100.100.100`) for its own outbound queries — irrelevant to whether *other* devices can find Minimox by name.
**Lesson:** Always test hostname resolution from the device that actually needs to do the looking-up (the desktop), not from the target server itself.

### Issue 13 — Desktop couldn't resolve `home.arpa` names at all
**Cause:** Desktop's DNS server was statically/manually set to `192.168.0.7` — a leftover from the old network — even though its IP and gateway had already correctly migrated to the new `10.10.10.x` network via DHCP.
**Fix:** Switched the network adapter's DNS assignment from Manual back to Automatic (DHCP), then `ipconfig /release` + `ipconfig /renew`.

### Issue 14 — "Can't reach Minimox in the browser" despite everything else working
**Cause:** Forgot to include the port number. Proxmox's web UI runs on `:8006`, not the default `443`.
**Fix:** Used the full URL: `https://minimox.home.arpa:8006`.

---

## 14. Roadmap / Future Work

- [ ] Purchase a UniFi-brand AP to replace the AX21, unlocking proper multi-SSID VLAN tagging (Main/IoT/Guest as separate wireless networks)
- [ ] Activate VLAN 30 (Guest) once the UniFi AP is in place — set Purpose to **Guest** specifically for automatic client isolation
- [ ] Build out VLAN 50 (Cyber Lab) on UCG-Ultra LAN 4 for isolated attack/defense practice
- [ ] Consider tightening VLAN 1 (Main LAN) trunk hardening — Native VLAN: None instead of Default, to fully eliminate VLAN-hopping attack surface on LAN 1 (optional, more relevant to lab/practice than home security necessity)
- [x] Apply the same DHCP/dhclient check to Bigmox proactively before it becomes an issue (confirmed already fine, but worth re-checking after any future Proxmox upgrade)

---

## 15. Command Reference (All Commands Used)

```bash
# --- Network diagnostics ---
ip a show eth0
ip a show vmbr0
ip link show
nmcli con show
nmcli con show --active
nmcli device status
nmcli device show eth0
nmcli con show "Wired connection 1" | grep ipv4
ping <ip>
nslookup <hostname>
curl http://<host>:8006 -k -m 5

# --- NetworkManager (Raspberry Pi) ---
nmcli con mod "Wired connection 1" ipv4.method auto
nmcli con mod "Wired connection 1" ipv4.addresses ""
nmcli con mod "Wired connection 1" ipv4.gateway ""
nmcli con mod "Wired connection 1" ipv4.dns ""
nmcli con down "Wired connection 1"
nmcli con up "Wired connection 1"

# --- Proxmox / Debian (ifupdown2 + dhclient) ---
cat /etc/network/interfaces
nano /etc/network/interfaces
systemctl restart networking
ifreload -a
dhclient -r vmbr0
dhclient vmbr0
apt update
apt install isc-dhcp-client
which dhclient

# --- Temporary manual IP (to bootstrap internet access) ---
ip addr add 10.40.40.50/24 dev vmbr0
ip route add default via 10.40.40.1
ip addr del 10.40.40.50/24 dev vmbr0
ip route del default via 10.40.40.1

# --- Logs ---
journalctl -u NetworkManager -n 50 --no-pager
journalctl -u NetworkManager -b --no-pager | grep -i eth0
systemctl status dhcpcd
systemctl status NetworkManager

# --- Tailscale (Raspberry Pi) ---
tailscale up --advertise-routes=10.99.99.0/24
cat /etc/resolv.conf

# --- Windows (Desktop) ---
ipconfig /all
ipconfig /release
ipconfig /renew
ipconfig /flushdns
```

---

## 16. Sources / References

**Standards & RFCs**
- RFC 1918 (Private IP Addressing) — https://datatracker.ietf.org/doc/html/rfc1918
- RFC 8375 (home.arpa) — https://datatracker.ietf.org/doc/html/rfc8375
- RFC 6762 (Multicast DNS) — https://datatracker.ietf.org/doc/html/rfc6762

**UniFi / Ubiquiti Official Documentation**
- UCG-Ultra Getting Started — https://help.ui.com/hc/en-us/articles/21427208862487
- UniFi Categories — https://help.ui.com/hc/en-us/categories/200320654-UniFi
- VLAN/Network Segmentation Guide — https://help.ui.com/hc/en-us/articles/360046144234
- Creating Virtual Networks (VLANs) — https://help.ui.com/hc/en-us/articles/9761080275607-Creating-Virtual-Networks-VLANs
- Fixed IP / DHCP Reservations — https://help.ui.com/hc/en-us/articles/9203184064535
- DNS Records and Local Hostnames — https://help.ui.com/hc/en-us/articles/15179064940439-UniFi-DNS-Records-and-Local-Hostnames
- Zone-Based Firewalls in UniFi — https://help.ui.com/hc/en-us/articles/115003173168-Zone-Based-Firewalls-in-UniFi
- UniFi Gateway Multicast DNS — https://help.ui.com/hc/en-us/articles/12648701398807-UniFi-Gateway-Multicast-DNS
- UniFi Updates — https://help.ui.com/hc/en-us/articles/7605005245975-UniFi-Updates
- Ubiquiti Community — Default Network/VLAN 1 — https://community.ui.com/questions/First-Time-Unifi-Setup-Management-VLAN-and-Default-Network-Confusion/a1072717-3693-488b-b4bf-a163961d83de
- Ubiquiti Community — Local DNS resolution — https://community.ui.com/questions/UniFi-USG-local-DNS-not-resolving-local-hostname-correctly/0f342dd2-7a1a-497e-8cc0-169952ac351e

**Third-Party UniFi Guides**
- Crosstalk Solutions — https://crosstalksolutions.com
- Crosstalk Solutions — UniFi Home Network Setup — https://crosstalksolutions.com/the-only-unifi-home-network-setup-youll-ever-need/
- LazyAdmin — VLAN Configuration — https://lazyadmin.nl/home-network/unifi-vlan-configuration/
- LazyAdmin — Zone-Based Firewall — https://lazyadmin.nl/home-network/unifi-zone-based-firewall/
- KJJ Tech — Zone-Based Firewall Practical Guide — https://kjjtech.com/blog/unifi-zone-based-firewall.html
- InsideWire — UniFi Network 2026 Setup Guide — https://insidewire.co.uk/blog/unifi-network-2026-setup/
- Flavor365 — UniFi Guest Network Setup — https://flavor365.com/unifi-guest-network-setup-a-step-by-step-tutorial/
- Ram's Tech Blog — Fixing Local Name Resolution on UniFi — https://nramkumar.org/tech/blog/2026/04/22/fixing-local-name-resolution-issues-in-unifi-gateways/
- techradar.info — UniFi Firmware/Software Guide 2026 — https://techradar.info/how-do-i-update-unifi-the-complete-2026-firmware-software-guide/
- Securing The Universe — UniFi InnerSpace — https://securingtheuniverse.com/2025/11/10/unifi-innerspace-managing-floorplans-and-network-coverage/

**Networking Security**
- Cisco Press — VLAN Hopping Attacks — https://www.ciscopress.com/articles/article.asp?p=2181830

**TP-Link**
- AP Mode Configuration FAQ — https://www.tp-link.com/us/support/faq/510/
- Community — AX21 VLAN limitation — https://community.tp-link.com/en/home/forum/topic/262666
- Community — Multi-SSID VLAN tagging — https://community.tp-link.com/us/home/forum/topic/195878

**Netgear**
- GS108T VLAN capacity (port-based vs. tag-based) — http://documentation.netgear.com/gs108t/enu/202-10337-01/GS108T_UM-06-11.html
- 802.1Q VLAN configuration (GS908E) — https://kb.netgear.com/000051459/How-do-I-create-and-manage-802-1Q-VLANs-on-my-NETGEAR-GS908E-switch
- GS305EP/GS308EP Manual — Basic vs. Advanced 802.1Q — https://www.downloads.netgear.com/files/GDC/GS305EP/GS305EP-EPP_GS308EP-EPP_UM_EN.pdf

**Proxmox**
- Network Configuration Wiki — https://pve.proxmox.com/wiki/Network_Configuration
- Forum — dhclient missing by default — https://forum.proxmox.com/threads/proxmox-9-0-4-doesnt-get-ip-by-dhcp.169721/

**Raspberry Pi**
- Network Manager setup — https://pimylifeup.com/raspberry-pi-network-manager/

**AT&T**
- IP Passthrough Support Article — https://www.att.com/support/article/u-verse-high-speed-internet/KAN1009812

---

*Document generated as a complete build record — June 2026.*
