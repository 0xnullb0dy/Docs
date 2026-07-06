# Homelab SOC Build Plan — EDR + SIEM

> **Purpose**
> This is the active planning runbook and progress tracker for building a self-hosted
> Security Operations Center (SOC) layer on top of the existing UniFi/VLAN network.
> It covers Wazuh (EDR/SIEM), Splunk (SIEM), and OpenEDR (dedicated EDR console),
> all running on-prem with zero cloud dependencies.
>
> **Cross-reference:** This document is a direct continuation of
> `Homelab-Network-Documentation.md`. All IP addresses, VLANs, hostnames, and
> firewall rules referenced here come from that document. Read it first if anything
> here seems out of context.

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
Block Servers to Management  | VLAN 40             | VLAN 99                 | Block  | All ← !!!  | NEEDS FIX (Phase 0)
```

> **!! THE ONE THING TO DO BEFORE ANYTHING ELSE !!**
> "Block Servers to Management" is currently set to Connection State = **All**.
> This will break Raspberry Pi ↔ Wazuh agent communication in the exact same way
> that Issue 10 broke Minimox ↔ Main LAN. The Pi INITIATES to the Wazuh manager
> (Management → Servers, allowed), but the manager's REPLIES travel back
> Servers → Management and will be dropped.
> Fix: Change Connection State to **New** on this rule before enrolling any agents.
> After changing, CLOSE and REOPEN the policy to confirm it actually saved
> (per Issue 10 lesson — the first silent-fail is documented in the network doc).

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

## Planned IP / DNS Allocations (New — This Build)

Add these to UCG-Ultra (Settings → Networks → VLAN 40 → DHCP Reservations
and Settings → DNS → Local Records) **during Phase 0**, before any VM is created.
Follow the same `home.arpa` convention used throughout the network.

```
Hostname              | IP            | VLAN | VM Host  | Purpose
----------------------|---------------|------|----------|------------------------
wazuh.home.arpa       | 10.40.40.12   | 40   | Bigmox   | Wazuh all-in-one + Splunk
openedr.home.arpa     | 10.40.40.13   | 40   | Minimox  | OpenEDR backend (Docker/ELK)
```

> **Why split across hosts?**
> Wazuh indexer + Splunk indexer + OpenEDR's ELK stack = three separate indexing
> engines running simultaneously. Stacking all three on one Proxmox host is likely
> to starve everything. Bigmox (direct LAN 2 connection) handles the primary SOC
> layer; Minimox handles the OpenEDR backend. This keeps the load balanced and
> means one host going down doesn't kill your entire monitoring capability.

---

## SOC Traffic Flow (After Full Build)

```
                        VLAN 40 — Servers (10.40.40.0/24)
 ┌──────────────────────────────────────────────────────────────────┐
 │                                                                  │
 │   ┌──────────────────────┐      ┌─────────────────────────────┐ │
 │   │  wazuh.home.arpa     │      │  openedr.home.arpa          │ │
 │   │  10.40.40.12         │      │  10.40.40.13                │ │
 │   │  (Bigmox VM)         │      │  (Minimox VM)               │ │
 │   │                      │      │                             │ │
 │   │  Wazuh Server        │      │  Docker stack:              │ │
 │   │  Wazuh Indexer       │      │  Elasticsearch              │ │
 │   │  Wazuh Dashboard     │      │  Logstash                   │ │
 │   │  Splunk Enterprise   │      │  Kibana                     │ │
 │   │  Splunk Univ. Fwd.   │      │  OpenEDR backend            │ │
 │   └─────────┬────────────┘      └────────────┬────────────────┘ │
 │             │                                │                  │
 └─────────────┼────────────────────────────────┼──────────────────┘
               │                                │
               │  Wazuh agents check in         │  Filebeat ships
               │  (agent initiates → manager)   │  telemetry (Filebeat → ELK)
               │                                │
 ┌─────────────▼────────────────────────────────▼──────────────────┐
 │                                                                  │
 │  VLAN 1 (Main LAN)     VLAN 99 (Mgmt)    VLAN 20 (IoT)         │
 │  Desktop 10.10.10.x    Pi 10.99.99.3     iPhone/iPad            │
 │  Laptop  10.10.10.x                      (Wazuh agent           │
 │                                           optional/future)       │
 └──────────────────────────────────────────────────────────────────┘
```

> **Firewall note on agent traffic direction**
> Wazuh agents always INITIATE the connection to the manager — the manager never
> reaches out to agents. This means:
> - Desktop/Laptop (VLAN 1) → wazuh.home.arpa (VLAN 40): Allowed by default
>   (Internal zone, same zone = free communication)
> - Pi (VLAN 99) → wazuh.home.arpa (VLAN 40): INITIATES from Mgmt → Servers
>   (allowed), replies go Servers → Mgmt (blocked if Conn State = All — FIX FIRST)

---

## Phase 0 — Network Prep

**Goal:** Everything is in place before a single tool is installed.
**Estimated time:** 2–3 hours
**Host:** UCG-Ultra web UI + Proxmox (Bigmox and Minimox)

### Checklist

- [x] **Fix the firewall rule** — UCG-Ultra → Settings → Security → Zone-Based Firewall
      - Find "Block Servers to Management" policy
      - Change Connection State from **All** → **New**
      - Save, close the policy, reopen it, confirm the change persisted
        (silent save failure is a documented gotcha — see Issue 10 in network doc)

- [x] **Add DHCP reservations for new VMs (before creating them)**
      - Settings → Networks → VLAN 40 (Servers) → DHCP → Add Reservation
      - `10.40.40.12` → MAC of Wazuh VM's NIC (set after VM creation, or reserve
        after first boot and move to static via reservation immediately)
      - `10.40.40.13` → MAC of OpenEDR VM's NIC (same approach)

- [x] **Add Local DNS records**
      - Settings → DNS → Local Records
      - `wazuh.home.arpa`  → `10.40.40.12`
      - `openedr.home.arpa` → `10.40.40.13`
      - Double-check both records after saving with `nslookup wazuh.home.arpa`
        from the desktop (not from the VM itself — see Issue 12 in network doc)

- [x] **Create Wazuh VM on Bigmox**
      - OS: Ubuntu 24.04 LTS (Wazuh requirement — supported OS list below)
      - Resources: 8 vCPU, 16 GB RAM (bumped up from the original 4 vCPU/8 GB minimum
        after real-world install attempts showed the dashboard's one-time bundling
        step alone can exhaust 12 GB — see Troubleshooting Log below)
      - Disk: 50 GB minimum (90 days retention for small lab)
      - Storage backend: **ZFS** on the spare 1 TB drive, not LVM-Thin. Switched
        after suspecting LVM-Thin (`Wuzzah-Storage`) as a contributing factor in
        the corruption issues hit during earlier install attempts — went with
        Proxmox's own recommended standard instead. Re-provisioning to confirm.
      - Network bridge: `vmbr0` (VLAN 40 — same bridge Bigmox uses for its own traffic)
      - Access Bigmox UI at `https://bigmox.home.arpa:8006`

