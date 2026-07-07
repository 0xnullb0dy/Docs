# Homelab Server & VM Specifications

> [!info] Purpose
> A quick-reference spec sheet for the two Proxmox hosts (Bigmox, Minimox) and every VM
> documented so far across `Homelab-Network-Documentation.md`,
> `Homelab-SOC-Build-Plan.md`, and `Homelab-IAM-Build-Plan.md`. Host hardware specs and
> the utilization snapshot below were pulled directly from Proxmox. Update this file as
> new VMs are built, planned VMs go live, or specs change.

---

## 1. Physical Hosts

| Field | Bigmox | Minimox |
|---|---|---|
| Role | Proxmox VE host | Proxmox VE host |
| Network connection | UCG-Ultra **LAN 2** (direct access port, VLAN 40) | Netgear managed switch, **Port 2** (VLAN 40, access) |
| VLAN / Subnet | 40 — Servers (10.40.40.0/24) | 40 — Servers (10.40.40.0/24) |
| IP address | 10.40.40.11 | 10.40.40.10 |
| Hostname (DNS) | `bigmox.home.arpa` | `minimox.home.arpa` |
| Web UI | `https://bigmox.home.arpa:8006` | `https://minimox.home.arpa:8006` |
| DHCP reservation | Yes (MAC-based) | Yes (MAC-based) |
| CPU(s) | 12 x AMD Ryzen 5 2600 Six-Core Processor (1 Socket) | 12 x AMD Ryzen 5 6600H with Radeon Graphics (1 Socket) |
| Total RAM | 31.29 GiB | 15.34 GiB |
| Total `/` disk | 36.81 GiB | 93.93 GiB |
| Swap | 8.00 GiB | 8.00 GiB |
| Kernel | Linux 7.0.2-6-pve (2026-05-20T08:55Z) | Linux 6.14.8-2-pve (2025-07-22T10:04Z) |
| Boot Mode | EFI | EFI |
| Proxmox Manager Version | pve-manager/9.2.2/b9984c6d90a4bd80 | pve-manager/9.0.3/025864202ebb6109 |
| Storage backend note | ZFS on a spare 1 TB drive (adopted for VM storage instead of LVM-Thin, after LVM-Thin was suspected in earlier corruption issues) | Not specified in source docs |
| Hosts (VMs) | Splunk VM *(repurposed from Wazuh)*, Ansible+Semaphore VM, Splunk SOAR VM *(planned)*, Keycloak VM *(planned)* | OpenEDR VM, OpenVAS VM, FreeIPA VM *(planned)*, IAM demo-apps VM *(planned)* |

### Live Utilization Snapshot — July 5, 2026

> **Stale as of the July 6 stack pivot.** This snapshot was pulled while Bigmox was
> still running the Wazuh VM (indexer + dashboard + Splunk all stacked together).
> The Wazuh VM has since been repurposed into the Splunk-only VM, and OpenVAS +
> Ansible/Semaphore VMs were added to the picture. Re-pull this snapshot before
> making any new sizing decisions — the numbers below no longer reflect what's
> actually running.

Point-in-time numbers pulled from Proxmox `Datacenter → [Host] → Summary`. These will
drift immediately — treat as a baseline, not a current reading, and re-pull before
making any sizing decision for new VMs.

| Metric | Bigmox | Minimox |
|---|---|---|
| CPU usage | 0.16% of 12 CPUs | 0.16% of 12 CPUs |
| IO delay | 0.00% | 0.00% |
| Load average (1/5/15 min) | 0.04, 0.12, 0.15 | 0.02, 0.05, 0.07 |
| RAM usage | 64.10% (20.06 GiB of 31.29 GiB) | 20.50% (3.14 GiB of 15.34 GiB) |
| KSM sharing | 0 B | 0 B |
| `/` disk usage | 30.47% (11.22 GiB of 36.81 GiB) | 29.58% (27.79 GiB of 93.93 GiB) |
| Swap usage | 0.00% (0 B of 8.00 GiB) | 0.00% (256.00 KiB of 8.00 GiB) |

