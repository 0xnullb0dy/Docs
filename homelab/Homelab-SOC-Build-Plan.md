# Homelab SOC Build Plan — SIEM + EDR + Vulnerability Mgmt + Automation

> **Purpose**
> This is the active planning runbook and progress tracker for the homelab SOC layer
> on top of the existing UniFi/VLAN network. It covers Splunk (primary SIEM), Wazuh
> (EDR-only role — manager + agent, no indexer/dashboard), OpenVAS/Greenbone
> (vulnerability scanning), Ansible + Semaphore (automation/config management), and
> — planned next — Splunk SOAR (orchestration). All running on-prem with zero cloud
> dependencies.
>
> **Pivot note (July 2026):** Wazuh was originally the centerpiece of this build and
> a huge amount of real troubleshooting happened getting it running (see the
> Troubleshooting Log — those issues are kept, marked **deprecated/historical**, not
> deleted). Wazuh was ultimately dropped in favor of Splunk as the primary SIEM: Splunk
> and SPL appear far more frequently on SOC analyst / security engineer job
> descriptions, and SPL is a directly testable interview skill in a way Wazuh's query
> language generally isn't. The Wazuh VM itself wasn't scrapped — it was repurposed
> into the new Splunk VM (same IP, same host, new OS role).
>
> **Second pivot note (July 2026) — EDR layer:** OpenEDR (both the jymcheong fork and
> ComodoSecurity's original project) was dropped from the EDR role — see Phase 3 for
> the full story. Wazuh is back in the stack, but narrowly and deliberately: manager +
> agent only, no indexer, no dashboard. It functions purely as an EDR engine (file
> integrity monitoring, rootkit detection, active response) whose alerts forward into
> Splunk the same way Sysmon and Auditd do. This does not reopen the SIEM decision
> above — Splunk remains the only dashboard/query surface in the whole stack.
>
> **Cross-reference:** This document is a direct continuation of
> `Homelab-Network-Documentation.md` (VLANs, hostnames, firewall rules) and
> `Homelab-Server-VM-Specs.md` (full VM inventory, host hardware, IP allocations —
> keep that file in sync with any VM changes made here). Read both if anything here
> seems out of context.

---

## Quick Reference — Existing Infrastructure

Before touching anything, know what's already in place. All of this is from the
network doc and must be kept consistent throughout this build.

### VLAN Map (relevant VLANs only)

```
VLAN  | Name       | Subnet          | Gateway      | Zone     | Key Devices
------|------------|-----------------|--------------|----------|-----------------------------
1     | Main LAN   | 10.10.10.0/24   | 10.10.10.1   | Internal | Desktop (PC), Laptop
20    | IoT        | 10.20.20.0/24   | 10.20.20.1   | IoT      | iPhone, iPad, Alexa
40    | Servers    | 10.40.40.0/24   | 10.40.40.1   | Internal | Bigmox, Minimox → SOC VMs go HERE
60    | Gaming     | 10.60.60.0/24   | 10.60.60.1   | Internal | PS5, Switch, Apple TV
99    | Management | 10.99.99.0/24   | 10.99.99.1   | Internal | Raspberry Pi, Netgear mgmt
50    | Cyber Lab  | 10.50.50.0/24   | 10.50.50.1   | (future) | RESERVED — not yet built
30    | Guest      | 10.30.30.0/24   | 10.30.30.1   | (future) | RESERVED — not yet built
```

### Existing DHCP Reservations

```
Device          | IP            | VLAN | Host
----------------|---------------|------|--------------------
Minimox         | 10.40.40.10   | 40   | minimox.home.arpa
Bigmox          | 10.40.40.11   | 40   | bigmox.home.arpa
Raspberry Pi    | 10.99.99.3    | 99   | pi.home.arpa
Netgear Switch  | 10.99.99.11   | 99   | (no DNS record)
iPhone          | 10.20.20.10   | 20   | —
iPad            | 10.20.20.11   | 20   | —
```

> **CRITICAL — Mini/Big are SWAPPED from the original plan**
> Minimox = `.10`, Bigmox = `.11`
> Every DHCP reservation AND DNS record must match. Mismatch = wrong host resolves.
> This already caused Issue 11 once. Don't repeat it.

### Existing Firewall Policies (Zone-Based)

```
Policy Name                  | Source              | Dest                    | Action | Conn State | Status
-----------------------------|---------------------|-------------------------|--------|------------|--------
IoT → Internal (auto)        | IoT zone (VLAN 20)  | Internal zone           | Block  | All        | Active
iPhone/iPad exception        | 10.20.20.10/.11     | VLAN 60, 40             | Allow  | All        | Active
Block Gaming to Internal     | VLAN 60             | VLAN 1, 40, 99          | Block  | All        | Active
Block Servers to Main        | VLAN 40             | VLAN 1 (Default)        | Block  | New ✓      | Fixed (Issue 10)
Block Servers to Management  | VLAN 40             | VLAN 99                 | Block  | All ← !!!  | STILL NEEDS FIX
```

> **!! THE ONE THING TO DO BEFORE ANYTHING ELSE !!**
> "Block Servers to Management" is currently set to Connection State = **All**.
> This will break Raspberry Pi ↔ Splunk/Ansible communication in the exact same way
> that Issue 10 broke Minimox ↔ Main LAN. The Pi's Splunk UF/Auditd traffic INITIATES
> to `splunk.home.arpa` (Management → Servers, allowed), but the manager's replies
> travel back Servers → Management and will be dropped.
> Fix: Change Connection State to **New** on this rule before enrolling any endpoints.
> After changing, CLOSE and REOPEN the policy to confirm it actually saved
> (per Issue 10 lesson — the first silent-fail is documented in the network doc).

> **New rules needed once the Ansible VM is built** (place both ABOVE the two block
> rules above so they're evaluated first):
> ```
> Allow Ansible → Management (SSH)
>   Source: 10.40.40.15 only
>   Dest:   10.99.99.0/24 (VLAN 99)
>   Port:   TCP 22
>
> Allow Ansible → Main (WinRM)
>   Source: 10.40.40.15 only
>   Dest:   10.10.10.0/24 (VLAN 1)
>   Ports:  TCP 5985, 5986
> ```
> Scope these to the Ansible VM's single source IP only — do not open Servers → Main
> or Servers → Management broadly, that defeats the purpose of the zone block rules.

### Physical Topology (relevant portion)

```
[UCG-Ultra]
     |
     |── LAN 1 (Trunk) ──→ [Netgear Managed Switch]
     |                            |── VLAN 40 port → Minimox (10.40.40.10)
     |                            |── VLAN 99 port → Raspberry Pi (10.99.99.3)
     |                            └── VLAN 1  port → Desktop, Laptop
     |
     |── LAN 2 (VLAN 40 Access) ──→ Bigmox (10.40.40.11, direct)
     |
     └── LAN 4 (reserved) ──→ Future VLAN 50 (Cyber Lab)
```

---

## Planned IP / DNS Allocations (Current Stack)

Add these to UCG-Ultra (Settings → Networks → VLAN 40 → DHCP Reservations
and Settings → DNS → Local Records), following the same `home.arpa` convention
used throughout the network.

```
Hostname              | IP            | VLAN | VM Host  | Purpose
----------------------|---------------|------|----------|------------------------------------
splunk.home.arpa      | 10.40.40.12   | 40   | Bigmox   | Splunk Enterprise — primary SIEM
                       |               |      |          | (repurposed from the old Wazuh VM)
wazuh-edr.home.arpa    | 10.40.40.13   | 40   | Minimox  | Wazuh manager + agent-only EDR
                       |               |      |          | (rebuilt fresh from the old OpenEDR VM,
                       |               |      |          | July 2026 — no indexer/dashboard)
scanner.home.arpa     | 10.40.40.14   | 40   | Bigmox   | OpenVAS (Greenbone) — vuln scanning
                       |               |      |          | (moved from Minimox, July 2026 — RAM)
ansible.home.arpa     | 10.40.40.15   | 40   | Minimox  | Ansible + Semaphore — automation
                       |               |      |          | (moved from Bigmox, July 2026 — RAM)
soar.home.arpa        | 10.40.40.16   | 40   | Bigmox   | Splunk SOAR CE — planned, not yet built
```

> **IP conflict resolved (July 2026):** These four/five addresses were originally
> going to land on `.14`–`.16`, which collided with the IP block already reserved in
> `Homelab-Server-VM-Specs.md` for the planned IAM build (FreeIPA, Keycloak, IAM
> demo-apps). Decision: the SOC stack keeps `.14`–`.16` since it's the active build;
> the IAM VMs were renumbered to `.17`–`.19` instead. See the VM Specs doc for the
> updated IAM addresses — this doc is the source of truth for `.12`–`.16`.

> **Why split across hosts?**
> Splunk's indexer is the heaviest resource consumer in this stack — stacking
> everything on one Proxmox host risks starving it. Bigmox (direct LAN 2 connection,
> 31.29 GiB RAM) carries Splunk + OpenVAS + the future SOAR instance; Minimox
> (15.34 GiB RAM) carries the Wazuh EDR manager (deliberately light now that there's
> no indexer/dashboard on it) + Ansible/Semaphore. This keeps load balanced and
> means one host going down doesn't kill your entire monitoring/scanning capability.
>
> **Updated (July 2026):** OpenVAS and Ansible swapped hosts — OpenVAS moved to
> Bigmox because it outgrew Minimox's smaller RAM pool once real scan load
> exceeded its original 4 GB allocation; Ansible (light at 2 GB) moved to Minimox
> in its place. See Phase 4's "Why This Moved" note for the full math.

---

## SOC Traffic Flow (Current Stack)

```
                        VLAN 40 — Servers (10.40.40.0/24)
 ┌────────────────────────────────────────────────────────────────────────────┐
 │                                                                            │
 │  ┌─────────────────────┐  ┌────────────────────┐  ┌───────────────────────┐│
 │  │ splunk.home.arpa     │  │ wazuh-edr.home.arpa│ │ scanner.home.arpa     ││
 │  │ 10.40.40.12 (Bigmox) │  │ 10.40.40.13 (Minimox)│ 10.40.40.14 (Bigmox) ││
 │  │ Splunk Enterprise    │  │ Wazuh manager only  │  │ OpenVAS / Greenbone  ││
 │  │ (primary SIEM)       │  │ (no indexer/dash)   │  │ (agentless scanner) ││
 │  └──────────┬───────────┘  └──────────┬──────────┘  └──────────┬───────────┘│
 │             │                         │                        │            │
 │  ┌──────────┴───────────┐            │            (scans every VLAN/IP,    │
 │  │ ansible.home.arpa     │            │             no agent required)     │
 │  │ 10.40.40.15 (Minimox) │            │                                    │
 │  │ Ansible + Semaphore   │            │                                    │
 │  └──────────┬───────────┘            │                                    │
 │             │                         │                                    │
 │  ┌──────────┴───────────┐            │                                    │
 │  │ soar.home.arpa (future)│            │                                    │
 │  │ 10.40.40.16 (Bigmox)  │            │                                    │
 │  │ Splunk SOAR CE        │            │                                    │
 │  └───────────────────────┘            │                                    │
 └──────────────┬────────────────────────┼────────────────────────────────────┘
                │ Splunk UF              │ Wazuh agent (FIM, rootkit
                │ (Sysmon+SwiftOnSecurity │ detection, active response) —
                │  Auditd)               │ Wazuh UF on the manager VM
                │                        │ tails alerts.json into Splunk
 ┌──────────────▼────────────────────────▼──────────────────────────────────┐
 │  VLAN 1 (Main LAN)                       VLAN 99 (Mgmt)   VLAN 20 (IoT)   │
 │  Desktop 10.10.10.x — OS unconfirmed,    Pi 10.99.99.3     iPhone/iPad    │
 │    telemetry + Wazuh agent plan pending  (Splunk UF        (OpenVAS      │
 │  Laptop  10.10.10.x — Fedora Linux        + Auditd)         scan target  │
 │    (Auditd + Splunk UF + Wazuh agent —                        only,     │
 │     Wazuh is cross-platform, unlike OpenEDR)                  no agent) │
 └──────────────────────────────────────────────────────────────────────────┘
```

> **Host swap (July 2026):** `scanner.home.arpa` and `ansible.home.arpa` traded
> physical hosts — OpenVAS moved Minimox → Bigmox (it outgrew Minimox's smaller
> RAM pool once real scan load exceeded its original 4 GB), and Ansible moved
> Bigmox → Minimox to free up the room OpenVAS needed. IPs and DNS names are
> unchanged either way (`vmbr0`/VLAN 40 exists on both hosts) — only the `(Host)`
> label in the diagram above changed. See Phase 4's "Why This Moved" note for the
> full RAM math.

> **Endpoint OS correction (see Troubleshooting Log — Current-Stack Issue 2):** The
> laptop was originally assumed Windows and routed toward Sysmon. It's actually
> running Fedora Linux — confirmed via Splunk's `hostname` field reporting `fedora`.
> It now belongs on the Auditd path. Unlike OpenEDR (Windows-only), Wazuh's agent
> supports Linux too, so the laptop is no longer excluded from the EDR track —
> see Phase 3. Desktop's OS was never independently confirmed either — don't assume
> Windows there without checking first.

> **Firewall note on traffic direction**
> Splunk UF and Wazuh agents always INITIATE toward their backend — the backend
> never reaches out to endpoints. Ansible is the one exception: it initiates OUT to
> endpoints (SSH/WinRM) to run playbooks, which is exactly why it needs its own
> explicit allow rules rather than relying on the passive agent-initiated model the
> rest of the stack uses.

---

## Phase 0 — Network Prep

**Goal:** Everything is in place before a single tool is installed/repurposed.
**Estimated time:** 2–3 hours
**Host:** UCG-Ultra web UI + Proxmox (Bigmox and Minimox)

### Checklist

- [x] **Fix the firewall rule** — UCG-Ultra → Settings → Security → Zone-Based Firewall
      - Find "Block Servers to Management" policy
      - Change Connection State from **All** → **New**
      - Save, close the policy, reopen it, confirm the change persisted
        (silent save failure is a documented gotcha — see Issue 10 in network doc)

- [x] **Add DHCP reservations for the new/repurposed VMs**
      - Settings → Networks → VLAN 40 (Servers) → DHCP → Add Reservation
      - `10.40.40.12` → the existing Wazuh VM's MAC (unchanged — it's being repurposed,
        not rebuilt, so this reservation should already exist)
      - `10.40.40.14` → MAC of new OpenVAS VM's NIC
      - `10.40.40.15` → MAC of new Ansible+Semaphore VM's NIC
      - `10.40.40.16` → reserve when Splunk SOAR VM is actually built (future)

- [x] **Add/update Local DNS records**
      - Settings → DNS → Local Records
      - Rename/repoint `wazuh.home.arpa` → `splunk.home.arpa` → `10.40.40.12`
        (or add `splunk.home.arpa` fresh and remove the old `wazuh.home.arpa` record
        once nothing references it)
      - `scanner.home.arpa` → `10.40.40.14`
      - `ansible.home.arpa` → `10.40.40.15`
      - `soar.home.arpa` → `10.40.40.16` (add when SOAR VM is built)
      - `wazuh-edr.home.arpa` → `10.40.40.13` (renamed from `openedr.home.arpa`,
        July 2026 — see Phase 3; remove the old record once nothing references it)
      - Double-check all records after saving with `nslookup <name>.home.arpa`
        from the desktop (not from the VM itself — see Issue 12 in network doc)

- [x] **Repurpose the existing Wazuh VM into the Splunk VM**
      - OS stays Ubuntu 24.04 LTS — no reinstall needed, just uninstall Wazuh packages and install Splunk Enterprise instead
      - Current specs (8 vCPU / 16 GB / 50 GB, ZFS storage) were sized for Wazuh's indexer + dashboard + Splunk running together. Running Splunk alone is lighter — reasonable to right-size down over time, but there's no urgency; the extra headroom doesn't hurt anything sitting idle.
      - **Open item:** the disk bump (50 GB → 100 GB) is worth doing given Splunk's own index storage needs — plan a disk resize during the repurposing pass. ⚠️ **RAM target corrected, July 2026:** the original 8 GB target here was a pre-usage guess and turned out to sit below Splunk's real observed peak (~9 GB) — see Phase 4's "Sizing the rebuild" note. Actual target is now **4 vCPU / 12 GB / 100 GB**.
      - Fully uninstall Wazuh first (see Troubleshooting Log — Wazuh section, now deprecated/historical) before installing Splunk to avoid port/service conflicts.

- [x] **Create OpenVAS VM on Minimox** (`scanner.home.arpa`) — ⚠️ **moved to Bigmox,
      July 2026**, rebuilt at 8 GB RAM after Minimox proved too tight once real
      scan load exceeded 4 GB; see Phase 4's "Why This Moved" note for the full
      RAM math.
      - OS: Ubuntu 22.04 LTS
      - Resources: 4 vCPU, 8 GB RAM, 100 GB disk (Greenbone's own minimum is 4 GB RAM / 20 GB disk for scanner + feed data — sized up for headroom)
      - Network bridge: `vmbr0` (VLAN 40)

- [x] **Create Ansible + Semaphore VM on Bigmox** (`ansible.home.arpa`) — ⚠️ **moved
      to Minimox, July 2026**, swapped with OpenVAS to free Bigmox's RAM for the
      heavier scanner workload; see Phase 4's "Why This Moved" note.
      - OS: Ubuntu 24.04 LTS
      - Resources: 2 vCPU, 2 GB RAM, 20 GB disk (Semaphore's own guidance: 2 GB RAM / 2 CPU cores minimum)
      - Network bridge: `vmbr0` (VLAN 40)

- [x] **Verify all VMs get the right IPs** — after first boot/repurpose:
```bash
ip a show ens18    # or eth0/ens3 depending on Proxmox NIC naming
```

- [x] **Confirm hostnames resolve from the desktop before moving to Phase 1**
      ```
      nslookup splunk.home.arpa
      nslookup wazuh-edr.home.arpa
      nslookup scanner.home.arpa
      nslookup ansible.home.arpa
      ```

### Sources
- Proxmox VM creation: https://pve.proxmox.com/wiki/Qemu/KVM_Virtual_Machines
- Greenbone Community Edition system requirements: https://greenbone.github.io/docs/latest/container/index.html
- Semaphore UI installation guide: https://semaphoreui.com/docs/admin-guide/installation

---

> [!info] Standing policy — telemetry baseline for every Linux VM (July 2026)
> Every Linux VM in this build gets the same three things, no exceptions decided
> case-by-case:
>
> - **Auditd** — OS-level audit trail (auth, sudo, file/process watch rules)
> - **Splunk Universal Forwarder** — ships Auditd's output to `splunk.home.arpa`
>   (Auditd and UF are really one decision, not two — Auditd has nowhere to go
>   without the UF)
> - **Wazuh agent** — FIM, rootkit detection, active response
>
> **The one exception:** `wazuh-edr.home.arpa` (the Wazuh manager itself) gets
> Auditd + UF, but *not* a Wazuh agent — it already watches its own filesystem via
> its own default local `syscheck` block, so a separate agent pointed at itself
> would be redundant.
>
> This wasn't the original plan — Phase 1's UF checklist and Phase 2's Auditd
> checklist were both written before `scanner.home.arpa` and `ansible.home.arpa`
> existed as their own VMs, and never got backfilled. Fixed below. The same
> baseline applies to every future IAM VM (`idm.home.arpa`, `iam.home.arpa`,
> `iam-apps.home.arpa`) from day one — see `Homelab-IAM-Build-Plan.md`.
>
> **Devices this doesn't apply to:** anything that can't run an agent at all —
> the Netgear switch, IoT devices, game consoles. That's exactly what OpenVAS's
> agentless scanning exists to cover instead (see Phase 4).

---

## Phase 1 — Splunk (Primary SIEM)

**Goal:** Splunk Enterprise running on the repurposed VM, receiving forwarded data
from every endpoint, with working indexes, a receiving port, and confirmed data flow.
This is now the centerpiece of the whole build — everything else either forwards into
Splunk or gets bridged into it.
**Estimated time:** 4–6 hours
**VM:** `splunk.home.arpa` (10.40.40.12, on Bigmox)

### What Splunk Is (and What the Free License Means for You)

Splunk is a SIEM/log aggregation platform built around its own Search Processing
Language (SPL). For a homelab, the free download gives you:

```
License Tier    | Ingest Limit | Alerting | Auth/Users | Duration
----------------|-------------|----------|------------|------------------
Enterprise Trial | Unlimited   | YES ✓    | YES ✓      | 60 days then auto-converts
Free (converted) | 500 MB/day  | NO ✗     | NO ✗       | Perpetual after trial
```

> **Important:** Do all alerting/dashboard work inside the 60-day trial window.
> After it converts to Free, alerting is disabled. The 500 MB/day limit is fine for
> a homelab with a handful of endpoints.

### Checklist

**Installation**

- [x] Download Splunk Enterprise from https://www.splunk.com/en_us/download/splunk-enterprise.html
```bash
wget -O splunk.deb 'https://download.splunk.com/products/splunk/releases/\
9.x.x/linux/splunk-9.x.x-linux-amd64.deb'
sudo dpkg -i splunk.deb
sudo /opt/splunk/bin/splunk start --accept-license
sudo /opt/splunk/bin/splunk enable boot-start
```

- [x] Access the Splunk web UI at `http://splunk.home.arpa:8000` and set admin
      credentials on first login.

**Configure Receiving + Indexes**

- [x] Splunk Web → Settings → Forwarding and Receiving → Receive Data → Add New
      → port `9997` → Save
- [x] Create indexes (Settings → Indexes → New Index), sized against the 190 GB
      shared volume cap noted in Server-VM-Specs.md (250 GB disk, ~220 GB usable
      after OS/app/minFreeSpace reservations):
      - `windows-events` — **75 GB** (Sysmon + Windows Security/System/Application
        logs; sized largest — Sysmon process-creation volume is the heaviest
        expected source)
      - `linux-audit` — **50 GB** (Auditd from Pi, Bigmox host, Minimox host,
        Laptop)
      - `wazuh-alerts` — **40 GB** *(renamed from `openedr-events`, July 2026 — see
        Phase 3's EDR pivot; fed by a Splunk UF on the Wazuh VM tailing
        `alerts.json`, not HEC)*
      - `openvas-scans` — **25 GB** (bridged from OpenVAS results — Phase 4/8;
        lowest volume — periodic scan snapshots, not continuous telemetry)

      Set via the **Max Size (MB)** field in the New Index dialog, or
      `maxTotalDataSizeMB` directly in `indexes.conf`: `76800` / `51200` /
      `40960` / `25600` respectively (75/50/40/25 GB × 1024).

      Indexes must exist before a forwarder/script writes to them — writes to a
      non-existent index fail silently.

**Universal Forwarder on Every Endpoint**

- [x] Install the Splunk Universal Forwarder (UF) on Desktop, Laptop, Pi, and both Proxmox hosts (Bigmox, Minimox). Download: https://www.splunk.com/en_us/download/universal-forwarder.html
      Point each install at `splunk.home.arpa:9997` as the receiving indexer.
      Actual per-source input configuration (Sysmon channel, Auditd file monitor)
      is covered in Phase 2.

- [ ] ⚠️ **Gap found July 2026:** this list was written before `scanner.home.arpa` and `ansible.home.arpa` existed as their own VMs — install the UF on both, same as every other Linux host, per the standing telemetry policy above. `wazuh-edr.home.arpa` already has one (Phase 3, Step 4). Every future IAM VM gets one too, from day one — see `Homelab-IAM-Build-Plan.md`.

- [x] Verify data is flowing once Phase 2 inputs are configured:
```spl
index="windows-events"
index="linux-audit"
```

### SPL Basics to Learn During This Phase

SPL is the actual interview-tested skill here — spend real time on this.

```spl
# Find all events in an index
index="windows-events"

# Filter to a specific event ID (4625 = failed logon)
index="windows-events" EventCode=4625

# Count failed logons by source host
index="windows-events" EventCode=4625 | stats count by host

# Alert-style search: more than 5 failed logons from same account in 10 min
index="windows-events" EventCode=4625
| bucket _time span=10m
| stats count by _time, Account_Name
| where count > 5
```

- [ ] Build at least 3 dashboards during the 60-day trial (failed logons, process
      creation via Sysmon, auditd privilege escalation attempts)
- [ ] Create at least 1 saved alert during the trial window

### Key Ports (for reference / future firewall rules)

```
Port  | Protocol | Direction                | Purpose
------|----------|--------------------------|----------------------------------
8000  | TCP      | Browser → Splunk         | Splunk Web UI
9997  | TCP      | UF → Splunk              | Forwarder receiving port
8089  | TCP      | Internal/API             | Splunk management port
8088  | TCP      | Endpoint/script → Splunk | HTTP Event Collector (Phase 8)
```

### Sources
- Splunk Enterprise download: https://www.splunk.com/en_us/download/splunk-enterprise.html
- Universal Forwarder download: https://www.splunk.com/en_us/download/universal-forwarder.html
- Splunk Free license limits: https://docs.splunk.com/Documentation/Splunk/latest/Admin/TypesofSplunklicenses
- SPL search reference: https://docs.splunk.com/Documentation/Splunk/latest/SearchReference/WhatsInThisManual

---

## Phase 2 — Endpoint Telemetry (Sysmon + Auditd)

**Goal:** Rich, structured telemetry flowing from every endpoint into Splunk —
Sysmon with the SwiftOnSecurity config on Windows machines, Auditd watch rules on
Linux machines — instead of relying on default OS logging alone.
**Estimated time:** 2–3 hours
**Endpoints:** Desktop (OS unconfirmed — verify before assuming Windows, see note
below) · Pi, Bigmox host, Minimox host, Laptop (Fedora Linux) — Linux/Auditd path

> **Endpoint OS status:** The laptop is confirmed **Fedora Linux** (see
> Troubleshooting Log — Current-Stack Issue 2), so it's listed under Auditd below,
> not Sysmon. Desktop's OS was never actually verified — it was defaulted to
> Windows without checking, which is exactly the mistake that got made with the
> laptop. Confirm Desktop's real OS before running either section below against it.

### Windows — Sysmon + SwiftOnSecurity

- [x] On the Splunk server, install the **Splunk Add-on for Microsoft Sysmon**
      (Splunkbase) — provides CIM-compliant field extractions for Sysmon events.

- [x] On each confirmed-Windows endpoint (Desktop, pending OS confirmation — do not run this section against the laptop, it's Fedora):
      - Download Sysmon from Microsoft Sysinternals:
        https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
      - Download the SwiftOnSecurity config (widely used baseline, pre-tuned to cut noise from normal Windows process activity):
        https://github.com/SwiftOnSecurity/sysmon-config
      - Install with the config applied:
        ```powershell
        sysmon64.exe -accepteula -i sysmonconfig-export.xml
        ```
      - Confirm it's running: `Get-Service Sysmon64`

- [x] Configure the Splunk UF's `inputs.conf` to collect the Sysmon operational
      log:
```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = windows-events
disabled = false
renderXml = false

[WinEventLog://Security]
index = windows-events
disabled = false

[WinEventLog://System]
index = windows-events
disabled = false
Restart the UF service after editing.
```
      

- [x] Verify in Splunk:
```spl
index="windows-events" source="WinEventLog:Microsoft-Windows-Sysmon/Operational"
```

### Linux — Auditd (Pi, Bigmox host, Minimox host, Laptop — Fedora, `scanner.home.arpa`, `ansible.home.arpa`)

> ⚠️ **Gap found July 2026:** `scanner.home.arpa` and `ansible.home.arpa` were
> missing from this list — same reason as the Phase 1 UF gap above, this section
> predates both VMs. Same steps below apply to them, no differences. Every future
> IAM VM gets this too, from day one.

- [x] Install and enable auditd:
```bash
sudo apt install -y auditd audispd-plugins
sudo systemctl enable --now auditd
```

- [x] Set `log_format=ENRICHED` in `/etc/audit/auditd.conf` (best practice for
      Splunk CIM field mapping), and change `log_group` from `root` to a group the
      Splunk UF user can read (commonly a dedicated `splunk` group) so the UF
      doesn't need to run as root just to read `/var/log/audit/audit.log`.

- [x] Add watch rules for security-relevant paths/syscalls, **persisted** (not
      just live `auditctl`, which doesn't survive a reboot). Create a dedicated
      file — keeps custom rules separate from the distro's stock
      `/etc/audit/rules.d/audit.rules` — and write rule syntax directly (no
      `auditctl`/`sudo` prefix inside the file itself):
```bash
sudo nano /etc/audit/rules.d/homelab.rules
```
```
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/sudoers -p wa -k sudoers_changes
-a always,exit -F arch=b64 -S execve -k exec_commands
```
      Load without a reboot, then verify:
```bash
sudo augenrules --load
sudo auditctl -l
```
      `auditctl -l` should list all four rules. Two things you'll see during
      `augenrules --load` that are **benign, not errors**: `Old style watch
      rules are slower` (expected — `-w` syntax is just less efficient
      in-kernel than an equivalent syscall-based `-a` rule, still works fine),
      and on some systems (e.g. Raspberry Pi OS) a repeated `auditctl -s`
      status dump — the field that actually matters there is `lost 0`
      (zero audit events dropped); `failure 1` is a config label (failure
      mode = log to kernel log), not an actual failure count.
      **Status: done on Laptop (Fedora) and Pi — Bigmox host and Minimox host
      still pending.**

- [x] Configure the Splunk UF's `inputs.conf` — on Linux this lives at:
```
/opt/splunkforwarder/etc/system/local/inputs.conf
```
```ini
[monitor:///var/log/audit/audit.log]
index = linux-audit
sourcetype = linux:audit
disabled = false
```
      Restart the UF after editing (requires root — this is also the command
      to use if you ever see a `SPLUNK_HOME ownership`/`Operation not
      permitted` warning, which just means Splunk detected files not owned by
      its boot-start user; run `sudo /opt/splunkforwarder/bin/splunk ftr` once
      to fix ownership, then commands run clean):
```bash
sudo /opt/splunkforwarder/bin/splunk restart
```
      Confirm the stanza actually loaded:
```bash
sudo /opt/splunkforwarder/bin/splunk btool inputs list --debug
```

- [x] Verify in Splunk:
```spl
index="linux-audit"
```

### Sources
- Splunk Add-on for Microsoft Sysmon: https://splunkbase.splunk.com/app/1914
- SwiftOnSecurity Sysmon config: https://github.com/SwiftOnSecurity/sysmon-config
- Sysmon (Microsoft Sysinternals): https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
- Configuring auditd for Splunk (Splunk Add-on for Linux): https://docs.splunk.com/Documentation/AddOns/released/Linux/Configure4
- Auditd + Splunk integration walkthrough: https://binadit.com/tutorials/configure-linux-audit-siem-integration-splunk

---

## Phase 3 — Wazuh (Manager + Agent Only — EDR Role, Not SIEM)

**Goal:** Wazuh manager running standalone (no indexer, no dashboard) on a freshly
rebuilt VM; Wazuh agents on Desktop and Laptop connected and confirmed `Active`;
alerts flowing into a new Splunk index via Universal Forwarder. Splunk remains the
only dashboard/query surface — Wazuh's own indexer and dashboard packages are
deliberately never installed. This phase is installs only — FIM watch paths and the
active-response auto-block rule that make the agents actually do file integrity
monitoring, rootkit detection, and active response are deliberately **not**
configured here; that's Phase 5's job, done as an Ansible playbook instead of
one-off SSH edits.
**Estimated time:** 3–5 hours
**VM:** `wazuh-edr.home.arpa` (10.40.40.13, on Minimox — rebuilt fresh from the old
OpenEDR VM, July 2026)

### Why This Is Wazuh Again, and Why That's Not a Contradiction

This slot went through three attempts before landing here — worth understanding why,
since it's easy to look at "Wazuh" in this doc twice and assume the SIEM decision got
reversed. It didn't.

**Attempt 1 — jymcheong/OpenEDR ("Free EDR").** Originally chosen for lower friction.
Turned out to be an abandoned fork — last real activity around 2021, renamed
specifically to distance itself from Comodo's project. This is what caused nearly
every Current-Stack Issue in the Troubleshooting Log below: no real Windows service
(it ran via scheduled tasks that silently deregistered themselves), a homebrew
SFTP-based upload mechanism, and the trust-policy/`evm.local.src` saga. Fully
uninstalled — see the Troubleshooting Log for the uninstall sequence, including the
1603 MSI errors hit along the way.

**Attempt 2 — ComodoSecurity/openedr (the "real" OpenEDR).** jymcheong's fork was
renamed *specifically* to avoid confusion with this project, so it seemed like the
obvious next move. Its own getting-started docs turned out to be a stub: no shipped
`docker-compose.yml` (just a pointer to "clone a pre-prepared ELK package" with no
link), a two-line Docker install doc, and — critically — no official Filebeat module
for its own telemetry format, meaning self-hosting it means hand-writing Logstash/
Filebeat parsers against undocumented output. The project's own README says as much:
its "Quick Start" section tells users to sign up for the paid Comodo Dragon Enterprise
cloud platform instead, describing self-hosting as "only a short-term solution until
all the easy-to-use packages for OpenEDR is finalized." Never actually built.

**Attempt 3 (current) — Wazuh, manager + agent only.** Research into what free,
self-hosted tools real organizations actually run turned up Wazuh as the one with
verifiable production adoption (30M+ downloads/year, ~16,000 GitHub stars, a 2026
Cybersecurity Stars Award, documented use by MSSPs and mid-market SOCs specifically
because it's free) — as opposed to the vendor "best of" listicles that kept ranking
OpenEDR first despite what its own docs actually contained.

The apparent contradiction: didn't we drop Wazuh already? Yes — the *full* Wazuh
stack (manager + wazuh-indexer/OpenSearch + wazuh-dashboard/Kibana-fork) was dropped
as the primary SIEM, specifically because SPL is more interview-relevant than Wazuh's
query language. That decision stands. What's running now is manager + agent only —
no indexer, no dashboard, no Wazuh UI you'd ever log into. Splunk's Universal
Forwarder tails the manager's local alerts file into Splunk exactly the way it
already tails Sysmon's and Auditd's output — Wazuh is a detection/response *source*
here, not a competing SIEM. The thing Splunk's UF can't do on its own — file
integrity monitoring, rootkit detection, and genuine active response (killing a
process, blocking an IP, quarantining a file) — is exactly the gap OpenEDR was
supposed to fill and couldn't.

One real capability upgrade out of this: Wazuh's agent is cross-platform. OpenEDR
was Windows-only, which is why the Fedora laptop was excluded from the EDR track
this whole time (see the endpoint OS note above). Wazuh covers both Desktop and
Laptop.

### Checklist

**Step 1 — Rebuild the VM fresh**

- [x] Destroy the old OpenEDR VM on Minimox (Docker/jymcheong remnants already torn
      down per the Troubleshooting Log) and create a new VM sized for a manager-only
      workload — no indexer means no OpenSearch-scale resource needs:
      - OS: Ubuntu 22.04 LTS
      - Resources: **2 vCPU, 4 GB RAM, 100 GB disk** (deliberately lighter than the
        old 4 vCPU/8 GB — Minimox is RAM-tight per Server-VM-Specs.md once the
        planned FreeIPA + demo-apps IAM VMs land there; this frees 4 GB back into
        that budget). Disk left at 100 GB since Minimox has ample free disk and
        shrinking a virtual disk safely is a needless risk for a resource that isn't
        actually scarce here.
      - Network bridge: `vmbr0` (VLAN 40)

- [x] Rename the DNS record `openedr.home.arpa` → `wazuh-edr.home.arpa` at
      `10.40.40.13` (done in Phase 0's checklist — confirm it actually took with
      `nslookup wazuh-edr.home.arpa` from the desktop).

**Step 2 — Install Wazuh manager only (no indexer, no dashboard)**

- [x] Add the Wazuh package repo:
```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee -a /etc/apt/sources.list.d/wazuh.list
sudo apt-get update
```
> **Gotcha hit during the actual install:** `sudo` has to sit on the `gpg` side of the
> pipe specifically, not on `curl` — `curl` just reads the key over the network (no
> permission issue), but `gpg --import` is what writes into `/usr/share/keyrings/`,
> a root-owned directory. Missing `sudo` there fails silently-ish (a `Permission
> denied` from gpg, easy to miss), the keyring file never gets created, and
> `apt-get update` then reports `NO_PUBKEY` and refuses to index the repo — which is
> why `wazuh-manager` shows as "Unable to locate package" even though the repo line
> itself was added successfully.
- [x] Install **only** the manager package — do not install `wazuh-indexer` or
      `wazuh-dashboard`:
```bash
sudo apt-get install -y wazuh-manager
sudo systemctl daemon-reload
sudo systemctl enable wazuh-manager
sudo systemctl start wazuh-manager
sudo systemctl status wazuh-manager
```
> **Skip the Filebeat-to-indexer step entirely.** Wazuh's official install guide
> normally has you configure Filebeat here to ship data to the indexer. Since
> there's no indexer in this build, skip that section completely — the manager
> writes alerts to `/var/ossec/logs/alerts/alerts.json` on its own regardless,
> which is the file Splunk reads directly in Step 4.

**Step 3 — Install agents (Desktop + Laptop)**

> **FIM + active-response config deliberately deferred — see Phase 5.** An earlier
> draft of this step had you hand-edit the manager's config directly over SSH: create
> a `windows` agent group with Windows-specific FIM paths, and replace the
> commented-out `<active-response>` placeholder in `ossec.conf` with a real
> auto-block-on-SSH-failure rule. One piece of that was actually applied — the
> `windows` group's `agent.conf` got created on disk — before deciding this
> shouldn't be a manual, untracked SSH session at all. Cleaning that up now so
> Phase 5's Ansible playbook starts from nothing rather than fighting a file it
> doesn't know about:
```bash
sudo rm -rf /var/ossec/etc/shared/windows
```
> `ossec.conf` itself was never actually edited — the active-response placeholder
> was only backed up, never replaced — so there's nothing to revert there. The
> `/var/ossec/etc/ossec.conf.bak` backup can stay; it's harmless.

- [x] **Desktop (Windows)** — confirm real OS first per the standing note in this doc, then:
```powershell
msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER="10.40.40.13" WAZUH_REGISTRATION_SERVER="10.40.40.13"
NET START WazuhSvc
```

- [x] **Laptop (Fedora)**:
```bash
sudo rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH
sudo tee /etc/yum.repos.d/wazuh.repo > /dev/null <<EOF
[wazuh]
gpgcheck=1
gpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH
enabled=1
name=EL-\$releasever - Wazuh
baseurl=https://packages.wazuh.com/4.x/yum/
priority=1
EOF
sudo WAZUH_MANAGER="10.40.40.13" dnf install -y wazuh-agent
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

- [x] Confirm both agents show `Active` on the manager:
```bash
sudo /var/ossec/bin/agent_control -l
```

- [ ] ⚠️ **Scope note, July 2026:** this step only ever covered the two workstation
      endpoints. Per the standing telemetry policy earlier in this doc, `scanner.home.arpa`
      and `ansible.home.arpa` need a Wazuh agent too — every Linux server VM gets one,
      not just workstations. Rather than a third manual install here, this is being
      done through Phase 5's Ansible pipeline instead, so agent package install and
      FIM/config land together in one playbook run per host. See Phase 5.

**Step 4 — Splunk UF on the Wazuh VM, tailing alerts.json**

- [x] Retire the `openedr-events` index (Settings → Indexes) and create `wazuh-alerts` in its place at the same **40 GB** allocation — reuses the existing 190 GB shared volume budget from Phase 1 rather than adding a new index on top of it. There's no OpenEDR historical data worth preserving.

- [x] Install Splunk UF on `wazuh-edr.home.arpa` (same as every other endpoint), point at `splunk.home.arpa:9997`.

- [x] Configure the monitor stanza —
      `/opt/splunkforwarder/etc/system/local/inputs.conf`:
```ini
[monitor:///var/ossec/logs/alerts/alerts.json]
disabled = 0
host = wazuh-edr.home.arpa
index = wazuh-alerts
sourcetype = wazuh-alerts
```

- [x] Configure JSON parsing —
      `/opt/splunkforwarder/etc/system/local/props.conf`:
```ini
[wazuh-alerts]
DATETIME_CONFIG =
INDEXED_EXTRACTIONS = json
KV_MODE = none
NO_BINARY_CHECK = true
category = Application
disabled = false
pulldown_type = true
```

- [x] Restart the UF and verify in Splunk:
```spl
index="wazuh-alerts"
```

### Key Ports

```
Port  | Protocol | Direction                    | Purpose
------|----------|-------------------------------|--------------------------------
1514  | UDP      | Agent → Manager               | Event data
1515  | TCP      | Agent → Manager               | Agent enrollment/registration
9997  | TCP      | UF (on Wazuh VM) → Splunk     | Forwarder receiving port (existing rule)
```
> No new firewall rule appears necessary — Desktop/Laptop (VLAN 1) initiating to the
> Wazuh VM (VLAN 40) follows the same direction as existing Splunk UF/Sysmon traffic,
> which already works under the current zone policy. Confirm once agents are
> installed rather than assuming.

### Sources
- Wazuh installation guide — server step by step: https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html
- Wazuh agent — Windows package install: https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html
- Wazuh agent — Linux package install: https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-linux.html
- Wazuh active response documentation: https://documentation.wazuh.com/current/user-manual/capabilities/active-response/how-to-configure.html
- Wazuh Splunk integration guide: https://documentation.wazuh.com/current/integrations-guide/splunk/index.html
- ComodoSecurity/openedr (evaluated, not used): https://github.com/ComodoSecurity/openedr
- jymcheong/OpenEDR (evaluated, not used — see Troubleshooting Log for uninstall): https://github.com/jymcheong/OpenEDR

---

## Phase 4 — OpenVAS / Greenbone (Vulnerability Scanning)

**Goal:** Greenbone Community Edition (OpenVAS) running on its own VM, feed data loaded, and a first authenticated/unauthenticated scan run against the homelab's own IP ranges — including devices with no agent (switch, IoT, consoles).
**Estimated time:** 4–6 hours (rebuild + feed sync from scratch on first run can take a while)
**VM:** `scanner.home.arpa` (10.40.40.14, rebuilt fresh on **Bigmox**, moved from Minimox,
July 2026)

### Why This Tool

Vulnerability management is a distinct, JD-common skill from both SIEM and EDR work, and it's the one tool in this stack that needs no endpoint agent — it covers every IP on the network, including the Netgear switch, IoT devices, and game consoles that can't run a Splunk UF or Wazuh agent. This also sets up the scan → attack → detect → remediate → re-scan loop planned for VLAN 50 (Cyber Lab) once that's built.

### Why This Moved From Minimox to Bigmox

Originally built on Minimox at 8 GB RAM (Greenbone's own documented minimum), but
the real VM was only ever actually configured at 4 GB — and even that wasn't
enough; it was running out of RAM mid-scan. Fixing that by just raising Minimox's
allocation doesn't work: Minimox is a 15.34 GiB host that's also carrying the
Wazuh EDR manager (4 GB) plus two still-unbuilt IAM VMs (FreeIPA + demo-apps, 4 GB
each planned). Even at OpenVAS's bare 8 GB minimum, that's 20 GB configured against
15.34 GiB physical before those IAM VMs are even built — and OpenVAS realistically
wants more than 8 GB once it's scanning multiple VLANs concurrently.

Dropping jymcheong/OpenEDR (8 GB) for Wazuh manager-only (4 GB) did free 4 GB back
into Minimox's budget, but that room was already earmarked for the FreeIPA/
demo-apps landing, not for absorbing a scanner that's already outgrowing its
original allocation. Bigmox has the physical headroom instead (31.29 GiB), and
Ansible + Semaphore — light at 2 GB — moves to Minimox in its place, which is
where the freed-up OpenEDR room actually gets used. See `Homelab-Server-VM-Specs.md`
for the full RAM math on both hosts.

**Sizing the rebuild:** starting at Greenbone's own documented minimum, **8 GB**
RAM / 4 vCPU (up from the 4 GB that proved insufficient).

**Update — Splunk trim actually happened, with real numbers instead of a guess:**
the "trim Splunk down" open item had been sitting on an untested **8 GB** target
since the original repurposing. Actual observed usage (a few days of Splunk
running alone, no Wazuh indexer/dashboard alongside it) never went past **9 GB** —
meaning the old 8 GB target was already *below* real peak usage and would have
been the wrong number to trim to. Trimmed to **12 GB** instead (16 → 12), which
gives Splunk ~3 GB of headroom above its observed ceiling rather than sitting
right at it. See the Right-Size checklist item and `Homelab-Server-VM-Specs.md`
for the resize commands and updated spec.

That trim alone frees the room OpenVAS needs — today's actual Bigmox math is
Splunk 12 GB + OpenVAS 8 GB = 20 GB of 31.29 GiB, comfortable with room to spare.
The tighter math only shows up once Splunk SOAR (8 GB, planned) and Keycloak
(4 GB, planned) actually get built — 12 + 8 + 8 + 4 = 32 GB, right at/slightly
over the 31.29 GiB ceiling depending on whether OpenVAS ends up needing more than
8 GB. Not an immediate problem; re-run this math once SOAR and Keycloak are real
and OpenVAS's post-rebuild usage is known, same as every other "re-run once built"
note in this doc.

### Checklist

- [x] **Destroy the old `scanner.home.arpa` VM on Minimox** (data isn't worth
      preserving — Greenbone's own feed will just re-sync fresh) and create a new
      VM on **Bigmox**:
      - OS: Ubuntu 22.04 LTS
      - Resources: **4 vCPU, 8 GB RAM, 100 GB disk** (RAM raised from the 4 GB that proved insufficient; disk unchanged, still plenty for feed + scan data)
      - Network bridge: `vmbr0` (VLAN 40) — same bridge/VLAN as Minimox, so `scanner.home.arpa` keeps the same IP, `10.40.40.14`; no DNS change needed.

- [x] Update the DHCP reservation for `scanner.home.arpa` in `Homelab-Network-Documentation.md` — the new VM has a new virtual NIC (new MAC address), so the existing MAC-based reservation for `10.40.40.14` needs to point at the new VM's MAC, not the old one.

- [x] Confirm DNS still resolves correctly post-rebuild:
```bash
nslookup scanner.home.arpa
```

- [x] Install Docker + Docker Compose on the OpenVAS VM (official Docker CE apt-repo install — Phase 3 no longer uses Docker now that it's Wazuh's native packages instead of the old jymcheong Docker stack):
```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
sudo apt-get remove -y $pkg
done
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
sudo systemctl status docker
docker compose version
```

- [x] Pull Greenbone's official Community Container compose file. **Verified July 2026** — the file was renamed from `docker-compose.yml` to `compose.yaml` upstream; the old filename no longer exists at this path (the doc's previous command was also just broken — the URL had gotten split across lines):
```bash
mkdir -p ~/openvas && cd ~/openvas
curl -f -L https://greenbone.github.io/docs/latest/_static/compose.yaml -o compose.yaml
```
      **Before starting it**, fix the default port bindings. The official
      compose file binds the `nginx` service (TLS termination + the GSA web
      UI) to `127.0.0.1` only — the same loopback-only gotcha that showed up
      repeatedly with the old jymcheong OpenEDR containers before that stack
      was decommissioned. Left as-is, `https://scanner.home.arpa` will not be
      reachable from any other machine on the network:
```bash
sed -i 's/127.0.0.1:443:443/443:443/; s/127.0.0.1:9392:9392/9392:9392/' compose.yaml
```
      Then start it:
```bash
docker compose up -d
```

- [ ] Wait for the vulnerability feed to fully sync on first start (this can take a while — check container logs for feed-sync completion before scanning):
```bash
docker compose logs -f gvmd
```

- [x] Access the Greenbone Security Assistant (GSA) web UI at
      `https://scanner.home.arpa` (self-signed cert auto-generated by the
      `gvm-config`/`nginx` containers on first start — accept the browser
      warning). **Modern Greenbone containers no longer ship a fixed
      `admin`/`admin` default** — an initial admin password is auto-generated
      on first startup. Find it with:
```bash
docker compose logs gvmd | grep -i "user\|password"
```

- [ ] Create a scan target covering the homelab's active subnets:
      ```
      10.10.10.0/24   (Main LAN)
      10.20.20.0/24   (IoT)
      10.40.40.0/24   (Servers)
      10.60.60.0/24   (Gaming)
      10.99.99.0/24   (Management)
      ```

- [ ] Run a first unauthenticated scan against VLAN 40 (your own SOC VMs) as a safe first test, then expand to other VLANs once you understand the scan duration/impact on live devices (avoid running heavy scans against always-on IoT/gaming devices during hours you're using them).

- [ ] Review results in GSA — CVSS-scored findings per host, exportable reports.

### Sources
- Greenbone Community Containers documentation: https://greenbone.github.io/docs/latest/container/index.html
- OpenVAS/GVM install walkthrough: https://serverspace.io/support/help/how-to-install-and-use-openvas-gvm-on-ubuntu/
- Greenbone Community Edition system requirements discussion (RAM sizing): https://forum.greenbone.net/t/openvas-system-requirments/1636

---

## Phase 5 — Ansible + Semaphore (Automation Layer)

**Goal:** Semaphore's web UI running on top of Ansible, with a working inventory
covering the homelab's Linux and Windows endpoints, and the Wazuh manager's FIM
(`windows` group `agent.conf`) and active-response (`ossec.conf`) config applied
via a real playbook instead of by hand over SSH. This is the execution layer that
Splunk SOAR will call into once that's built.
**Estimated time:** 4–5 hours
**VM:** `ansible.home.arpa` (10.40.40.15, moved to **Minimox** July 2026 — Bigmox
needed the RAM headroom for OpenVAS instead; see Phase 4's "Why This Moved" note)

### Why This Tool

Config management/automation is its own job track (Security Automation Engineer, DevSecOps), and Ansible + a UI on top of it (Semaphore) is directly relevant there. It's also the actual "do something" half of detect → respond — Splunk can tell you something's wrong, but Ansible is what actually executes a fix.

### First Real Target — Wazuh Manager Config

Phase 3 deliberately stopped at manager + agent installs and left the Wazuh
manager's own config alone. That config — the `windows` agent group's FIM paths
and the SSH-brute-force active-response rule — is this playbook's actual first
job, for two reasons: it's a real change against a real box (not a throwaway
fact-gathering run), and it fixes the actual problem with doing it by hand — no
record of what changed, no easy revert, no way to re-apply cleanly after a VM
rebuild.

Two pieces of config, two different Ansible modules, because they're structurally
different edits:

- **`windows` group's `agent.conf`** — this file doesn't exist until Ansible
  creates it, so it's a straight file drop: `ansible.builtin.template` (or
  `copy`, since there's no templating needed yet — every value is static) writing
  the full file to `/var/ossec/etc/shared/windows/agent.conf` on
  `wazuh-edr.home.arpa`.

- **`ossec.conf`'s active-response block** — this file already exists with real
  content around the spot being edited, so a full-file overwrite is wrong here.
  `ansible.builtin.blockinfile`, marked with its own begin/end comment markers,
  inserted after the last `<command>` block and before `<!-- Log analysis -->`.
  Idempotent by design — reruns don't duplicate the block, and it's easy to see
  in a diff exactly what Ansible owns versus what shipped with the package.

Both need a handler that restarts `wazuh-manager` only when the file actually
changed (`notify:` on each task, `listen: wazuh-manager` restart handler) —
active-response and FIM group changes aren't hot-reloaded.

Rough shape:
```yaml
# inventory: wazuh group
[wazuh]
wazuh-edr.home.arpa

# playbook
- hosts: wazuh
  become: true
  tasks:
    - name: Deploy windows FIM group config
      ansible.builtin.copy:
        src: files/windows-agent.conf
        dest: /var/ossec/etc/shared/windows/agent.conf
        owner: wazuh
        group: wazuh
        mode: "0644"
      notify: Restart wazuh-manager

    - name: Enable SSH brute-force active-response
      ansible.builtin.blockinfile:
        path: /var/ossec/etc/ossec.conf
        marker: "<!-- {mark} ANSIBLE MANAGED BLOCK: active-response -->"
        insertafter: '</command>'
        block: |
          <active-response>
            <disabled>no</disabled>
            <command>firewall-drop</command>
            <location>local</location>
            <rules_id>5712</rules_id>
            <timeout>600</timeout>
          </active-response>
      notify: Restart wazuh-manager

  handlers:
    - name: Restart wazuh-manager
      ansible.builtin.systemd:
        name: wazuh-manager
        state: restarted
```
The `5712` rule ID still isn't independently verified against the installed
ruleset — check `grep -r 'id="5712"' /var/ossec/ruleset/rules/` on the manager
before trusting it, same caveat as when this was first drafted in Phase 3.

### Second Target — Wazuh Agent Rollout to Server VMs

Per the standing telemetry policy earlier in this doc, `scanner.home.arpa` and
`ansible.home.arpa` both need a Wazuh agent (Phase 3 only ever installed agents
on the two workstation endpoints, Desktop and Laptop). Rather than a third round
of manual `dnf`/`apt` commands, this is the actual point of having Ansible
manage Wazuh at all — package install and enrollment become one more playbook
against the same `wazuh` group pattern used above, and every future IAM VM
inherits this for free once it's in inventory:

```yaml
- hosts: linux_servers   # scanner.home.arpa, ansible.home.arpa, future IAM VMs
  become: true
  tasks:
    - name: Add Wazuh apt repo
      # same GPG key / repo steps as Phase 3, Step 2 — sudo on the gpg side of
      # the pipe, not curl (see Phase 3's documented gotcha)
      ...

    - name: Install wazuh-agent
      ansible.builtin.apt:
        name: wazuh-agent
        state: present
      environment:
        WAZUH_MANAGER: "10.40.40.13"

    - name: Enable and start wazuh-agent
      ansible.builtin.systemd:
        name: wazuh-agent
        enabled: true
        state: started
```
No FIM/active-response config needed on these agents beyond the manager's own
default group — they're server infrastructure, not Windows workstations, so the
`windows` group config above doesn't apply. Confirm enrollment the same way as
Phase 3: `sudo /var/ossec/bin/agent_control -l` on the manager should list both
as `Active`.

### Checklist

- [ ] Install Ansible on the VM:
```bash
sudo apt update && sudo apt install -y ansible
```

- [ ] Install Semaphore (native package is the simplest path):
      ```bash
      curl -s https://raw.githubusercontent.com/semaphoreui/semaphore/develop/deployment/install.sh | sudo bash
      ```
      Choose a database backend during setup — BoltDB needs no separate service
      and is fine for a single-user homelab; PostgreSQL/MySQL is the better
      choice if you ever want concurrent multi-user access.

- [ ] Start Semaphore and confirm the web UI is reachable at
      `http://ansible.home.arpa:3000`.

- [ ] Set up SSH key-based auth from the Ansible VM to Linux endpoints (Pi,
      Bigmox host, Minimox host) — avoid password auth for playbook runs.

- [ ] Set up WinRM on Desktop/Laptop for Windows playbook targets (this is the
      more fiddly half — WinRM needs to be explicitly enabled and configured for
      Ansible's `winrm` connection plugin; budget extra time here).

- [ ] Build a basic inventory in Semaphore covering all endpoints, grouped by OS,
      including a `wazuh` group containing just `wazuh-edr.home.arpa`, and a
      `linux_servers` group containing `scanner.home.arpa` and `ansible.home.arpa`
      (add each future IAM VM to this group as it's built).

- [ ] Run a first read-only playbook against each OS type (e.g., a fact-gathering
      or patch-check playbook) to confirm SSH/WinRM connectivity actually works
      before trusting Ansible with anything that changes state.

- [ ] Write and run the Wazuh config playbook from the "First Real Target" section
      above — deploy the `windows` group's `agent.conf` via `copy`, add the
      active-response block via `blockinfile`, restart `wazuh-manager` via the
      handler.

- [ ] Verify on `wazuh-edr.home.arpa` that both pieces landed correctly:
```bash
cat /var/ossec/etc/shared/windows/agent.conf
grep -A6 'ANSIBLE MANAGED BLOCK' /var/ossec/etc/ossec.conf
sudo systemctl status wazuh-manager
```

- [ ] Re-run the same playbook a second time and confirm Ansible reports the
      tasks as unchanged (not re-applied) — this is the actual point of doing it
      this way instead of by hand: idempotent, safe to rerun after a VM rebuild.

- [ ] Write and run the agent-rollout playbook from "Second Target" above against
      the `linux_servers` group — installs and enrolls the Wazuh agent on
      `scanner.home.arpa` and `ansible.home.arpa`, closing the gap flagged in
      Phase 3. Confirm both show `Active` via `agent_control -l` on the manager.

### Sources
- Semaphore UI installation guide: https://semaphoreui.com/docs/admin-guide/installation
- Semaphore install walkthrough (Ubuntu/Debian): https://computingforgeeks.com/install-semaphore-ubuntu-debian/
- Ansible documentation: https://docs.ansible.com/
- `ansible.builtin.blockinfile` module docs: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/blockinfile_module.html
- `ansible.builtin.copy` module docs: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html

---

## Phase 6 — Validation

**Goal:** Generate real (simulated) attack activity and confirm it surfaces across
Splunk and OpenVAS, with Wazuh's FIM/active-response contributing where relevant —
document what each tool caught vs. missed. This is the actual cybersecurity practice
payoff.
**Estimated time:** 2–3 hours
**Hosts:** Desktop/Laptop (attack source) + test VM (target)

### Why a Test VM, Not Your Daily Driver

Spin up a small Ubuntu or Kali VM on Bigmox or Minimox, enroll it with Splunk UF + a
Wazuh agent — cross-platform, unlike OpenEDR, so this works the same whether the test
VM ends up Linux or Windows — and use it as the victim machine.

### Tests to Run

**Test 1 — EICAR Antivirus Test File**
```bash
echo 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*' \
  > /tmp/eicar.txt
```
Expected: Auditd watch rule (if `/tmp` is monitored) or Sysmon file-create event surfaces in Splunk.

**Test 2 — Failed SSH Brute-Force**
```bash
for i in {1..10}; do ssh wronguser@10.40.40.x 2>/dev/null; done
```
Expected: Auditd logs the failed attempts; build a Splunk detection search for repeated failures from the same source in a short window (this is now something *you* build in SPL rather than something a prebuilt Wazuh rule hands you — that's the point).

**Test 3 — Nmap Port Scan**
```bash
nmap -sV 10.40.40.x
```
Expected: Visible in an OpenVAS scan of the target if run around the same time; also a good candidate for a Splunk detection built on Auditd/network telemetry.

**Test 4 — Suspicious Process Execution (Windows, Desktop only — pending OS confirmation)**
```powershell
$encoded = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes("whoami"))
powershell -encodedCommand $encoded
```
Expected: Sysmon Event ID 1 (process creation) with the encoded command line visible in Splunk. Wazuh's coverage here is narrower than OpenEDR's process-tree view would have been — this test isn't a strong FIM/active-response trigger unless the command happens to touch a watched path, so Wazuh's real contribution to detection engineering in this build is closer to Tests 1–2 (FIM, active response) than to general process visibility, which Sysmon already owns. Only valid once Desktop's OS is actually confirmed Windows.

**Test 4b — Suspicious Process Execution (Laptop, Fedora/Auditd equivalent)**
```bash
echo d2hvYW1p | base64 -d | bash
```
Expected: The `exec_commands` auditd rule from Phase 2 fires (execve syscall watch), visible in Splunk under `index="linux-audit"`. This is the laptop's equivalent of Test 4 — no Sysmon/OpenEDR coverage exists for it, so Auditd is the only detection path.

### Validation Checklist

- [ ] Test 1 (EICAR): file-create event visible in Splunk
- [ ] Test 2 (Brute-force): failed logons visible in Splunk `linux-audit`; custom SPL detection built and saved
- [ ] Test 3 (Nmap): scan visible in OpenVAS results; detection attempted in Splunk
- [ ] Test 4 (PowerShell, Desktop): Sysmon event visible in Splunk — pending Desktop OS confirmation (Wazuh's coverage here is narrower than OpenEDR's was, see Test 4 note)
- [ ] Test 4b (Laptop/Auditd): exec_commands event visible in Splunk `linux-audit`
- [ ] Document what each tool caught vs. missed in the Troubleshooting Log below

### Sources
- EICAR test file standard: https://www.eicar.org/download-anti-malware-testfile/
- Nmap official site: https://nmap.org/
- Atomic Red Team (more advanced test cases for later): https://github.com/redcanaryco/atomic-red-team

---

## Phase 7 — Cleanup and Tie-Down

**Goal:** No loose firewall ends, resource usage reviewed, everything documented.
**Estimated time:** 1–2 hours

### Checklist

- [ ] **Re-verify all connection-state fixes are still in place** - "Block Servers to Management" = New, "Block Servers to Main" = New

- [ ] **Check Bigmox and Minimox resource usage** — re-pull a fresh snapshot in Proxmox now that the stack has changed (Splunk repurposed VM, new OpenVAS
      and Ansible VMs). The last documented snapshot (`Homelab-Server-VM-Specs.md`,
      July 5, 2026) was taken while the VM was still running Wazuh — it's stale
      for sizing decisions now.

- [ ] **Verify all DNS records are correct and consistent**
      ```
      nslookup splunk.home.arpa      → 10.40.40.12
      nslookup wazuh-edr.home.arpa   → 10.40.40.13
      nslookup scanner.home.arpa     → 10.40.40.14
      nslookup ansible.home.arpa     → 10.40.40.15
      ```

- [ ] **Plan VLAN 50 (Cyber Lab) zone assignment** — per the network doc roadmap, create a dedicated zone (not Internal) with narrow allow rules only for monitoring/scanning traffic back to `splunk.home.arpa` and `scanner.home.arpa`.

- [ ] **Note the Splunk trial expiry date** and export saved searches/dashboards before it converts to Free (Settings → Licensing).

---

## Phase 8 — Unified Dashboard & SOAR (Capstone)

**Goal:** One Splunk view correlating Splunk-native telemetry (Sysmon/Auditd), Wazuh
EDR alerts, and OpenVAS (bridged via HEC) — the primary demo screen for the YouTube
series / recruiter walkthrough. Then, once built, Splunk SOAR completes the pipeline:
SIEM detects → SOAR orchestrates → Ansible executes.
**Estimated time:** 2–4 hours for the dashboard + bridge script (lighter than
originally scoped — Wazuh needed no bridge script, only OpenVAS does now). SOAR
itself is a separate future build, not yet started.
**VM:** Bridge script runs on the OpenVAS VM; dashboard built in Splunk
(`splunk.home.arpa:8000`); SOAR (future) on `soar.home.arpa`

### Why Only OpenVAS Needs a Bridge Script

OpenVAS's results live in its own scan-report database with no flat file Splunk's UF
can tail, so it still needs a small script that queries the source directly and
pushes events into Splunk over its HTTP Event Collector (HEC, port 8088). Wazuh
needs no equivalent bridge — its manager already writes `alerts.json` to disk, and
the Splunk UF configured in Phase 3 tails it directly into `index=wazuh-alerts`,
the same pattern Sysmon and Auditd already use. This is one of the concrete
advantages of the Wazuh pivot over OpenEDR: one less moving part in this phase.

### Checklist

**Enable Splunk HEC**

- [ ] Splunk Web → Settings → Data Inputs → HTTP Event Collector → New Token
      (`openvas-hec`)
- [ ] Global Settings → set to **Enabled**
- [ ] Confirm the `openvas-scans` index exists
- [ ] Verify HEC is reachable:
```bash
curl -k https://splunk.home.arpa:8088/services/collector/health \
-H "Authorization: Splunk <token>"
```

**Build the Bridge Script**

- [ ] OpenVAS → Splunk: Python script on the OpenVAS VM against the Greenbone/GVM
      API (GMP — Greenbone Management Protocol) to pull completed scan results and
      push them as HEC events to `https://splunk.home.arpa:8088/services/collector/event`.
- [ ] Schedule it on a cron/systemd timer (every 60 seconds is reasonable to
      start).

**Verify and Build the Dashboard**

- [ ] Confirm events landing: `index="wazuh-alerts"` (already flowing since Phase 3,
      no bridge needed), `index="openvas-scans"` (via the bridge script above)
- [ ] In Splunk Dashboard Studio, build one unified dashboard with panels for
      Windows/Sysmon events, Linux/Auditd events, Wazuh EDR alerts, and OpenVAS
      findings — plus a correlated view joining across sources on hostname/IP.
- [ ] Re-run the Phase 6 PowerShell encoded-command test and confirm it shows up
      on the unified dashboard without needing a separate console for any source —
      the whole point of dropping OpenEDR's standalone UI in favor of Wazuh
      forwarding straight into Splunk.

**Splunk SOAR (Future — Not Yet Built)**

- [ ] Install Splunk SOAR On-premises Community Edition on `soar.home.arpa`.
      Note: SOAR **cannot** run inside Docker/Podman — it needs a dedicated,
      unprivileged Linux install (creates its own `phantom` OS user). Run
      `soar-prepare-system.sh` first to validate prerequisites.
      Free tier: 100 automated actions/day, 5 active cases.
- [ ] Connect SOAR to Splunk as a detection source and to Ansible/Semaphore as
      an execution target — this is the actual "orchestration" piece: SOAR
      receives a Splunk-detected event, decides on a playbook, and triggers
      Ansible to execute it.

### Sources
- Splunk HTTP Event Collector: https://docs.splunk.com/Documentation/Splunk/latest/Data/UsetheHTTPEventCollector
- Splunk Dashboard Studio: https://docs.splunk.com/Documentation/Splunk/latest/DashStudio/dsOverview
- Splunk SOAR (On-premises) system requirements: https://help.splunk.com/en/splunk-soar/soar-on-premises/install-and-upgrade-soar-on-premises/8.5.0/system-requirements
- Splunk SOAR installation methods: https://help.splunk.com/en/splunk-soar/soar-on-premises/install-and-upgrade-soar-on-premises/8.4.0/get-splunk-soar-on-premises/installation-methods-for-splunk-soar-on-premises

---

## New Roadmap Items

- [ ] VLAN 50 (Cyber Lab) — create a dedicated zone, not Internal (see Phase 7)
- [ ] VLAN 30 (Guest) — reserved, not yet built, no SOC dependency
- [ ] Once VLAN 50 exists, point OpenVAS at it for the scan → attack → detect →
      remediate → re-scan loop
- [ ] Explore Atomic Red Team for structured attack simulation on VLAN 50
- [ ] Splunk SOAR is the final piece before the build is "done" for the YouTube
      series — schedule it last, after Phases 1–7 are stable
- [ ] Right-size the Splunk VM's RAM down from its Wazuh-era 16 GB — observed
      usage never exceeded 9 GB over several days of Splunk running alone, so
      trim to **12 GB** (not the old untested 8 GB target, which actually sat
      below the real observed peak) via `qm set <vmid> --memory 12288` on Bigmox
      (shut the VM down first — memory hot-plug isn't guaranteed to reflect
      correctly inside the guest for a downsize; safest as `qm shutdown <vmid>`
      → `qm set <vmid> --memory 12288` → `qm start <vmid>`).
      vCPU right-size (8 → 4) still open, lower priority — it isn't blocking
      anything else's headroom the way the memory trim was.

---

## Command Reference (This Build)

Broken out by program, command + what it actually does, for lookup rather than
reading top to bottom. Multi-step workflows (persisting audit rules, tracing a
missing-data search) stay as ordered code blocks since sequence matters there.

### Splunk (indexer/search head — `splunk.home.arpa`)

| Command | What it does |
|---|---|
| `sudo /opt/splunk/bin/splunk start` | Start the indexer/search head |
| `sudo /opt/splunk/bin/splunk stop` | Stop it |
| `sudo /opt/splunk/bin/splunk restart` | Restart — needed after most server-side config changes |
| `sudo /opt/splunk/bin/splunk status` | Confirm it's actually running |

### Splunk Universal Forwarder (any endpoint)

**Download + install** (check https://www.splunk.com/en_us/download/universal-forwarder.html
for the current version — `9.x.x` below is a placeholder, same convention as the
Enterprise install in Phase 1):

```bash
# Ubuntu/Debian (scanner.home.arpa, ansible.home.arpa, Pi, Bigmox/Minimox hosts)
wget -O splunkforwarder.deb 'https://download.splunk.com/products/universalforwarder/releases/\
9.x.x/linux/splunkforwarder-9.x.x-linux-2.6-amd64.deb'
sudo dpkg -i splunkforwarder.deb
sudo /opt/splunkforwarder/bin/splunk start --accept-license
sudo /opt/splunkforwarder/bin/splunk enable boot-start
```

```bash
# Fedora/RHEL (Laptop)
wget -O splunkforwarder.rpm 'https://download.splunk.com/products/universalforwarder/releases/\
9.x.x/linux/splunkforwarder-9.x.x-linux-x86_64.rpm'
sudo rpm -i splunkforwarder.rpm
sudo /opt/splunkforwarder/bin/splunk start --accept-license
sudo /opt/splunkforwarder/bin/splunk enable boot-start
```

```powershell
# Windows (Desktop) — download the .msi from the same page, then:
msiexec.exe /i splunkforwarder.msi AGREETOLICENSE=Yes RECEIVING_INDEXER="splunk.home.arpa:9997" WINEVENTLOG_SEC_ENABLE=1 /quiet
```

**Configure — point it at the indexer** (not automatic on Linux; the Windows
`.msi` flags above already do this at install time):
```bash
sudo /opt/splunkforwarder/bin/splunk add forward-server splunk.home.arpa:9997 -auth admin:<password>
```

| Command | What it does |
|---|---|
| `sudo /opt/splunkforwarder/bin/splunk start` | Start the UF |
| `sudo /opt/splunkforwarder/bin/splunk restart` | Restart — needed after editing `inputs.conf`/`outputs.conf` |
| `sudo /opt/splunkforwarder/bin/splunk status` | Confirm the UF is running |
| `sudo tail -f /opt/splunkforwarder/var/log/splunk/splunkd.log` | Live UF log — first place to look when data isn't arriving |
| `sudo /opt/splunkforwarder/bin/splunk list monitor` | List every file/path this UF is currently watching |
| `sudo /opt/splunkforwarder/bin/splunk list forward-server` | Confirm the UF actually has an active receiving indexer configured |
| `sudo /opt/splunkforwarder/bin/splunk add forward-server splunk.home.arpa:9997 -auth admin:<password>` | Point this UF at the indexer — **not automatic**, has to be run explicitly on every new UF install (the actual cause of the "no results" issue on the Wazuh VM) |
| `sudo /opt/splunkforwarder/bin/splunk btool inputs list --debug` | Show every `inputs.conf` stanza actually loaded, and which file it came from |
| `sudo /opt/splunkforwarder/bin/splunk ftr` | Fixes `SPLUNK_HOME` ownership if you see that warning on start |

`inputs.conf` locations (create if missing, restart the UF after editing):
- Linux: `/opt/splunkforwarder/etc/system/local/inputs.conf`
- Windows: `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`

**Windows UF (PowerShell)** — same commands, different binary path:

| Command | What it does |
|---|---|
| `& 'C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe' list forward-server` | Confirm the UF is actively connected to the indexer |
| `& 'C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe' btool inputs list --debug` | Confirm a given `inputs.conf` stanza is actually loading, and from where |
| `Get-Content 'C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log' -Tail 200 \| Select-String -Pattern 'Sysmon'` | Search the UF's own log for a specific channel/input's errors |

### Auditd (Linux endpoints)

**Install + enable:**
```bash
sudo apt install -y auditd audispd-plugins
sudo systemctl enable --now auditd
```

**Configure** — CIM-friendly log format + a readable log group (edit
`/etc/audit/auditd.conf`: `log_format = ENRICHED`, `log_group = splunk` or
whichever group the UF runs as), then add the homelab's watch rules:
```bash
sudo nano /etc/audit/rules.d/homelab.rules
```
```
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/sudoers -p wa -k sudoers_changes
-a always,exit -F arch=b64 -S execve -k exec_commands
```
```bash
sudo augenrules --load
sudo auditctl -l                             # confirm all four rules loaded
```
Then point the UF at the log (`/opt/splunkforwarder/etc/system/local/inputs.conf`):
```ini
[monitor:///var/log/audit/audit.log]
index = linux-audit
sourcetype = linux:audit
disabled = false
```
```bash
sudo /opt/splunkforwarder/bin/splunk restart
```
Full walkthrough with the benign-warning caveats (`Old style watch rules are
slower`, `lost 0` vs. `failure 1`) is in Phase 2.

| Command | What it does |
|---|---|
| `sudo systemctl status auditd` | Confirm the daemon is running |
| `sudo auditctl -l` | List active rules |
| `sudo tail -f /var/log/audit/audit.log` | Live audit log |

### Sysmon (Windows endpoints)

| Command | What it does |
|---|---|
| `Get-Service Sysmon64` | Confirm the service is running |
| `sysmon64.exe -c` | Print the currently loaded config |

### Docker (OpenVAS VM)

| Command | What it does |
|---|---|
| `docker ps` | List running containers |
| `docker logs -f <container_name>` | Follow a container's logs |
| `docker compose restart` | Restart the stack in place |
| `docker compose down` | Tear the stack down (keeps volumes) |
| `docker compose up -d` | Bring the stack back up, detached |

### Wazuh (manager: `wazuh-edr.home.arpa`; agents: Desktop, Laptop, `scanner.home.arpa`, `ansible.home.arpa`)

**Download + install the agent** — manager is always `10.40.40.13`:

```bash
# Ubuntu (scanner.home.arpa, ansible.home.arpa — same repo method as the
# manager install in Phase 3, Step 2; sudo goes on the gpg side of the pipe)
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee -a /etc/apt/sources.list.d/wazuh.list
sudo apt-get update
sudo WAZUH_MANAGER="10.40.40.13" apt-get install -y wazuh-agent
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

```bash
# Fedora (Laptop)
sudo rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH
sudo tee /etc/yum.repos.d/wazuh.repo > /dev/null <<EOF
[wazuh]
gpgcheck=1
gpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH
enabled=1
name=EL-\$releasever - Wazuh
baseurl=https://packages.wazuh.com/4.x/yum/
priority=1
EOF
sudo WAZUH_MANAGER="10.40.40.13" dnf install -y wazuh-agent
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

```powershell
# Windows (Desktop)
msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER="10.40.40.13" WAZUH_REGISTRATION_SERVER="10.40.40.13"
NET START WazuhSvc
```

After any install, confirm enrollment from the manager: `sudo /var/ossec/bin/agent_control -l`
should list the new host as `Active` within a minute or two.

| Command | What it does |
|---|---|
| `sudo systemctl status wazuh-manager` | Confirm the manager is running |
| `sudo systemctl restart wazuh-manager` | Restart — required after any config change, nothing here hot-reloads |
| `sudo /var/ossec/bin/agent_control -l` | List connected agents and their status (`Active` / `Never connected` / etc.) |
| `sudo tail -f /var/ossec/logs/ossec.log` | Manager log — scan-end messages, connection errors |
| `sudo tail -f /var/ossec/logs/alerts/alerts.json` | Raw alerts, the same file Splunk's UF tails |
| `sudo wc -l /var/ossec/logs/alerts/alerts.json` | Quick check for whether anything's landing in alerts at all |

Windows agent (Desktop):

| Command | What it does |
|---|---|
| `Get-Service WazuhSvc` | Confirm the agent service is running |
| `Restart-Service WazuhSvc` | Required after any centralized syscheck/config change |

Linux agent (Laptop/Fedora, `scanner.home.arpa`, `ansible.home.arpa` — same commands regardless of distro):

| Command | What it does |
|---|---|
| `sudo systemctl status wazuh-agent` | Confirm the agent is running |
| `sudo systemctl restart wazuh-agent` | Restart after a config change |

### Ansible / Semaphore

| Command | What it does |
|---|---|
| `sudo systemctl status semaphore` | Confirm the Semaphore web UI service is running |
| `ansible all -m ping -i inventory.ini` | Confirm SSH/WinRM connectivity to every inventory host |
| `ansible-playbook -i inventory.ini playbook.yml` | Run a playbook |

### Networking / General

| Command | What it does |
|---|---|
| `ip a show ens18` | Confirm the interface's actual IP |
| `nslookup splunk.home.arpa` | Confirm DNS resolution |
| `nslookup wazuh-edr.home.arpa` | Confirm DNS resolution |
| `nslookup scanner.home.arpa` | Confirm DNS resolution |
| `nslookup ansible.home.arpa` | Confirm DNS resolution |
| `sudo ss -tlnp` | List all listening ports on this host |
| `sudo dhclient -r ens18 && sudo dhclient ens18` | Release and renew DHCP on the interface |

Port checks (`sudo ss -tlnp \| grep <port>`):

| Port | What it's for |
|---|---|
| `9997` | Splunk receiving (indexer side) |
| `8000` | Splunk web |
| `1514` | Wazuh agent event data |
| `1515` | Wazuh agent enrollment |
| `3000` | Semaphore web |

No web UI port to check for Wazuh itself — deliberately no dashboard installed (Phase 3).

### Debugging "index has data but search shows nothing"

Find every actual source/sourcetype landing in an index for a given host — use
this instead of guessing at exact `source=` strings (see Current-Stack Issue 5).
Always set the time range picker to **All Time** first:
```spl
index="windows-events" host="TDesktop*"
| stats count by source, sourcetype
```

---

## Troubleshooting Log

*Append to this as issues are encountered. Same format as the network doc.*

> ## ⚠️ DEPRECATED — Wazuh (Historical Record Only)
> Wazuh was the original centerpiece of this build (see the Pivot note at the top
> of this doc) and was dropped in favor of Splunk as the primary SIEM. The issues
> below are kept **for historical reference and lessons learned** — several of the
> underlying lessons (dashboard XML editor quirks, Windows path-length limits on
> agent-based file scanning, config-sync/restart timing) are general enough that
> they're worth remembering even outside Wazuh specifically. Nothing below reflects
> the current active stack.

### Issue 1 — Kernel panics during Ubuntu install on Bigmox (Wazuh VM)
**Symptom:** Ubuntu installer repeatedly threw kernel panics while installing on the Wazuh VM (Bigmox).
**Cause:** Bad/corrupted installer image.
**Fix:** Downloaded a fresh copy of the Ubuntu installer ISO and reran the install — completed cleanly with no further panics.

### Issue 2 — Bigmox appeared to lose networking entirely ("dead") after a reboot
**Symptom:** After a reboot, Bigmox was unreachable — looked like it wasn't getting a valid IP, initially suspected as a dead NIC or failed hardware. Drove a whole side-investigation into moving the entire SOC stack onto Minimox alone.
**Cause:** A networking config file had reverted to manual/static instead of DHCP. Most likely a package update overwrote the config at some point, and the change only actually took effect once the box was rebooted — so the reboot looked like the trigger, but the real cause had been sitting there dormant since whatever update wrote it.
**Fix:** Corrected the networking config back to DHCP. Bigmox came back immediately — not a hardware failure, not a dead NIC. No data or hardware was actually lost.
**Note:** Don't write this class of issue off as "weird" or hardware-related again before checking the actual interface config first.

### Issue 3 — Wazuh dashboard's centralized-config editor rejects valid agent.conf XML
**Symptom:** Saving a `<syscheck>` block via Management → Groups → agent.conf threw `AxiosError: mismatched tag` errors at various line/column positions, even though the XML looked structurally valid.
**Cause:** A trailing backslash immediately before a closing tag (e.g., `C:\Windows\System32\</directories>`) is misinterpreted as an escape character by Wazuh's actual XML parser — not just a dashboard UI quirk. Confirmed via Wazuh GitHub issues #34132 and #21439.
**Fix:** Remove trailing backslashes from directory paths before the closing tag.

### Issue 4 — FIM `file_limit` setting rejected with "Invalid element in the configuration"
**Symptom:** Saving a `<file_limit>` block inside `<syscheck>` produced a syntax error claiming the syscheck remote configuration was corrupted.
**Cause:** Not fully root-caused, but element ordering appears to matter — the version that saved successfully places `<file_limit>` before the `<directories>` entries.
**Fix:** Reorder `<file_limit>` before `<directories>` within `<syscheck>`.

### Issue 5 — FIM silently stopped tracking new files/folders under C:\Users
**Symptom:** `realtime="yes"` FIM was configured on `C:\Users`, but creating new files/folders there produced no alerts.
**Cause:** The Windows agent had hit the default `file_limit` of 100,000 monitored files — files beyond the cap are silently dropped from monitoring.
**Fix:** Raised `file_limit` to 200,000 via centralized config (later found to be nearly consumed by baseline scan alone — see Issue 9).

### Issue 6 — Group config file "not found" on the manager despite the dashboard showing content
**Symptom:** `sudo cat /var/ossec/etc/shared/windows/agent.conf` returned "No such file or directory," even though the dashboard's Groups editor displayed saved content for the "windows" group.
**Cause:** Group folder names are case-sensitive on the manager's Linux filesystem. The group had actually been created as `Windows` (capital W), not `windows`.
**Fix:** Use `ls -la /var/ossec/etc/shared/` to confirm exact folder casing before assuming a path.

### Issue 7 — Correct config on disk, but changes still not taking effect
**Symptom:** The Windows agent had pulled the correct, updated `agent.conf`, but FIM behavior hadn't changed.
**Cause:** Centralized config files sync to the agent automatically via keepalive checksum comparison, but the running `wazuh-agent` service only loads the file into memory on startup — `file_limit` and other syscheck settings are not hot-reloadable.
**Fix:** Restart the Wazuh agent service any time a centralized syscheck setting changes.

### Issue 8 — No FIM alerts immediately after restart despite correct, loaded config
**Symptom:** A test file created within a minute of restarting the Wazuh Windows agent produced no syscheck "added" alert.
**Cause:** Real-time FIM monitoring only activates after the initial baseline scan of all configured directories completes. Files created during the scan window are folded into the new baseline silently instead of generating an alert.
**Fix:** Watch `ossec.log` for the scan-end message before testing; create a fresh test file only after the baseline scan has finished.

### Issue 9 — Baseline scan spamming warnings and re-hitting the raised file_limit
**Symptom:** During the post-restart baseline scan, `ossec.log` filled with repeated "path too long" warnings for deeply nested paths (VS Code extensions, npm `node_modules` trees, Windows AppContainer folders under `AppData\Local\Packages`). Separately, after raising `file_limit` to 200,000, a new warning fired at 199,936/200,000 — the raised limit was nearly consumed by the baseline scan alone.
**Cause:** Confirmed: Wazuh's Windows agent enforces a hardcoded ~260-character path limit for FIM (a legacy holdover from the old Windows `MAX_PATH` restriction), acknowledged as an open, unresolved limitation by the Wazuh dev team (GitHub issues #26801, #11583). The sheer number of files under `C:\Users` from ordinary dev-tool clutter (node_modules, extension caches, package folders) is what actually drove the file count that high.
**Fix (as far as this got before the pivot):** Planned mitigation was adding `<ignore type="sregex">` entries (`node_modules`, `AppData\Local\Packages`, `.vscode\extensions`) to exclude high-churn, low-security-value folders from FIM scope rather than continuing to raise the numeric limit. Never confirmed/closed out before the decision to drop Wazuh entirely.

### Current-Stack Issue 1 — Laptop's Splunk `sourceHost` shows the Raspberry Pi's IP instead of its own
**Symptom:** Querying `index=_internal sourcetype=splunkd group=tcpin_connections` for forwarder connections showed the laptop's `sourceHost` as the Raspberry Pi's IP (10.99.99.3) instead of the laptop's actual address. The `hostname` field correctly showed `fedora`.
**Cause:** Not a misconfiguration — standard, documented Tailscale behavior. The Pi (10.99.99.3) acts as a Tailscale subnet router for remote access, and Tailscale performs SNAT (source network address translation) by default on subnet-router traffic, rewriting the source IP of anything routed through it to look like it came from the router itself once it crosses into the home network.
**Fix:** Not fixed, and not going to be — accepted as a permanent limitation. Disabling SNAT on the Pi was considered but rejected: the Pi is also a Tailscale exit node, and Tailscale's own docs plus an open GitHub issue confirm disabling SNAT on a node that's both a subnet router and exit node risks breaking exit-node internet connectivity. A dedicated second subnet-router VM (which would sidestep the issue entirely) was also considered and explicitly rejected — no new VM will be built just for this.
**Decision:** Use the `host` field (self-reported by the Universal Forwarder, unaffected by SNAT) for all laptop attribution in searches and dashboards — never `sourceHost` for this machine. Standing rule going forward for any Tailscale-connected device.

```spl
# Check forwarder connection + confirm hostname identity
index=_internal sourcetype=splunkd group=tcpin_connections (connectionType=cooked OR connectionType=cookedSSL)
| dedup hostname
| table hostname, sourceHost, fwdType, guid, os, arch

# Pull actual data from the laptop (primary query going forward)
host=fedora earliest=-1h
| sort - _time
| table _time, index, sourcetype, source

# Find the exact hostname string if unsure of casing
| metadata type=hosts
| table host, firstTime, lastTime, totalCount
| sort - lastTime
```

### Current-Stack Issue 2 — Laptop's actual OS is Fedora, not Windows
**Symptom:** The Phase 2 telemetry plan had assumed both Desktop and Laptop were Windows machines and routed both toward Sysmon + SwiftOnSecurity.
**Cause:** Wrong assumption — the laptop is running Fedora Linux, discovered while working through the Splunk forwarder attribution issue above (`hostname` field reported `fedora`, not a Windows computer name).
**Fix:** Laptop moved to the Auditd telemetry path instead of Sysmon (Phase 2 updated). OpenEDR's Windows-only agent doesn't apply to it either (Phase 3 updated) — the laptop's EDR-level coverage is out of scope for OpenEDR under the current stack; Auditd is its only detection path (see Phase 6, Test 4b).
**Open item:** Desktop's actual OS was never independently confirmed either — it was assumed Windows by default, the same unverified-assumption mistake that caused this issue in the first place. Verify Desktop's real OS before treating any Sysmon/OpenEDR step in this doc as final for that machine.

### Current-Stack Issue 3 — `splunkd` randomly crashing on `splunk.home.arpa` (intermittent, UI drops)
**Symptom:** Running Splunk manually (`sudo su - splunk`, then `/opt/splunk/bin/splunk start`), `splunkd` would randomly go offline with no clear trigger — web UI became unreachable and the process had to be manually restarted. At one point the web UI also reported an IOWait warning shortly before a crash.
**Ruled out:**
- Kernel-level OOM — `journalctl -k | grep oom` clean, `free -h` showed 15Gi free at time of crash.
- Disk exhaustion — `df -h` showed 79G free.
- systemd cgroup memory limits — initially misdiagnosed against a nonexistent unit `Splunkd.service` (wrong name); the real unit is lowercase `splunk.service` (an LSB-wrapped SysV init script from `splunk enable boot-start`), and even checked correctly it had no memory limit configured.
- Boot-start root-deprecation failure — `splunk.service` was separately found in a failed state (`Running Splunk Enterprise as root is deprecated`), but this is irrelevant to the live crashes since Splunk is being started manually via `sudo su - splunk`, not through that systemd path. Worth fixing eventually, not the cause here.
- `_metrics` internal index STMgr error (`unexpected rc=-105 from st_txn_put`) — real, observed once preceding a crash, but never proven as root cause; likely just another downstream symptom.

**Cause (part 1 — confirmed, addressed):** Live-tailing `splunkd.log` during a crash caught a Go panic — `attempt to tamper with binary: checksum changed[edge-processor-config]` — inside Splunk's internal sidecar/`TeleportSupervisorThread` architecture, tied to the `splunk_pipeline_builders` app's Edge Processor (SPL2) binary verification (OpAMP-based). This triggered a crash-restart loop that cascaded into cross-sidecar connection failures (postgres on `localhost:5435`, orchestrator on `localhost:39311`) and ultimately took down `splunkd` entirely. Note the `edge_processor_enabled = false` default in `server.conf` did **not** prevent this — the crash originated in a separate code path.
**Fix (part 1):** Disabled the entire `splunk_pipeline_builders` app. Pipeline Builders / Edge Processor is an optional SPL2 data-preprocessing feature (filter/mask/enrich/route before indexing) aimed at large production deployments trimming ingest cost — unrelated to core indexing, search, dashboards, or alerting, so disabling it has zero functional impact on this build.

**Cause (part 2 — confirmed, addressed):** Splunk kept crashing after the Pipeline Builders fix. A second live-tail caught the real underlying cause: `mongod` (Splunk's embedded KVStore, `mongod-8.0`) crashing with `exit code 4, status: PID killed by signal 4: Illegal instruction` (SIGILL), logged as `KV Store process terminated abnormally`. MongoDB 5.0+ requires AVX CPU instructions; `splunk.home.arpa` is a VM on the Bigmox Proxmox host, and Proxmox's default VM CPU type (`kvm64` / `x86-64-v2-AES`) does not expose AVX to the guest even when the physical host supports it. Every time KVStore died, dependent REST calls and the dashboard-studio install script began failing (`Connection refused ... localhost:5435`, `kvstore current status is failed`), and this instability is the more likely true root cause of the broader crash pattern — the Edge Processor panic may have just been one visible symptom riding on top of it.
**Fix (part 2):** Change the VM's CPU type in Proxmox to `host` (or `x86-64-v3`) via `qm set <vmid> --cpu host`, then a full `qm stop`/`qm start` (a guest-level reboot does not re-negotiate CPU flags — they're set when the QEMU process starts). Verified post-change from inside the guest with `grep -o 'avx[0-9a-z_]*' /proc/cpuinfo | sort -u`.

**Status:** Both fixes applied. Splunk has been stable since the CPU type change — **not yet confirmed long-term**, monitoring continues.

### Current-Stack Issue 4 — Bigmox host-level RAM failure cascading through disk, package installs, and VM startup (full rebuild saga, July 10, 2026)
**Symptom (chain, in order):**
1. `zpool status -v` on the original `wazuh-storage` pool (single `ata-WDC_WD10EZEX-00BN5A0` drive) showed `DEGRADED`, 38 CKSUM errors, a scrub that found 20 errors it couldn't repair (no redundancy on a single-disk pool), and a permanent error in `wazuh-storage/vm-100-disk-0` — the Splunk VM's own disk.
2. Splunk crashed with a SIGSEGV inside `STMgr::HandleWritable::sync` (`No memory mapped at address`, `Last errno: 2` / ENOENT) on an `IndexerTPoolWorker` thread — consistent with a memory-mapped bucket file going invalid mid-write.
3. After removing the drive and building a fresh 2 TB `Storage` pool (`ata-ST2000DX002-2DV164`, confirmed `ONLINE`, 0 errors), corruption kept happening on the new, healthy pool: `dpkg -i` on the Splunk `.deb` failed (`lzma error: compressed data is corrupt`) despite a matching file hash, and moments later a completely unrelated, freshly-fetched `binutils-x86-64-linux-gnu` package from Ubuntu's own mirrors failed the same way (`invalid tar header checksum`) — even though `apt` had already hash-verified it on download. Disk space was never the issue (86 GB free on `/`, 1.7 GB free on `/boot`).
4. After a RAM swap (see Cause below) and rebuild attempt, VM start failed with `TASK ERROR: KVM virtualisation configured, but not available.` `modprobe kvm_amd` returned `Operation not supported`, and `dmesg` showed `kvm_amd: SVM not supported by CPU 8` — one specific core reporting inconsistent CPUID versus the rest.

**Cause:**
- **Root cause: failing RAM**, confirmed via `memtest86` — 15,391+ errors on the very first pass, on a box with no XMP/DOCP overclock active (confirmed off, memory running at Auto/JEDEC speed) — ruling out an unstable-overclock explanation and pointing to genuine hardware failure. This explains the package-install corruption on the *new, healthy* pool: ZFS can only checksum-verify data it's handed. If RAM flips a bit while data is being generated/decompressed (before it reaches the block device), ZFS checksums the already-corrupt bytes and `zpool status` stays clean — while the file itself is logically broken. That's a different failure mode from a **read**-time flip during scrub, which *does* show up as CKSUM errors — same underlying bad RAM, two different symptoms depending on which side of the checksum the flip landed on.
- **Compounding, independently-confirmed issue:** the original `wazuh-storage` drive also had a physically damaged pin on its own SATA connector (not the cable or the motherboard port) — a second, genuinely separate hardware fault contributing real corruption/DEGRADED-pool symptoms on top of the RAM issue. SMART data on that drive was otherwise clean (0 reallocated/pending/uncorrectable sectors, 1 lifetime CRC error, overall PASSED) — consistent with a connector/interface-level fault rather than failing media.
- **Third, independent issue:** SVM (AMD-V) was disabled in BIOS — most likely from a defaults reset made while chasing the XMP/DOCP and RAM troubleshooting. This alone blocks all KVM-accelerated VMs on the host regardless of RAM/disk health, and explains the CPU-8-specific inconsistency (some AMD boards report CPUID inconsistently across cores while SVM is disabled).
- **Mirroring the pool was evaluated and ruled out** — not possible on this hardware. Scheduled scrubs are the standing mitigation instead (see task list / New Roadmap Items).

**Fix:**
1. Removed the damaged-connector drive entirely; built a fresh single-vdev `Storage` pool on a new 2 TB `ST2000DX002` drive (confirmed `ONLINE`, 0 errors both before and after the RAM fix).
2. Ran `memtest86`, confirmed real hardware failure, physically swapped the RAM sticks.
3. Discovered SVM disabled in BIOS, re-enabled it, and did a **full cold boot** (power off, wait, power back on) rather than a warm reboot — AMD platforms don't reliably re-apply per-core SVM MSR state on a warm reboot alone. Confirmed via `modprobe kvm_amd` succeeding and `/dev/kvm` present.
4. Splunk VM now boots; web UI confirmed reachable.

**Cross-reference:** This likely retroactively explains the `_metrics` index STMgr error (`unexpected rc=-105 from st_txn_put`) noted in Current-Stack Issue 3 above, which that entry explicitly flagged as "never proven as root cause; likely just another downstream symptom" — bad RAM is now a strong candidate for that symptom too. This does **not** change Issue 3's own root cause or fix (missing AVX exposure from the `kvm64` CPU type, fixed by switching to `host`) — that was independently confirmed via direct `/proc/cpuinfo` verification and remains valid; it's a separate, compounding problem on the same box, not an alternate explanation for it.

**Status:** RAM swapped, SVM enabled, Splunk web UI confirmed loading. Not yet confirmed long-term stable — hold off on declaring the rebuild done until Splunk, then Ansible/Semaphore and SOAR, have run clean for a few days. A full multi-pass `memtest86` run on the replacement RAM is still worth doing for formal confirmation (a short run without errors is a good sign, not a guarantee).

### Current-Stack Issue 5 — Sysmon events never reach `windows-events`, despite valid config (Security/System from the same input work fine)
**Symptom:** After Phase 2 Sysmon setup on `TDesktop`, the build plan's verification search
```spl
index="windows-events" source="WinEventLog:Microsoft-Windows-Sysmon/Operational"
```
returned no results, even with the time range set to All Time. A broad, unfiltered `index="windows-events"` search *did* return results, which initially looked like a plain search-syntax/time-range issue. The command that actually exposed the real problem:
```spl
index="windows-events" host="TDesktop*"
| stats count by source, sourcetype
```
Output only showed `WinEventLog:Security` (4,671 events) and `WinEventLog:System` (577 events) — `WinEventLog:Microsoft-Windows-Sysmon/Operational` was completely absent. Confirms Sysmon events genuinely aren't being ingested; this was never a search-side problem.

**Ruled out:**
- Sysmon not running — `Get-Service Sysmon64` shows `Running`.
- Sysmon not generating events — `Get-WinEvent -LogName 'Microsoft-Windows-Sysmon/Operational' -MaxEvents 5` showed live Process Create events.
- UF not connected — `splunk.exe list forward-server` shows `splunk.home.arpa:9997` as an active forward.
- Bad `inputs.conf` — confirmed via `splunk.exe btool inputs list --debug`: `[WinEventLog://Microsoft-Windows-Sysmon/Operational]`, `disabled = false`, `index = windows-events`, loaded from `etc\system\local\inputs.conf` — same file, same pattern as the working Security/System stanzas.
- Receiving/index config on the Splunk side — port 9997 confirmed listening, `windows-events` index confirmed to exist and actively receive data (just not from Sysmon).

**Cause (confirmed):** `splunkd.log` on TDesktop showed the real error:
```
WinEventLogChannel::subscribeToEvtChannel: Could not subscribe to Windows Event Log channel
'Microsoft-Windows-Sysmon/Operational'
WinEventLogChannel::init: Init failed, unable to subscribe ... errorCode=5
```
`errorCode=5` is Windows' `ERROR_ACCESS_DENIED`. Checked what account the UF service actually runs as:
```powershell
Get-CimInstance Win32_Service -Filter "Name='SplunkForwarder'" | Select-Object Name, StartName
```
→ `NT SERVICE\SplunkForwarder` — a Windows **virtual service account** (the modern Splunk UF installer's default, not Local System and not a real user). Security/System channels include the generic "SERVICE" well-known group in their default ACLs, so any service — including a virtual service account — can read them. Sysmon's installer (`sysmon64.exe -i`) sets a narrower channel ACL (SYSTEM + Administrators + whichever account ran the install) that does **not** include the generic service group, so the virtual account was flatly denied on that one channel while Security/System worked fine.

**Fix:** Switched the service to run as Local System, which bypasses the channel ACL restriction entirely:
```powershell
sc.exe config SplunkForwarder obj= "LocalSystem"
Restart-Service SplunkForwarder
```
Confirmed working via:
```spl
index="windows-events" source="WinEventLog:Microsoft-Windows-Sysmon/Operational"
```
(Least-privilege alternative, not used here: keep the virtual service account and explicitly grant it access to just the Sysmon channel via `wevtutil sl` with the account's resolved SID — more correct for a hardened environment, more fragile to get right by hand. Local System was the pragmatic call for this homelab.)

**Status:** Resolved.

### Current-Stack Issue 6 — OpenEDR Windows agent (`edrsvc`) installed but service disabled, won't start
**Symptom:** On TDesktop, double-clicking `OpenEdrAgent.msi` completed without obvious error, but the interactive installer likely never applied the `EDRSVRADDRESS=10.40.40.13` property (that's only reliably set via command-line `msiexec /i ... EDRSVRADDRESS=...`). Attempting to reinstall via command line initially failed with "the installation package could not be opened" — caused by running `msiexec` without the full path to the `.msi` file (it wasn't in the current directory). After locating the file in `Downloads` and using the full path, the reinstall completed, but `Get-Service edrsvc` showed `Stopped`, and `Start-Service edrsvc` failed with a generic, unhelpful error.

**Cause:** `net start edrsvc` surfaced the real reason — Windows error 1058 ("the service cannot be started ... because it is disabled"). `sc.exe qc edrsvc` confirmed `START_TYPE: 4 DISABLED`. (Also corrected while investigating: the real install path is `C:\Program Files\COMODO\EdrAgentV2\`, not `C:\Program Files\OpenEdr\EdrAgentV2` as originally assumed from the vendor's general documentation.) Root cause of *why* it installed disabled wasn't fully confirmed — leading theory is the installer intentionally ships the service disabled until a valid server address is detected, which the GUI double-click install path likely never provided.

**Fix:**
```powershell
sc.exe config edrsvc start= auto
Start-Service edrsvc
Get-Service edrsvc
```
(Note the required space after `start=` — `sc.exe` syntax quirk.)

**Status:** Superseded by Current-Stack Issue 7 below. The service fix here worked (confirmed `edrsvc` stayed `Running`, and telemetry briefly confirmed flowing to the backend) — but a more serious problem showed up right after, which is why the agent ended up fully uninstalled again. Keeping this entry since the disabled-service fix itself is still valid/reusable if this agent goes on a different machine.

### Current-Stack Issue 7 — OpenEDR Windows agent kills Electron apps (Obsidian, Claude Desktop/Cowork) on TDesktop
**Symptom:** After the Issue 6 fix got `edrsvc` running and telemetry confirmed flowing from TDesktop to the OpenEDR backend, Electron-based apps — Obsidian, Claude Desktop (Cowork) — started getting killed instantly on launch. This blocked using Cowork itself on that machine, so this round of troubleshooting had to happen in a separate session.

**Investigation path:**
1. Confirmed `edrsvc` running and tamper-protected (`Stop-Service`/`net stop` both blocked by the agent itself).
2. Checked the OpenEDR Wekan/OrientDB backend (Triage board) — real telemetry was confirmed flowing from TDesktop; the "Organisation" card needed moving from the Profiling list to the Detection list (Wekan is Kanban-based — swimlanes/lists/cards, not a traditional dashboard) to activate alerting.
3. Hit an unrelated Wekan access issue — logged in as a different account than the one the installer seeded; fixed by adding the logged-in account as a board member directly via MongoDB rather than recovering the original seeded account.
4. Standard removal failed: `msiexec /x {GUID}` → Error 1605 ("product not currently installed") — an MSI database inconsistency, likely because the actual working install method bypasses normal MSI registration (see Cause below).

**Cause (confirmed):** The real, correct install path for this agent is **not** the raw `OpenEdrAgent.msi` from ComodoSecurity's GitHub releases — it's a PowerShell installer from a third repo, `jymcheong/openedrClient`, pointed at the backend's client-config port (`10.40.40.13:8888`, the same `clientconfighosting` container found while chasing Issue 6):
```powershell
[scriptblock]::Create((New-Object System.Net.WebClient).DownloadString('https://raw.githubusercontent.com/jymcheong/openedrClient/master/install.ps1')).Invoke()
```
This is genuinely the intended installer — `EdrAgentV2`/`edrsvc`/`edrdrv.sys` is not a mistaken or rogue install; jymcheong's OpenEDR project is built directly on Comodo's real, open-sourced EDR engine (kernel driver included), so the Comodo branding throughout (`C:\Program Files\COMODO\EdrAgentV2\`, "Comodo EDR Service") is expected. This same script also silently installs **Sysmon and NXLog** alongside the EDR agent.

The actual problem: the agent's local policy file (`evm.local.src`) ships with an extremely narrow default trust allowlist —
```
knownTrustedPaths: git.exe, devenv.exe, cl.exe, msbuild.exe
```
— four developer tools, nothing else. With no management console connected to broaden this policy, the agent kills anything it doesn't recognize, which includes essentially all consumer Electron apps (Obsidian, Claude Desktop/Cowork, likely Discord/VS Code/Slack too).

**Fix:** Removed via the project's own paired uninstaller (run elevated):
```powershell
[scriptblock]::Create((New-Object System.Net.WebClient).DownloadString('https://raw.githubusercontent.com/jymcheong/openedrClient/master/uninstall.ps1')).Invoke()
```
If it hangs, the project documents killing stuck processes and re-running with a `-u force` flag. Verify all three components installed alongside are actually gone, not just the EDR piece:
```powershell
Get-Service edrsvc -ErrorAction SilentlyContinue
Get-ChildItem "C:\Program Files\COMODO\" -ErrorAction SilentlyContinue
Get-Service Sysmon64 -ErrorAction SilentlyContinue
```
All three should return empty/not-found. Confirmed Obsidian and Claude Desktop/Cowork launch normally again after this.

**Open decision — not yet made:** whether/how to run OpenEDR on TDesktop again.
- **Option A:** Configure the Organisation board's actual policy settings on the backend to broaden the trust allowlist *before* redeploying to the desktop.
- **Option B:** Don't run the OpenEDR client on the daily-driver desktop at all — reserve it for a dedicated, lower-stakes test/lab VM (this VM already exists as a concept — see Phase 6, "Why a Test VM, Not Your Daily Driver"), and rely on Sysmon + Auditd + Splunk (already the decided telemetry stack) for the desktop's actual day-to-day monitoring.

**Status:** Uninstalled, TDesktop confirmed back to normal. Decision above still open — revisit before attempting to reinstall.

*(Log new issues below this line as they come up, same Symptom/Cause/Fix format.)*

---

## Sources & References

**Splunk**
- Splunk Enterprise download: https://www.splunk.com/en_us/download/splunk-enterprise.html
- Universal Forwarder download: https://www.splunk.com/en_us/download/universal-forwarder.html
- License types (Free vs Enterprise vs Trial): https://docs.splunk.com/Documentation/Splunk/latest/Admin/TypesofSplunklicenses
- SPL search reference: https://docs.splunk.com/Documentation/Splunk/latest/SearchReference/WhatsInThisManual
- HTTP Event Collector: https://docs.splunk.com/Documentation/Splunk/latest/Data/UsetheHTTPEventCollector
- Dashboard Studio: https://docs.splunk.com/Documentation/Splunk/latest/DashStudio/dsOverview
- Splunk Add-on for Microsoft Sysmon: https://splunkbase.splunk.com/app/1914
- Splunk Add-on for Linux (auditd config): https://docs.splunk.com/Documentation/AddOns/released/Linux/Configure4

**Splunk SOAR (Future)**
- System requirements: https://help.splunk.com/en/splunk-soar/soar-on-premises/install-and-upgrade-soar-on-premises/8.5.0/system-requirements
- Installation methods: https://help.splunk.com/en/splunk-soar/soar-on-premises/install-and-upgrade-soar-on-premises/8.4.0/get-splunk-soar-on-premises/installation-methods-for-splunk-soar-on-premises

**Sysmon / Windows Telemetry**
- Sysmon (Microsoft Sysinternals): https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
- SwiftOnSecurity Sysmon config: https://github.com/SwiftOnSecurity/sysmon-config
- Windows Security Event ID Encyclopedia: https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/

**Auditd / Linux Telemetry**
- Configure auditd for Splunk SIEM: https://binadit.com/tutorials/configure-linux-audit-siem-integration-splunk

**OpenEDR**
- ComodoSecurity/openedr: https://github.com/ComodoSecurity/openedr
- jymcheong/OpenEDR ("Free EDR" fork, backend): https://github.com/jymcheong/OpenEDR
- jymcheong/openedrClient (Windows agent installer): https://github.com/jymcheong/openedrClient

**OpenVAS / Greenbone**
- Greenbone Community Containers documentation: https://greenbone.github.io/docs/latest/container/index.html
- OpenVAS/GVM install walkthrough: https://serverspace.io/support/help/how-to-install-and-use-openvas-gvm-on-ubuntu/

**Ansible + Semaphore**
- Semaphore UI installation guide: https://semaphoreui.com/docs/admin-guide/installation
- Semaphore install walkthrough: https://computingforgeeks.com/install-semaphore-ubuntu-debian/
- Ansible documentation: https://docs.ansible.com/

**Docker**
- Docker install on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Docker Compose reference: https://docs.docker.com/compose/

**Validation / Attack Simulation**
- EICAR test file: https://www.eicar.org/download-anti-malware-testfile/
- Nmap: https://nmap.org/
- Atomic Red Team (future use): https://github.com/redcanaryco/atomic-red-team

**Wazuh (Deprecated — kept for historical troubleshooting reference only)**
- Wazuh documentation: https://documentation.wazuh.com/current/
- FIM path-length issue: https://github.com/wazuh/wazuh/issues/26801

---

*Document started: July 2026 | Last updated: July 10, 2026 — added Current-Stack
Issue 4 (RAM/disk/SVM troubleshooting saga)*
*Cross-references: Homelab-Network-Documentation.md, Homelab-Server-VM-Specs.md*