- [x] **Create OpenEDR VM on Minimox**
      - OS: Ubuntu 22.04 LTS (Docker tested/confirmed on this version per guides)
      - Resources: 4 vCPU, 8 GB RAM, 100 GB disk
      - Network bridge: `vmbr0` (VLAN 40)
      - Access Minimox UI at `https://minimox.home.arpa:8006`

- [x] **Verify both VMs get the right IPs** — after first boot:
      ```bash
      ip a show ens18    # or eth0/ens3 depending on Proxmox NIC naming
      ```
      Should show `10.40.40.12` (Wazuh VM) or `10.40.40.13` (OpenEDR VM).
      If DHCP didn't pick up the reservation yet, release/renew:
      ```bash
      sudo dhclient -r ens18 && sudo dhclient ens18
      ```

- [x] **Confirm hostnames resolve from the desktop before moving to Phase 1**
      ```
      nslookup wazuh.home.arpa
      nslookup openedr.home.arpa
      ```
      Expected results: `10.40.40.12` and `10.40.40.13` respectively.
      If they don't resolve, check DNS records in UniFi before proceeding.

### Sources
- Wazuh supported OS list: https://documentation.wazuh.com/current/installation-guide/wazuh-server/index.html
- Wazuh hardware requirements: https://documentation.wazuh.com/current/quickstart.html
- Proxmox VM creation: https://pve.proxmox.com/wiki/Qemu/KVM_Virtual_Machines

---

## Phase 1 — Wazuh (All-in-One)

**Goal:** Wazuh server, indexer, and dashboard running on the Wazuh VM; agents
enrolled on all major devices; meaningful alerts visible in the dashboard.
**Estimated time:** 4–6 hours
**VM:** `wazuh.home.arpa` (10.40.40.12, on Bigmox)

### What Wazuh Is

Wazuh is a free, open-source unified XDR and SIEM platform. Its three central
components are normally deployed on a single host for homelab use:

```
Wazuh Agent (on each endpoint)
        │
        │  TCP 1514 (agent traffic)
        │  TCP 1515 (agent enrollment)
        ▼
Wazuh Server   ← analyzes events, runs rules, generates alerts
        │
        │  Filebeat (ships alerts to indexer)
        ▼
Wazuh Indexer  ← stores and indexes all alert data (OpenSearch-based)
        │
        ▼
Wazuh Dashboard ← web UI for search, alerts, SCA, FIM, vuln detection
```

The "assisted installer" does all three in a single command and takes about
10–15 minutes to run. This is the recommended path for homelab.

### Checklist

**Installation**

- [x] SSH into the Wazuh VM (`ssh user@wazuh.home.arpa` or `ssh user@10.40.40.12`)
- [x] Download and run the assisted installer:
      ```bash
      curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
      sudo bash ./wazuh-install.sh -a
      ```
      The `-a` flag = all-in-one (server + indexer + dashboard on one host).
      The installer will print admin credentials when finished. **Save these.**