> [!info] Decision — Keycloak stays on Bigmox
> Ran the numbers both ways before finalizing this:
> - **Bigmox** is at 64% RAM usage (20.06 of 31.29 GiB) with only the Wazuh VM running
>   — but that's ~11.23 GiB free in absolute terms. Adding Keycloak's planned 4 GB
>   still leaves ~7 GiB of headroom, and Bigmox's total 31.29 GiB pool is large enough
>   to absorb it comfortably.
> - **Minimox** looks roomier by percentage (only 20.5% used, ~12.2 GiB free), but it's
>   already committed to both the OpenEDR VM (8 GB configured) and the two planned IAM
>   VMs, FreeIPA + demo-apps (4 GB + 4 GB configured = 8 GB more). That's 16 GB of
>   configured RAM stacked on a host with only 15.34 GiB total physical RAM — already
>   tight before Keycloak is even considered. Adding Keycloak's 4 GB on top would push
>   Minimox into genuine overcommitment territory.
>
> **Conclusion: Keycloak (`iam.home.arpa`) stays on Bigmox.** FreeIPA (`idm.home.arpa`)
> and the demo-apps VM (`iam-apps.home.arpa`) stay on Minimox, per the original IAM
> plan — no host reassignment needed. Worth keeping an eye on Minimox's own 16 GB vs.
> 15.34 GiB configured-RAM math once `idm` and `iam-apps` actually go live; may need to
> trim one of their RAM allocations or rely on Proxmox memory ballooning.

### Mini/Big Swap Reminder

Originally planned as Bigmox = `.10` / Minimox = `.11`, but deliberately swapped during
setup to **Minimox = `.10` / Bigmox = `.11`**. Both DHCP reservations and DNS records
must match this swap — a mismatch here was the root cause of a documented
troubleshooting issue (Issue 11, network doc).

---

## 2. VM Inventory

All VMs currently documented, across both source files:

| VM / Hostname | Host | Purpose | OS | vCPU | RAM | Disk | Storage Backend | Network / VLAN | IP |
|---|---|---|---|---|---|---|---|---|---|
| `splunk.home.arpa` *(repurposed from `wazuh.home.arpa`)* | Bigmox | Splunk Enterprise — primary SIEM. Wazuh fully removed; this is the same VM, repurposed, not a rebuild | Ubuntu 24.04 LTS | 8 vCPU *(current — target is 4 vCPU, see note)* | 16 GB *(current — target is 8 GB)* | 50 GB *(current — target is 100 GB, resize planned)* | ZFS | `vmbr0`, VLAN 40 | 10.40.40.12 |
| `openedr.home.arpa` | Minimox | OpenEDR backend (Docker: OrientDB + SFTP receiver + web frontend, jymcheong fork) | Ubuntu 22.04 LTS | 4 vCPU | 8 GB | 100 GB | Not specified | `vmbr0`, VLAN 40 | 10.40.40.13 |
| `scanner.home.arpa` | Minimox | OpenVAS / Greenbone Community Edition — agentless vulnerability scanning across all VLANs | Ubuntu 22.04 LTS | 4 vCPU | 8 GB | 100 GB | Not specified | `vmbr0`, VLAN 40 | 10.40.40.14 |
| `ansible.home.arpa` | Bigmox | Ansible + Semaphore — config management / automation execution layer (future SOAR target) | Ubuntu 24.04 LTS | 2 vCPU | 2 GB | 20 GB | Not specified | `vmbr0`, VLAN 40 | 10.40.40.15 |
| `soar.home.arpa` *(planned)* | Bigmox | Splunk SOAR On-premises Community Edition — orchestration layer, not yet built | Ubuntu 22.04 LTS | 4 vCPU | 8 GB | 100 GB | Not specified | `vmbr0`, VLAN 40 | 10.40.40.16 |
| Test/attack-target VM *(optional, recommended)* | Bigmox or Minimox | Splunk UF + OpenEDR-enrolled victim machine for Phase 6 validation testing (brute-force, EICAR, Nmap scans) — kept separate from daily-driver machines | Ubuntu or Kali (unspecified) | Not specified | Not specified | Not specified | Not specified | VLAN 40 (implied) | Not assigned |
| `idm.home.arpa` *(planned)* | Minimox | FreeIPA identity backend (LDAP + Kerberos) for the IAM build | Rocky Linux 9 | 2 vCPU | 4 GB | 30 GB | Not specified | `vmbr0`, VLAN 40 | 10.40.40.17 |
| `iam.home.arpa` *(planned)* | Bigmox | Keycloak (OIDC/SAML/OAuth2 broker), Docker: Keycloak + Postgres | Ubuntu 24.04 LTS | 2 vCPU | 4 GB | 20 GB | Not specified | `vmbr0`, VLAN 40 | 10.40.40.18 |
| `iam-apps.home.arpa` *(planned)* | Minimox | Demo relying-party apps: Grafana (OIDC), BookStack (SAML2), Portainer CE (OAuth2), via Docker Compose | Ubuntu 24.04 LTS | 2 vCPU | 4 GB | 30 GB | Not specified | `vmbr0`, VLAN 40 | 10.40.40.19 |