- [x] Access the dashboard at `https://wazuh.home.arpa` (port 443, not 8006 — that's Proxmox). Accept the self-signed cert warning in the browser.

- [ ] **Disable auto-updates immediately after install** to prevent accidental upgrades breaking the environment (Wazuh's own recommendation):
      ```bash
      # Ubuntu/Debian:
      sudo sed -i "s/^deb/#deb/" /etc/apt/sources.list.d/wazuh.list
      sudo apt-get update
      ```

**Agent Enrollment**

Enroll endpoints in this order: Desktop first (easiest, Windows), then Laptop,
then Raspberry Pi (most likely to hit the firewall issue if Phase 0 was skipped).

- [x] **Desktop (Windows)**
      In the Wazuh dashboard: Agents → Deploy New Agent → Windows
      The UI generates a full install command. Paste it into an elevated PowerShell.
      Verify in the dashboard that the agent shows as Active.

- [x] **Laptop (Windows or Linux — adjust accordingly)**
      Same process as desktop.

- [x] **Raspberry Pi (Linux/Debian)**
      Same enrollment command process. Before enrolling, confirm the Phase 0 firewall
      fix is in place (Block Servers to Management = New, not All).
      After enrollment, verify from the Pi:
      ```bash
      sudo systemctl status wazuh-agent
      sudo journalctl -u wazuh-agent -n 30 --no-pager
      ```
      And verify from the desktop that the Pi agent appears Active in the dashboard.

- [x] **Test VM (optional — recommended for lab practice)**
      Spin up a small Ubuntu or Kali VM on Bigmox or Minimox, enroll it as a Wazuh
      agent. This becomes your attack target in Phase 5 validation without touching
      production machines.

**Core Capabilities to Enable**

- [x] **File Integrity Monitoring (FIM)**
      FIM watches files/directories for unauthorized changes — a core EDR behavior.
      Edit `/var/ossec/etc/ossec.conf` directly on each agent (primary method per
      Wazuh docs), restart with `systemctl restart wazuh-agent` to apply.
      **Correction:** there is no "Management → Configuration → Syscheck" menu in
      the dashboard — that was wrong in an earlier draft of this doc. For centralized config instead of per-agent editing, use **Management → Groups** → select the agent group → Edit the group's `agent.conf` → add a `<syscheck>` block inside
      `<agent_config>`. Centralized config takes precedence over local `ossec.conf` if the same directory is defined in both.
      Key paths to monitor on Linux agents:
      ```
      /etc/passwd
      /etc/shadow
      /etc/sudoers
      /var/ossec/etc/
      /tmp/
      ```
      Key paths on Windows agents:
      ```
      HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services
      C:\Windows\System32\
      C:\Users\
      ```

      > **Real-world implementation note (see Troubleshooting Log Issues 3–9):**
      > Instead of `os=` attributes inside one shared `agent.conf`, we ended up
      > creating two separate agent groups — `Windows` and `Linux` — each with its
      > own `agent.conf`, after the dashboard's XML validator choked on multi-block
      > `os=` configs (a real parser bug, not just a UI issue — Issue 3). Group
      > folder names are case-sensitive on the manager's filesystem
      > (`/var/ossec/etc/shared/<GroupName>/`) — double-check exact casing if a
      > file ever looks "missing" (Issue 6).
      >
      > If monitoring a broad Windows tree like `C:\Users`, proactively raise
      > `file_limit` above the 100,000 default — it's easy to hit that cap
      > silently (Issue 5), since dev environments (node_modules, VS Code
      > extensions, AppData caches) inflate file counts fast. Expect heavy
      > "path too long" warnings on deeply nested Windows paths too — the
      > Windows agent has a hardcoded ~260-character path limit that's still
      > unresolved upstream (Issue 9).
      >
      > Any `file_limit`/syscheck config change requires a full agent service
      > restart to take effect — it does not hot-reload (Issue 7). Real-time
      > alerts also won't fire until the post-restart baseline scan fully
      > completes, which can take a while on large trees (Issue 8).

- [x] **Security Configuration Assessment (SCA)**
      Runs CIS benchmark-style checks against each endpoint.
      Enabled by default on new agents. Review the SCA dashboard tab to see
      pass/fail posture for each enrolled machine.

- [x] **Vulnerability Detection**
      Enabled in dashboard → Vulnerabilities (requires indexer to have processed
      at least one full scan cycle — give it 30–60 minutes after agent enrollment).

- [ ] **Active Response (optional, but worth knowing exists)**
      Wazuh can auto-block IPs, kill processes, etc. Leave disabled for now —
      understand what it's firing on before enabling auto-remediation in a lab
      that includes your daily-driver desktop.

### Key Ports (for reference / future firewall rules)

```
Port  | Protocol | Direction           | Purpose
------|----------|---------------------|-----------------------------------
1514  | TCP/UDP  | Agent → Manager     | Agent event traffic
1515  | TCP      | Agent → Manager     | Agent auto-enrollment
1516  | TCP      | Manager ↔ Manager   | Wazuh cluster (not needed, single node)
443   | TCP      | Browser → Dashboard | Wazuh web UI
9200  | TCP      | Internal            | Wazuh Indexer (OpenSearch) — localhost only
```

### Sources
- Wazuh Quickstart (assisted install): https://documentation.wazuh.com/current/quickstart.html
- Wazuh Agent Installation (Linux): https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-linux.html
- Wazuh Agent Installation (Windows): https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html
- FIM documentation: https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/index.html
- SCA documentation: https://documentation.wazuh.com/current/user-manual/capabilities/sec-config-assessment/index.html
- Vulnerability detection: https://documentation.wazuh.com/current/user-manual/capabilities/vulnerability-detection/index.html

---

## Phase 2 — Splunk (Self-Hosted, On Wazuh VM)

**Goal:** Splunk Enterprise running on the same VM as Wazuh, with Splunk Universal
Forwarders shipping Windows event logs from the desktop/laptop directly to Splunk
(independent of Wazuh). Wazuh → Splunk integration comes in Phase 3.
**Estimated time:** 4–6 hours
**VM:** `wazuh.home.arpa` (10.40.40.12) — same box as Wazuh

### What Splunk Is (and What the Free License Means for You)

Splunk is a SIEM/log aggregation platform built around its own Search Processing
Language (SPL). For a homelab, the free download gives you:

```
License Tier    | Ingest Limit | Alerting | Auth/Users | Duration
----------------|-------------|----------|------------|------------------
Enterprise Trial | Unlimited   | YES ✓    | YES ✓      | 60 days then auto-converts
Free (converted) | 500 MB/day  | NO ✗     | NO ✗       | Perpetual after trial
```

> **Important:** Do all alerting work (Phase 2 dashboards and alert rules) inside
> the 60-day trial window. After it converts to Free, alerting is disabled. The
> 500 MB/day limit is fine for a homelab with a handful of agents — your daily
> log volume from 3–5 endpoints will be well under that. The lack of authentication
> on Free is acceptable since Splunk is on VLAN 40 (Servers), not exposed to the
> internet or untrusted VLANs.

### Checklist

**Installation**

- [ ] Download Splunk Enterprise from https://www.splunk.com/en_us/download/splunk-enterprise.html
      (requires a free Splunk account)
      Direct download on the Wazuh VM:
      ```bash
      wget -O splunk.deb 'https://download.splunk.com/products/splunk/releases/\
      9.x.x/linux/splunk-9.x.x-linux-amd64.deb'
      sudo dpkg -i splunk.deb
      ```
      Replace the URL with the current version from the download page.

- [ ] Start Splunk and accept the license:
      ```bash
      sudo /opt/splunk/bin/splunk start --accept-license
      sudo /opt/splunk/bin/splunk enable boot-start
      ```

- [ ] Access the Splunk web UI at `http://wazuh.home.arpa:8000`
      Default port is 8000 (not 443, not 8006 — a different port from everything
      else in this lab — add it to your mental port map).

- [ ] Set admin credentials when prompted on first login.

**Universal Forwarder on Endpoints**

The Universal Forwarder (UF) is a lightweight agent that ships logs directly to
Splunk without running Splunk locally. It's free with no license limit on the
forwarder itself.

- [ ] **Configure Splunk to receive forwarded data first** (indexer receiving port):
      Splunk Web → Settings → Forwarding and Receiving → Receive Data → Add New
      Enter port `9997` → Save

- [ ] **Create a `windows-events` index** (or name of your choice):
      Settings → Indexes → New Index → Name: `windows-events` → Save
      Indexes must exist before the forwarder can write to them.
      A forwarder writing to a non-existent index fails silently — no error shown.

- [ ] **Install the Universal Forwarder on Desktop (Windows)**:
      Download from: https://www.splunk.com/en_us/download/universal-forwarder.html
      During install, point it at `wazuh.home.arpa:9997` as the indexer.
      After install, configure which logs to ship:
      ```
      C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
      ```
      Add:
      ```ini
      [WinEventLog://Security]
      index = windows-events
      disabled = false

      [WinEventLog://System]
      index = windows-events
      disabled = false

      [WinEventLog://Application]
      index = windows-events
      disabled = false
      ```
      Restart the forwarder service after editing.

- [ ] **Repeat UF install on Laptop**

- [ ] **Verify data is flowing in Splunk:**
      Search & Reporting → search: `index="windows-events"` → should return events

**SPL Basics to Learn During This Phase**

SPL (Search Processing Language) is how you query data in Splunk.
Spend time here — this is the SIEM skill that matters for job experience.

```spl
# Find all events in an index
index="windows-events"

# Filter to a specific event ID (4625 = failed logon)
index="windows-events" EventCode=4625

# Count failed logons by source host
index="windows-events" EventCode=4625 | stats count by host

# Find events in a time window
index="windows-events" EventCode=4625 earliest=-1h latest=now

# Table output with specific fields
index="windows-events" EventCode=4625
| table _time, host, Account_Name, Failure_Reason

# Alert-style search: more than 5 failed logons from same account in 10 min
index="windows-events" EventCode=4625
| bucket _time span=10m
| stats count by _time, Account_Name
| where count > 5
```

- [ ] Build at least 3 dashboards during the 60-day trial:
      - Failed logon attempts over time (EventCode 4625)
      - Logon success events by user (EventCode 4624)
      - Process creation events (EventCode 4688, requires audit policy enabled)

- [ ] Create at least 1 saved alert during the 60-day trial:
      Search → Save As → Alert → set trigger condition → save

### Enable Windows Audit Policy for Better Logs

By default, Windows doesn't log process creation. Enable it on your desktop/laptop:

```
Group Policy Editor (gpedit.msc) → Computer Configuration
  → Windows Settings → Security Settings → Advanced Audit Policy Configuration
    → System Audit Policies → Detailed Tracking
      → Audit Process Creation: Enable (Success)

Also enable command line logging in registry:
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit
  ProcessCreationIncludeCmdLine_Enabled = 1 (DWORD)
```

### Key Ports

```
Port  | Protocol | Direction                | Purpose
------|----------|--------------------------|----------------------------------
8000  | TCP      | Browser → Splunk         | Splunk Web UI
9997  | TCP      | UF → Splunk              | Forwarder receiving port
8089  | TCP      | Internal/API             | Splunk management port
8088  | TCP      | Wazuh server → Splunk    | HTTP Event Collector (Phase 3)
```

### Sources
- Splunk Enterprise download: https://www.splunk.com/en_us/download/splunk-enterprise.html
- Universal Forwarder download: https://www.splunk.com/en_us/download/universal-forwarder.html
- Splunk Free license limits: https://docs.splunk.com/Documentation/Splunk/latest/Admin/TypesofSplunklicenses
- SPL search reference: https://docs.splunk.com/Documentation/Splunk/latest/SearchReference/WhatsInThisManual
- Windows Security Event IDs reference: https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/
- Windows audit policy guide: https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/advanced-security-audit-policy-settings

---

## Phase 3 — Wazuh → Splunk Integration

**Goal:** Wazuh alerts flow into Splunk automatically. The Splunk `wazuh-alerts`
index gets populated, the prebuilt Wazuh dashboards for Splunk are imported, and
a search in Splunk confirms end-to-end data flow.
**Estimated time:** 2–4 hours
**VM:** `wazuh.home.arpa` (10.40.40.12) — configuration on both services, same host

### How It Works

```
Wazuh Manager
  │
  │ writes alerts to:
  │ /var/ossec/logs/alerts/alerts.json
  │
  ▼
Splunk Universal Forwarder
  (installed on the Wazuh VM itself)
  │
  │ tails alerts.json
  │ ships to Splunk indexer on port 9997
  │
  ▼
Splunk Indexer (wazuh-alerts index)
  │
  ▼
Splunk Dashboard / Search
```

> **Why a Forwarder on the Wazuh VM?**
> The Splunk Universal Forwarder is free and lightweight. It reads the Wazuh alerts
> JSON file directly and ships it to Splunk. This avoids needing to run Logstash
> (another tool, more complexity). The forwarder and Splunk indexer are on the same
> host, so the "network" hop is just localhost.

### Checklist

**Configure Splunk to Accept Wazuh Data**

- [ ] In Splunk Web, create the `wazuh-alerts` index (if not already done):
      Settings → Indexes → New Index → Name: `wazuh-alerts` → Save
      **This index must exist before the forwarder starts — or data is silently dropped.**

- [ ] Confirm the receiving port (9997) is already configured from Phase 2.
      If Splunk is running on the same host as the forwarder, verify it's listening:
      ```bash
      sudo ss -tlnp | grep 9997
      ```

**Install Splunk Universal Forwarder on the Wazuh VM**

- [ ] Download and install the Linux UF on the Wazuh VM:
      ```bash
      wget -O splunkforwarder.deb \
        'https://download.splunk.com/products/universalforwarder/releases/\
      9.x.x/linux/splunkforwarder-9.x.x-linux-amd64.deb'
      sudo dpkg -i splunkforwarder.deb
      ```

- [ ] Configure the forwarder to point at the local Splunk indexer:
      ```bash
      sudo /opt/splunkforwarder/bin/splunk add forward-server localhost:9997 \
        -auth admin:YOUR_SPLUNK_PASSWORD
      ```

- [ ] Tell the forwarder to monitor Wazuh's alerts file:
      ```bash
      sudo /opt/splunkforwarder/bin/splunk add monitor \
        /var/ossec/logs/alerts/alerts.json \
        -index wazuh-alerts \
        -sourcetype wazuh:alerts \
        -auth admin:YOUR_SPLUNK_PASSWORD
      ```

- [ ] Start the forwarder and enable on boot:
      ```bash
      sudo /opt/splunkforwarder/bin/splunk start
      sudo /opt/splunkforwarder/bin/splunk enable boot-start
      ```

**Verify Data Is Flowing**

- [ ] In Splunk Web → Search & Reporting, run:
      ```spl
      index="wazuh-alerts"
      ```
      If no results appear immediately, wait 2–3 minutes (forwarder has a polling delay)
      then try again.

- [ ] If still no results after 5 minutes, check the forwarder log:
      ```bash
      sudo tail -f /opt/splunkforwarder/var/log/splunk/splunkd.log
      ```
      Look for connection errors or "index does not exist" messages.
      Common cause: the `wazuh-alerts` index wasn't created before the forwarder
      started. Create it in Splunk Web, then restart the forwarder.

- [ ] **The fishbucket problem:** The Splunk forwarder tracks what it's already read
      in an internal database. On first start it may think it has already processed
      all existing content in `alerts.json`. If searches return zero results but
      the log shows no errors, trigger a new Wazuh alert (e.g. SSH with wrong
      password to one of your agents) to generate fresh content in the file.

**Import Wazuh Dashboards for Splunk**

- [ ] Download the prebuilt Wazuh Splunk dashboards from:
      https://github.com/wazuh/wazuh-splunk (check the repository for current
      dashboard files compatible with your Wazuh version)

- [ ] In Splunk Web: Search & Reporting → Dashboards → Create New Dashboard
      → Dashboard Studio → paste dashboard JSON content → Save

**Useful SPL Searches for Wazuh Data**

```spl
# All Wazuh alerts
index="wazuh-alerts"

# Extract and display key fields from Wazuh JSON
index="wazuh-alerts"
| spath
| table _time, rule.description, rule.level, agent.name, data.srcip

# High severity alerts only (level 10+)
index="wazuh-alerts"
| spath rule.level
| where 'rule.level' >= 10

# Alerts sorted by most recent
index="wazuh-alerts"
| spath
| sort - _time
| table _time, agent.name, rule.description, rule.level

# MITRE ATT&CK technique view
index="wazuh-alerts"
| spath rule.mitre.id
| stats count by 'rule.mitre.id', rule.description
| sort - count
```

### Sources
- Wazuh → Splunk integration official docs: https://documentation.wazuh.com/current/integrations-guide/splunk/index.html
- Wazuh Splunk integration blog walkthrough: https://wazuh.com/blog/detection-with-splunk-integration/
- Real-world lab forwarder setup + gotchas: https://medium.com/@raynardwaits/integrating-cloud-hosted-wazuh-with-on-premise-splunk-part-2-universal-forwarder-setup-7641d6f3c036
- Wazuh/Splunk dashboard repo: https://github.com/wazuh/wazuh-splunk

---

## Phase 4 — OpenEDR (Self-Hosted, Docker)

**Goal:** OpenEDR backend running via Docker on the OpenEDR VM; Windows agent
installed on the Desktop; endpoint telemetry (process trees, file events, network
connections) visible in the Kibana frontend.
**Estimated time:** 5–8 hours (budget the most slack here — most moving parts)
**VM:** `openedr.home.arpa` (10.40.40.13, on Minimox)

### What OpenEDR Is and Why It's Different from Wazuh

Wazuh is an alert-driven SIEM/XDR — it ingests logs, applies rules, and fires
alerts. OpenEDR is event-driven telemetry at a lower level: every process creation,
file open, registry write, and network connection on a Windows endpoint is recorded
as a base event and sent to the backend. The value is in the process tree view and
raw behavioral data, not in curated alerts.

```
Two separate projects exist — know which one you're using:

1. ComodoSecurity/openedr (GitHub)
   - The original "official" Comodo fork
   - Backend: ELK stack (Elasticsearch + Logstash + Kibana) via Docker
   - Agent: Windows MSI (signed by Comodo Security Solutions)
   - Telemetry path: Agent → Filebeat → Logstash → Elasticsearch → Kibana
   - Status: Works, but ELK integration is manual (no prebuilt Filebeat module)
   - Repo: https://github.com/ComodoSecurity/openedr

2. jymcheong/OpenEDR (renamed "Free EDR" to avoid name confusion)
   - Community fork, more turnkey
   - Backend: docker-compose script, SFTP receiver + OrientDB + web frontend
   - Agent: Windows endpoint agent
   - Install: Single curl command + shell script
   - Status: More plug-and-play, less enterprise-feeling dashboard
   - Repo: https://github.com/jymcheong/OpenEDR
```

> **Recommendation:** Start with jymcheong/OpenEDR for lower friction and a faster
> first working result. Once you understand the telemetry model, try the ComodoSecurity
> path for the ELK experience (which transfers to other lab setups).

### Checklist — jymcheong/OpenEDR (Recommended First)

**Backend Setup (on OpenEDR VM)**

- [ ] SSH into the OpenEDR VM (`ssh user@openedr.home.arpa`)

- [ ] Install Docker and Docker Compose:
      ```bash
      sudo apt update
      sudo apt install -y ca-certificates curl
      sudo install -m 0755 -d /etc/apt/keyrings
      sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
        -o /etc/apt/keyrings/docker.asc
      sudo chmod a+r /etc/apt/keyrings/docker.asc
      # Add Docker repo
      echo "deb [arch=$(dpkg --print-architecture) \
        signed-by=/etc/apt/keyrings/docker.asc] \
        https://download.docker.com/linux/ubuntu \
        $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
        sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
      sudo apt update
      sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
      # Add your user to docker group so you don't need sudo
      sudo usermod -aG docker $USER
      # Log out and back in, then verify:
      docker --version
      docker compose version
      ```

- [ ] Run the OpenEDR install script:
      ```bash
      curl -L https://github.com/jymcheong/OpenEDR/tarball/master | tar xz \
        && mv jym* openEDR \
        && cd openEDR \
        && ./install.sh
      ```
      The script will prompt for two IP addresses:
      - **SFTP IP** (where agents send telemetry): enter `10.40.40.13`
      - **Frontend IP** (web interface): enter `10.40.40.13`
      Use the same IP for both in a non-production/lab environment.

- [ ] Verify containers are running:
      ```bash
      docker ps
      ```
      Should show OrientDB, the SFTP receiver, and the web frontend containers.
      Check logs if something fails:
      ```bash
      docker logs -f orientdb
      ```

- [ ] Access the web frontend at: `http://openedr.home.arpa:8080`
      Access OrientDB at: `http://openedr.home.arpa:2480`

**Windows Agent Setup (on Desktop)**

- [ ] Download the OpenEDR Windows agent MSI from the ComodoSecurity GitHub:
      https://github.com/ComodoSecurity/openedr/releases
      Look for the signed MSI installer — `OpenEdrAgent*.msi`

- [ ] Install with the backend IP configured:
      ```
      msiexec /i OpenEdrAgent.msi EDRSVRADDRESS=10.40.40.13
      ```
      Or run the MSI normally and configure the server address in the installer UI.

- [ ] Verify telemetry is appearing in the OrientDB console and the web UI.
      If no data appears within 2–3 minutes of install, check:
      - Windows Firewall on the desktop is not blocking the agent's outbound connection
        (the agent reaches out TO the backend — same direction as Wazuh agents)
      - The SFTP port (22) on the backend VM is open and Docker is mapping it:
        ```bash
        sudo ss -tlnp | grep 22
        docker ps   # confirm SFTP container is up
        ```

### VLAN Consideration for OpenEDR Agent Traffic

The OpenEDR agent on the Desktop (VLAN 1 / Main LAN) connects to the OpenEDR
backend (VLAN 40 / Servers). Both VLANs are in the **Internal zone** in your
UniFi ZBF setup. Internal zone devices can reach each other freely by default —
no new firewall rules needed for this traffic path.

If you later install an agent on IoT-zone devices (currently only the AX21 WAP
is on VLAN 20 / IoT), that traffic would be blocked by the automatic IoT→Internal
block rule. You'd need an explicit allow rule for that specific source IP and
destination port before IoT agents could check in.

### Sources
- ComodoSecurity/openedr (original repo): https://github.com/ComodoSecurity/openedr
- ComodoSecurity Docker + ELK guide: https://github.com/ComodoSecurity/openedr/blob/main/getting-started/DockerInstallation.md
- jymcheong/OpenEDR (community fork — "Free EDR"): https://github.com/jymcheong/OpenEDR
- Full walkthrough (ComodoSecurity + ELK, Proxmox-based): https://medium.com/@panagiotisfisk/setting-up-openedr-with-elk-integration-feeffa1e891b
- Docker install documentation: https://docs.docker.com/engine/install/ubuntu/

---

## Phase 5 — Validation

**Goal:** Generate real (simulated) attack activity, confirm it surfaces in all
three platforms — Wazuh dashboard, Splunk, and OpenEDR — and document what each
tool caught vs. missed. This is the actual cybersecurity practice payoff.
**Estimated time:** 2–3 hours
**Hosts:** Desktop/Laptop (attack source) + test VM (target)

### Why a Test VM, Not Your Daily Driver

When you run active detections against a real machine (brute-force, scanning,
malware drop), you want to do it against a dedicated target — not your desktop.
Spin up a small Ubuntu or Kali VM on Bigmox or Minimox, enroll it as a Wazuh
agent, and use it as the victim machine. Your desktop is the "attacker" or the
"analyst workstation."

### Tests to Run

**Test 1 — EICAR Antivirus Test File (safe malware simulation)**
```
# Download the EICAR test string to a file on the test VM or desktop:
echo 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*' \
  > /tmp/eicar.txt
```
Expected: Wazuh FIM fires an alert about the new file in `/tmp`. If Windows Defender
is monitoring the desktop, it may delete the file immediately — that action itself
is also visible in Wazuh and OpenEDR.

**Test 2 — Failed SSH Brute-Force (classic detection)**
```bash
# From desktop, against the test VM (adjust IP):
for i in {1..10}; do
  ssh wronguser@10.40.40.x 2>/dev/null
done
```
Expected: Wazuh fires a brute-force alert (rule 5763 or similar). Should also
appear in Splunk as a Wazuh alert via the Phase 3 integration.

**Test 3 — Nmap Port Scan (network reconnaissance)**
```bash
# Install nmap on desktop or test VM:
sudo apt install nmap   # Linux
# Or use Nmap for Windows: https://nmap.org/download.html

# Scan another host on VLAN 40:
nmap -sV 10.40.40.x
```
Expected: Wazuh picks this up via network traffic analysis. OpenEDR on the target
should show the incoming connection attempts in the process/network view.

**Test 4 — Suspicious Process Execution (Windows)**
On the desktop, run something Wazuh and OpenEDR consider interesting:
```powershell
# Invoke-Expression running an encoded command (common malware technique):
$encoded = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes("whoami"))
powershell -encodedCommand $encoded
```
Expected: OpenEDR process tree shows the PowerShell invocation. Wazuh may fire
a Windows security event alert (EventCode 4688 if audit policy is enabled from
Phase 2). Splunk should show the process creation event.

### Validation Checklist

- [ ] Test 1 (EICAR): Alert visible in Wazuh FIM dashboard
- [ ] Test 1 (EICAR): Alert visible in Splunk `wazuh-alerts` index
- [ ] Test 2 (Brute-force): Wazuh fires brute-force rule
- [ ] Test 2 (Brute-force): Splunk shows the correlated Wazuh alert
- [ ] Test 3 (Nmap): Detection visible in Wazuh
- [ ] Test 4 (PowerShell): Process tree visible in OpenEDR
- [ ] Test 4 (PowerShell): Event 4688 visible in Splunk `windows-events` index
- [ ] Document what each tool caught vs missed in the Troubleshooting Log below

### Sources
- EICAR test file standard: https://www.eicar.org/download-anti-malware-testfile/
- Wazuh brute-force PoC: https://documentation.wazuh.com/current/proof-of-concept-guide/detect-brute-force-attack.html
- Wazuh FIM PoC: https://documentation.wazuh.com/current/proof-of-concept-guide/poc-file-integrity-monitoring.html
- Nmap official site: https://nmap.org/
- Atomic Red Team (more advanced test cases for later): https://github.com/redcanaryco/atomic-red-team

---

## Phase 6 — Cleanup and Tie-Down

**Goal:** No loose firewall ends, resource usage reviewed, everything is documented,
and future-work items are captured.
**Estimated time:** 1–2 hours

### Checklist

- [ ] **Re-verify all connection-state fixes from Phase 0 are still in place**
      UniFi ZBF → open "Block Servers to Management" → confirm Connection State = New
      Open "Block Servers to Main" → confirm still New (carried over from Issue 10 fix)

- [ ] **Check Bigmox and Minimox resource usage**
      In Proxmox UI: Node → Summary → check CPU and RAM usage across all running VMs
      If either host is consistently above 80% RAM, consider:
      - Moving Splunk off the Wazuh VM onto its own smaller VM on Minimox
      - Reducing log retention in Wazuh (default 90 days can be shortened)
      - Reducing OpenEDR telemetry verbosity in the agent config

- [ ] **Verify all DNS records are correct and consistent**
      From desktop, run all four:
      ```
      nslookup minimox.home.arpa   → should return 10.40.40.10
      nslookup bigmox.home.arpa    → should return 10.40.40.11
      nslookup wazuh.home.arpa     → should return 10.40.40.12
      nslookup openedr.home.arpa   → should return 10.40.40.13
      ```

- [ ] **Plan VLAN 50 (Cyber Lab) zone assignment**
      Per the network doc roadmap, VLAN 50 is the future pentest/attack-defense lab
      connected on UCG-Ultra LAN 4. When you build it:
      - Do NOT put it in the Internal zone — Internal-zone networks can reach each
        other freely. An isolated attack lab in the Internal zone can reach VLAN 1,
        40, and 99 by default.
      - Create a **new dedicated zone** for VLAN 50 (e.g. "CyberLab" zone)
      - Write explicit, narrow allow rules: only monitoring traffic from VLAN 50
        to the Wazuh manager (10.40.40.12) on port 1514/1515 should be permitted
      - Everything else from VLAN 50 → other zones: Block

- [ ] **Wazuh → update check**
      Wazuh recommends disabling the package repo after install to prevent accidental
      upgrades. Confirm the repo is disabled:
      ```bash
      cat /etc/apt/sources.list.d/wazuh.list
      # Lines should be commented out (#deb ...) not active
      ```

- [ ] **Note the Splunk trial expiry date** and what features drop when it converts:
      Settings → Licensing → check trial expiration
      Features lost after conversion: Alerting, Multi-user auth, Distributed search
      Action items: Export all saved searches and dashboard definitions before expiry
      as a backup.

---

## Phase 7 — Unified Dashboard & Correlation (Capstone)

**Goal:** Consolidate Wazuh, Windows-event, and OpenEDR data into a single Splunk
view. This is the primary demo screen for the YouTube series / recruiter
walkthrough — one dashboard showing correlated detections across all three tools,
instead of tabbing between three separate UIs. This is the final step of the build.
**Estimated time:** 4–6 hours (this is real scripting work, not a forwarder install —
budget accordingly, more likely to run long than Phases 1–3 did)
**VM:** Script runs on the OpenEDR VM (`openedr.home.arpa`, keeps OrientDB queries
local); dashboard is built in Splunk on the Wazuh VM (`wazuh.home.arpa:8000`)

### Why This One's Harder Than Phase 3

Phase 3's Wazuh→Splunk integration works because Wazuh writes a flat
`alerts.json` log file — the Splunk Universal Forwarder just tails it. OpenEDR
(jymcheong fork) has no equivalent flat file: its data sits in OrientDB behind
an SFTP receiver. There's nothing to tail, so this needs a small script that
queries OrientDB directly and pushes events into Splunk over its HTTP Event
Collector (HEC) — the same port (`8088`) already reserved in the Phase 1 port
table but unused until now.

### Checklist

**Enable Splunk HEC**

- [ ] Splunk Web → Settings → Data Inputs → HTTP Event Collector → New Token
      Name it `openedr-hec`, set/create the destination index: `openedr-events`
- [ ] Settings → Data Inputs → HTTP Event Collector → Global Settings → set to **Enabled**
- [ ] Note the generated token — you'll need it in the polling script below
- [ ] Confirm the `openedr-events` index exists (Settings → Indexes → New Index,
      if it wasn't auto-created by the token setup)
- [ ] Verify HEC is reachable:
      ```bash
      curl -k https://wazuh.home.arpa:8088/services/collector/health \
        -H "Authorization: Splunk <token>"
      ```

**Build the OrientDB → Splunk Polling Script**

- [ ] On the OpenEDR VM, identify OrientDB's REST query surface for whatever
      classes/tables hold event data (varies by install — treat this as a
      discovery step, not a known quantity). OrientDB's REST API is typically
      reachable at `http://openedr.home.arpa:2480/query/<db>/sql/<query>`.
- [ ] Write a Python script (same language choice as the Minecraft server
      automation script — JSON parsing gets messy fast in bash) that:
      - Queries OrientDB for events newer than the last poll, tracking a
        "last seen" timestamp/ID in a local state file (same pattern already
        used for tracking "current file ID" in the modpack updater script)
      - Formats each event to match Splunk HEC's expected payload:
        ```json
        {"event": {...}, "sourcetype": "openedr:events", "index": "openedr-events"}
        ```
      - POSTs batches to `https://wazuh.home.arpa:8088/services/collector/event`
        with the HEC token in the `Authorization: Splunk <token>` header
- [ ] Add the shebang `#!/usr/bin/env python3` and `chmod +x` it
- [ ] Schedule it on a cron/systemd timer (every 60 seconds is a reasonable
      start) — verify the cron expression on crontab.guru before deploying

**Verify and Build the Dashboard**

- [ ] Confirm events are landing in Splunk:
      ```spl
      index="openedr-events"
      ```
- [ ] In Splunk (Dashboard Studio), build one unified dashboard with:
      - Panel 1: Wazuh alerts timeline — `index="wazuh-alerts"`
      - Panel 2: Windows security events — `index="windows-events"`
      - Panel 3: OpenEDR telemetry — `index="openedr-events"`
      - Panel 4 (the payoff panel): a correlated view joining across all three
        indexes on a shared field (hostname or timestamp) for one incident
- [ ] Re-run the Phase 5 PowerShell encoded-command test (it's the one that
      already touches all three tools) and confirm it shows up on the unified
      dashboard without opening Wazuh's or OpenEDR's UI separately
- [ ] Clean up the dashboard for recording — hide raw SPL panels if desired,
      keep only the visual layer. This is your capstone footage.

### Sources
- Splunk HTTP Event Collector: https://docs.splunk.com/Documentation/Splunk/latest/Data/UsetheHTTPEventCollector
- HEC health/troubleshooting: https://docs.splunk.com/Documentation/Splunk/latest/Data/TroubleshootHTTPEventCollector
- OrientDB REST API reference: https://orientdb.org/docs/3.2.x/misc/OrientDB-REST.html
- Splunk Dashboard Studio: https://docs.splunk.com/Documentation/Splunk/latest/DashStudio/dsOverview
- crontab.guru (cron expression verification): https://crontab.guru/

---

## New Roadmap Items (Additions to Network Doc Roadmap)

- [ ] VLAN 50 (Cyber Lab) — create a dedicated zone, not Internal (see Phase 6)
- [ ] Install Wazuh agent on any future Linux VMs created for lab work
- [ ] Consider adding a Wazuh agent on the Raspberry Pi's `tailscale0` interface
      for visibility into traffic entering via Tailscale
- [ ] After Splunk trial expires: evaluate keeping Splunk Free (dashboards only),
      switching to Grafana + Loki (fully free alternative), or finding a student
      license via Splunk's academic program
- [ ] Explore Atomic Red Team for structured attack simulation on VLAN 50 once built
- [ ] Phase 7 (Unified Dashboard) is the final step before the build is considered
      "done" for the YouTube series — schedule it last, after Phases 1–6 are stable

---

## Command Reference (This Build)

```bash
# ── WAZUH ──────────────────────────────────────────────────────────
# Check Wazuh service status
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard

# Restart all Wazuh services
sudo systemctl restart wazuh-manager wazuh-indexer wazuh-dashboard

# Check agent status (on the manager)
sudo /var/ossec/bin/agent_control -l

# Wazuh alert log (raw JSON)
sudo tail -f /var/ossec/logs/alerts/alerts.json

# Check Wazuh agent status (on an endpoint)
sudo systemctl status wazuh-agent
sudo journalctl -u wazuh-agent -n 30 --no-pager

# Disable Wazuh repo after install (Ubuntu/Debian)
sudo sed -i "s/^deb/#deb/" /etc/apt/sources.list.d/wazuh.list


# ── SPLUNK ─────────────────────────────────────────────────────────
# Start/stop/restart Splunk
sudo /opt/splunk/bin/splunk start
sudo /opt/splunk/bin/splunk stop
sudo /opt/splunk/bin/splunk restart

# Check Splunk status
sudo /opt/splunk/bin/splunk status

# Splunk Universal Forwarder (on Wazuh VM)
sudo /opt/splunkforwarder/bin/splunk start
sudo /opt/splunkforwarder/bin/splunk status
sudo tail -f /opt/splunkforwarder/var/log/splunk/splunkd.log

# Add a monitor input
sudo /opt/splunkforwarder/bin/splunk add monitor \
  /var/ossec/logs/alerts/alerts.json \
  -index wazuh-alerts \
  -sourcetype wazuh:alerts \
  -auth admin:PASSWORD

# List current monitor inputs
sudo /opt/splunkforwarder/bin/splunk list monitor


# ── DOCKER (OpenEDR VM) ────────────────────────────────────────────
# Check running containers
docker ps

# Check container logs
docker logs -f <container_name>

# Restart all containers in compose file
cd ~/openEDR && docker compose restart

# Stop all containers
cd ~/openEDR && docker compose down

# Start containers
cd ~/openEDR && docker compose up -d


# ── NETWORKING / GENERAL ───────────────────────────────────────────
# Confirm IP on a VM
ip a show ens18

# DNS resolution test (always run from the machine that needs to look up, not the target)
nslookup wazuh.home.arpa
nslookup openedr.home.arpa

# Check open ports on a host
sudo ss -tlnp

# Release and renew DHCP (Linux)
sudo dhclient -r ens18 && sudo dhclient ens18

# Check which ports a service is listening on
sudo ss -tlnp | grep 9997    # Splunk receiving
sudo ss -tlnp | grep 1514    # Wazuh agent port
sudo ss -tlnp | grep 443     # Wazuh dashboard
sudo ss -tlnp | grep 8000    # Splunk web
sudo ss -tlnp | grep 8080    # OpenEDR web
```

---

## Troubleshooting Log

*Append to this as issues are encountered. Same format as the network doc.*

### Issue 1 — Kernel panics during Ubuntu install on Bigmox (Wazuh VM)
**Symptom:** Ubuntu installer repeatedly threw kernel panics while installing on the Wazuh VM (Bigmox).
**Cause:** Bad/corrupted installer image.
**Fix:** Downloaded a fresh copy of the Ubuntu installer ISO and reran the install — completed cleanly with no further panics.

### Issue 2 — Bigmox appeared to lose networking entirely ("dead") after a reboot
**Symptom:** After a reboot, Bigmox was unreachable — looked like it wasn't getting a valid IP, initially suspected as a dead NIC or failed hardware. Drove a whole side-investigation into moving the entire SOC stack onto Minimox alone.
**Cause:** A networking config file had reverted to manual/static instead of DHCP. Most likely a package update overwrote the config at some point, and the change only actually took effect once the box was rebooted — so the reboot looked like the trigger, but the real cause had been sitting there dormant since whatever update wrote it.
**Fix:** Corrected the networking config back to DHCP. Bigmox came back immediately — not a hardware failure, not a dead NIC. No data or hardware was actually lost.
**Note:** Don't write this class of issue off as "weird" or hardware-related again before checking the actual interface config first — this one cost significant time chasing a full host-replacement contingency plan (see the Minimox-only resource analysis earlier in this doc's history) that turned out to be unnecessary.

### Issue 3 — Wazuh dashboard's centralized-config editor rejects valid agent.conf XML
**Symptom:** Saving a `<syscheck>` block via Management → Groups → agent.conf threw `AxiosError: mismatched tag` errors at various line/column positions, even though the XML looked structurally valid.
**Cause:** A trailing backslash immediately before a closing tag (e.g., `C:\Windows\System32\</directories>`) is misinterpreted as an escape character by Wazuh's actual XML parser — not just a dashboard UI quirk. Confirmed via Wazuh GitHub issues #34132 and #21439.
**Fix:** Remove trailing backslashes from directory paths before the closing tag (e.g., `C:\Windows\System32` instead of `C:\Windows\System32\`).

### Issue 4 — FIM `file_limit` setting rejected with "Invalid element in the configuration"
**Symptom:** Saving a `<file_limit>` block inside `<syscheck>` produced: `Wazuh syntax error: Invalid element in the configuration: 'file_limit'... Syscheck remote configuration... is corrupted.`
**Cause:** Not fully root-caused, but element ordering appears to matter — the version that saved successfully places `<file_limit>` before the `<directories>` entries.
**Fix:** Reorder `<file_limit>` to appear before `<directories>` within `<syscheck>`. Confirmed the resulting config saved successfully and was present correctly on the manager's filesystem afterward.

### Issue 5 — FIM silently stopped tracking new files/folders under C:\Users
**Symptom:** `realtime="yes"` FIM was configured on `C:\Users`, but creating new files/folders there produced no alerts.
**Cause:** The Windows agent had hit the default `file_limit` of 100,000 monitored files (99,989/100,000 per a rootcheck warning) — files beyond the cap are silently dropped from monitoring.
**Fix:** Raised `file_limit` to 200,000 via the Windows group's centralized `agent.conf` (see Issue 4 for the save-order fix needed first).

### Issue 6 — Group config file "not found" on the manager despite the dashboard showing content
**Symptom:** `sudo cat /var/ossec/etc/shared/windows/agent.conf` returned "No such file or directory," even though the dashboard's Groups editor displayed saved content for the "windows" group.
**Cause:** Group folder names are case-sensitive on the manager's Linux filesystem. The group had actually been created as `Windows` (capital W), not `windows`.
**Fix:** Use `ls -la /var/ossec/etc/shared/` to confirm exact folder casing before assuming a path — correct path was `/var/ossec/etc/shared/Windows/agent.conf`.

### Issue 7 — Correct config on disk, but changes still not taking effect
**Symptom:** The Windows agent had pulled the correct, updated `agent.conf` (verified locally at `C:\Program Files (x86)\ossec-agent\shared\agent.conf`), but FIM behavior hadn't changed.
**Cause:** Centralized config files sync to the agent automatically (via keepalive checksum comparison), but the running `wazuh-agent` service only loads the file into memory on startup — `file_limit` and other syscheck settings are not hot-reloadable. The agent's `ossec.log` confirmed no restart had occurred since before the config was finalized.
**Fix:** Restart the Wazuh agent service (`Restart-Service WazuhSvc`) any time a centralized syscheck setting changes.

### Issue 8 — No FIM alerts immediately after restart despite correct, loaded config
**Symptom:** A test file created within a minute of restarting the Wazuh Windows agent produced no syscheck "added" alert.
**Cause:** Real-time FIM monitoring only activates after the initial baseline scan of all configured directories completes. With `C:\Users` containing roughly 100,000 files, that baseline scan takes meaningful time after every restart — files created during the scan window are folded into the new baseline silently instead of generating an alert.
**Fix:** Watch `ossec.log` for the scan-end message before testing; create a fresh test file only after the baseline scan has finished (a file created during the scan window won't retroactively alert).

### Issue 9 — (Ongoing/Unresolved) Baseline scan spamming warnings and possibly stalling on deep/sandboxed paths
**Symptom:** During the post-restart baseline scan, `ossec.log` filled with repeated `WARNING: (6720): The path '...' is too long` entries for deeply nested paths (VS Code extensions, npm `node_modules` trees, and Windows AppContainer folders under `AppData\Local\Packages`). Scan progress appeared to stop producing new log output for 10+ minutes while processing paths under `AppData\Local\Packages`.
**Cause:** Confirmed: Wazuh's Windows agent enforces a hardcoded ~260-character path limit for FIM (a legacy holdover from the old Windows `MAX_PATH` restriction that Windows itself has since removed) — acknowledged as an open, unresolved limitation by the Wazuh dev team (GitHub issues #26801, #11583). Suspected but unconfirmed: folders under `AppData\Local\Packages` are Windows AppContainer-sandboxed (UWP/MSIX app data) with restrictive ACLs and possible reparse-point redirection, which can cause third-party scanners to hang or slow dramatically.
**Fix:** Not yet resolved. Next diagnostic step is checking whether the `wazuh-agent` process is still consuming CPU (i.e., actually working, just quiet in the log) vs. genuinely stalled. If confirmed stalled, the planned mitigation is excluding `AppData\Local\Packages` (and similar low-value app-sandbox cache folders) from the `C:\Users` FIM scope rather than fighting the AppContainer restrictions.

---

## Sources & References

**Wazuh**
- Official quickstart / hardware requirements: https://documentation.wazuh.com/current/quickstart.html
- Wazuh server installation: https://documentation.wazuh.com/current/installation-guide/wazuh-server/index.html
- Wazuh agent installation: https://documentation.wazuh.com/current/installation-guide/wazuh-agent/index.html
- File Integrity Monitoring: https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/index.html
- Security Configuration Assessment: https://documentation.wazuh.com/current/user-manual/capabilities/sec-config-assessment/index.html
- Vulnerability Detection: https://documentation.wazuh.com/current/user-manual/capabilities/vulnerability-detection/index.html
- Wazuh → Splunk integration docs: https://documentation.wazuh.com/current/integrations-guide/splunk/index.html
- Wazuh → Splunk integration blog: https://wazuh.com/blog/detection-with-splunk-integration/
- Wazuh Proof of Concept guides: https://documentation.wazuh.com/current/proof-of-concept-guide/index.html

**Splunk**
- Splunk Enterprise download: https://www.splunk.com/en_us/download/splunk-enterprise.html
- Universal Forwarder download: https://www.splunk.com/en_us/download/universal-forwarder.html
- License types (Free vs Enterprise vs Trial): https://docs.splunk.com/Documentation/Splunk/latest/Admin/TypesofSplunklicenses
- SPL search reference: https://docs.splunk.com/Documentation/Splunk/latest/SearchReference/WhatsInThisManual
- Enable receiver port: https://docs.splunk.com/Documentation/Splunk/latest/Forwarding/Enableareceiver
- Real-world Wazuh+Splunk lab (forwarder setup + gotchas): https://medium.com/@raynardwaits/integrating-cloud-hosted-wazuh-with-on-premise-splunk-part-2-universal-forwarder-setup-7641d6f3c036

**OpenEDR**
- ComodoSecurity/openedr (original repo): https://github.com/ComodoSecurity/openedr
- ComodoSecurity Docker install guide: https://github.com/ComodoSecurity/openedr/blob/main/getting-started/DockerInstallation.md
- ComodoSecurity ELK setup guide: https://github.com/ComodoSecurity/openedr/blob/main/getting-started/SettingELK.md
- jymcheong/OpenEDR ("Free EDR" fork, more turnkey): https://github.com/jymcheong/OpenEDR
- Full Proxmox-based OpenEDR + ELK walkthrough: https://medium.com/@panagiotisfisk/setting-up-openedr-with-elk-integration-feeffa1e891b
- ComodoSecurity + ELK walkthrough (Medium): https://medium.com/@raghavvram/openedr-comodosecurity-958b1b9dcc17

**Docker**
- Docker install on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Docker Compose reference: https://docs.docker.com/compose/

**Windows Security / Audit Policy**
- Windows Security Event ID Encyclopedia: https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/
- Advanced Audit Policy (Microsoft): https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/advanced-security-audit-policy-settings
- Enable process creation logging: https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/component-updates/command-line-process-auditing

**Validation / Attack Simulation**
- EICAR test file: https://www.eicar.org/download-anti-malware-testfile/
- Nmap: https://nmap.org/
- Atomic Red Team (future use): https://github.com/redcanaryco/atomic-red-team

---

*Document started: July 2026 | Last updated: July 5, 2026*
*Cross-references: Homelab-Network-Documentation.md (June 2026)*