> [!warning] IAM VM addresses renumbered (July 6, 2026)
> The IAM build's three planned VMs originally sat at `.14`/`.15`/`.16`. The new SOC
> stack (OpenVAS, Ansible+Semaphore, future Splunk SOAR) needed those exact
> addresses, and since the SOC build is the active project, the IAM VMs were moved
> to `.17`/`.18`/`.19` instead. Nothing else about the IAM plan changed — same hosts,
> same specs, just new IPs. Update `Homelab-IAM-Build-Plan.md` if it hardcodes the
> old addresses anywhere.

> [!note] Planned vs. built
> The three IAM VMs and `soar.home.arpa` above are **planned, not yet created**.
> IAM specs come from `Homelab-IAM-Build-Plan.md`; host assignment (Keycloak →
> Bigmox; FreeIPA and demo-apps → Minimox) is finalized — see the decision note
> above. Update each row from "planned" to built once the VM actually exists, and
> confirm real specs match what was planned.

> [!note] Wazuh dropped in favor of Splunk (July 6, 2026)
> Wazuh was evaluated and fully removed from the stack — Splunk was kept instead
> since it appears far more often on SOC analyst / security engineer job
> descriptions and SPL is a directly testable interview skill. The VM at
> `10.40.40.12` wasn't rebuilt, just repurposed: Wazuh packages removed, Splunk
> Enterprise installed in their place. See `Homelab-SOC-Build-Plan.md`'s
> Troubleshooting Log for the full (now-deprecated) Wazuh install saga, kept for
> historical reference.

> [!note] Why the SOC stack is split across two hosts
> Splunk's indexer and OpenEDR's ELK-style backend are both meaningful resource
> consumers — stacking everything on one Proxmox host risks starving both. Bigmox
> (direct LAN 2 connection) carries Splunk + Ansible/Semaphore + the future SOAR
> instance; Minimox carries OpenEDR + OpenVAS. One host going down doesn't take out
> the entire monitoring/scanning capability at once.

> [!note] Splunk VM resource sizing — right-sizing pending
> The VM's current 8 vCPU / 16 GB / 50 GB was sized for Wazuh's indexer + dashboard
> running alongside Splunk — a real bottleneck was hit during that install (the
> dashboard's one-time bundling step alone could exhaust 12 GB of RAM). Running
> Splunk alone is lighter; the new target spec is 4 vCPU / 8 GB, with disk actually
> increased to 100 GB for Splunk's own index storage needs. No urgency on the
> CPU/RAM trim — the extra headroom sitting idle doesn't hurt anything — but the
> disk resize is worth doing intentionally.

---

## 3. Non-Proxmox Compute (for context)

| Device | Role | Notes |
|---|---|---|
| Raspberry Pi | Tailscale subnet router; Splunk Universal Forwarder + Auditd (enrolled in SOC build, formerly a Wazuh agent) | VLAN 99 (Management), 10.99.99.3, `pi.home.arpa` — not a Proxmox VM, listed here since it participates in the SOC build |

---

## 4. Source Documents Scanned

- `Homelab-Network-Documentation.md` — hardware inventory, VLAN/IP scheme, DHCP reservations
- `Homelab-SOC-Build-Plan.md` — rewritten July 6, 2026 for the Wazuh → Splunk-primary
  stack pivot (Splunk, OpenEDR, OpenVAS, Ansible+Semaphore, future Splunk SOAR)
- `Homelab-IAM-Build-Plan.md` — planned FreeIPA/Keycloak/demo-apps VM specs (IPs
  renumbered July 6, 2026 due to conflict with the new SOC stack — see warning above)
- Proxmox `Datacenter → [Host] → Summary` — physical host CPU/RAM/disk/kernel/version and live utilization snapshot (Bigmox, Minimox), pulled directly, July 5, 2026 (now stale — see warning above)

This document was updated directly (not a fresh read-only scan) on July 6, 2026 to
reflect the SOC Build Plan rewrite. Re-verify against Proxmox directly before making
sizing decisions.

---

*Compiled July 2026 as a standing reference. Last updated: July 6, 2026 (Wazuh → Splunk pivot). Update whenever a new VM is created or specs change on Bigmox/Minimox.*
