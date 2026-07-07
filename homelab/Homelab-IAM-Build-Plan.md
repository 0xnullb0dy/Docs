# Homelab IAM Build Plan — Keycloak + FreeIPA

> **Purpose**
> This is the active planning runbook and progress tracker for building a self-hosted
> Identity and Access Management (IAM) layer on top of the existing UniFi/VLAN network
> and Proxmox hosts. Goal: a portfolio-grade project that proves hands-on IAM skill to
> recruiters — SSO across three protocols (OIDC, SAML, OAuth2), RBAC/ABAC, MFA,
> provisioning/deprovisioning lifecycle, and identity audit logging — using only free
> or open-source software (Keycloak, FreeIPA, and open-source demo apps).
>
> **Cross-reference:** This document assumes `Homelab-Network-Documentation.md` (VLANs,
> firewall zones, DNS convention) and reuses the same VLAN 40 / `home.arpa` pattern
> established for the SOC build in `Homelab-SOC-Build-Plan.md`. Check
> `Homelab-Server-VM-Specs.md` for current Bigmox/Minimox headroom before sizing new VMs.

---

## Why Keycloak + FreeIPA

Quick rationale, so the tool choice doesn't need re-litigating mid-build:

```
Tool       | License        | Role in this build              | Why
-----------|-----------------|----------------------------------|---------------------------------
Keycloak   | Apache 2.0      | Central IdP / SSO broker         | Broadest protocol coverage
           |                 | (OIDC, SAML2, OAuth2)             | (OIDC + SAML + LDAP + Kerberos
           |                 |                                    | federation) — most recognizable
           |                 |                                    | to recruiters/enterprise roles
FreeIPA    | GPL             | Identity backend (LDAP + Kerberos)| Enterprise Linux/UNIX identity
           |                 |                                    | management — federates into
           |                 |                                    | Keycloak via LDAP User Federation
```

> **Note on alternatives considered**
> Authentik is lighter-weight (~150–200 MB RAM vs. Keycloak's ~1.25 GB+ base) and has a
> friendlier visual flow editor — genuinely worth a look as a stretch/future phase (see
> Roadmap). But Keycloak was chosen as the core because its OIDC/SAML/LDAP/Kerberos
> protocol breadth and enterprise track record are what most IAM job postings actually
> reference. FreeIPA was chosen over standalone OpenLDAP because it bundles Kerberos +
> LDAP + a CA out of the box, which maps more directly to "enterprise identity backend"
> experience than raw OpenLDAP would.

---

## Quick Reference — Existing Infrastructure

Reused from the network doc and SOC doc — do not diverge from this without updating both.

### VLAN Map (relevant VLANs only)

```
VLAN  | Name       | Subnet          | Gateway      | Zone     | Key Devices
------|------------|-----------------|--------------|----------|-----------------------------
1     | Main LAN   | 10.10.10.0/24   | 10.10.10.1   | Internal | Desktop (PC), Laptop
40    | Servers    | 10.40.40.0/24   | 10.40.40.1   | Internal | Bigmox, Minimox, Wazuh, OpenEDR
99    | Management | 10.99.99.0/24   | 10.99.99.1   | Internal | Raspberry Pi, Netgear mgmt
```

### Existing DHCP Reservations (VLAN 40)

```
Device          | IP            | Host
----------------|---------------|--------------------
Minimox         | 10.40.40.10   | minimox.home.arpa
Bigmox          | 10.40.40.11   | bigmox.home.arpa
Wazuh VM        | 10.40.40.12   | wazuh.home.arpa
OpenEDR VM      | 10.40.40.13   | openedr.home.arpa
```

> **Firewall reminder (carried over from SOC build)**
> All of VLAN 40 sits in the **Internal** zone alongside VLAN 1 and 99, which means
> free communication by default between Main LAN, Servers, and Management. No new
> Zone-Based Firewall rules should be needed for this build *unless* a demo app needs
> to be reachable from IoT (VLAN 20) — in which case an explicit allow rule is required
> (IoT → Internal is blocked by default).

---

## Planned IP / DNS Allocations (New — This Build)

Add these to UCG-Ultra (Settings → Networks → VLAN 40 → DHCP Reservations and
Settings → DNS → Local Records) **during Phase 0**, before any VM is created.

```
Hostname                | IP            | VLAN | VM Host  | Purpose
-------------------------|---------------|------|----------|--------------------------------
idm.home.arpa            | 10.40.40.14   | 40   | Minimox  | FreeIPA (LDAP + Kerberos backend)
iam.home.arpa            | 10.40.40.15   | 40   | Bigmox   | Keycloak (OIDC/SAML/OAuth2 broker)
iam-apps.home.arpa       | 10.40.40.16   | 40   | Minimox  | Demo relying-party apps (Grafana,
                          |               |      |          | BookStack, Portainer CE)
```

> **Why a separate host for the demo apps?**
> Keeping relying-party apps (`iam-apps`) on a different VM from the IdP (`iam`) mirrors
> how real environments separate the identity provider from the applications that trust
> it — and it means breaking a demo app config can't accidentally take down the IdP
> itself mid-recruiter-demo.

> **Host assignment — finalized, based on real Proxmox headroom**
> Pulled live specs from Proxmox (`Homelab-Server-VM-Specs.md`) before deciding:
> - Bigmox: 31.29 GiB total RAM, 64% used (Wazuh VM only) → ~11.23 GiB free. Adding
>   Keycloak's planned 4 GB still leaves ~7 GiB of headroom.
> - Minimox: 15.34 GiB total RAM, only 20.5% used currently, but already carries the
>   OpenEDR VM (8 GB configured) plus the two other planned IAM VMs here (FreeIPA 4 GB
>   + demo-apps 4 GB) — that's 16 GB of configured RAM against a 15.34 GiB host before
>   Keycloak is even added. Putting Keycloak there too would overcommit it.
>
> **Decision: Keycloak (`iam.home.arpa`) stays on Bigmox.** FreeIPA (`idm.home.arpa`)
> and the demo-apps VM (`iam-apps.home.arpa`) stay on Minimox. This is locked in, not
> a placeholder — no further host-reassignment decision needed before building. Do
> keep an eye on Minimox's own RAM math (16 GB configured vs. 15.34 GiB physical) once
> `idm` and `iam-apps` actually go live — may need to trim one VM's RAM or lean on
> Proxmox memory ballooning.

---

## Planned VM Specs

```
VM                  | OS                | vCPU | RAM   | Disk   | Notes
---------------------|-------------------|------|-------|--------|----------------------------
idm.home.arpa        | Rocky Linux 9     | 2    | 4 GB  | 30 GB  | FreeIPA officially targets
(FreeIPA)            |                   |      |       |        | RHEL-family distros, not
                     |                   |      |       |        | Debian/Ubuntu — use Rocky
                     |                   |      |       |        | or AlmaLinux 9
iam.home.arpa        | Ubuntu 24.04 LTS  | 2    | 4 GB  | 20 GB  | Docker install: Keycloak +
(Keycloak)           |                   |      |       |        | Postgres containers
iam-apps.home.arpa   | Ubuntu 24.04 LTS  | 2    | 4 GB  | 30 GB  | Docker Compose: Grafana +
(demo apps)          |                   |      |       |        | BookStack + Portainer CE
```

These are starting estimates (Keycloak's own docs cite ~1.25 GB+ RAM for a bare base
instance, more in practice with Postgres alongside it). Revise upward the same way the
Wazuh VM was revised in the SOC build if real installs run tight.

---

## IAM Architecture (After Full Build)

```
                         VLAN 40 — Servers (10.40.40.0/24)
 ┌────────────────────────────────────────────────────────────────────────┐
 │                                                                        │
 │   ┌───────────────────┐        ┌──────────────────┐                   │
 │   │  idm.home.arpa    │◄──────►│  iam.home.arpa   │                   │
 │   │  10.40.40.14      │  LDAP  │  10.40.40.15     │                   │
 │   │  (Minimox VM)     │  federation │  (Bigmox VM)      │                   │
 │   │                   │        │                  │                   │
 │   │  FreeIPA:         │        │  Keycloak:        │                   │
 │   │  - LDAP directory │        │  - OIDC provider   │                   │
 │   │  - Kerberos realm │        │  - SAML2 IdP       │                   │
 │   │  - Users/Groups   │        │  - RBAC/UMA2 authz │                   │
 │   │                   │        │  - MFA (TOTP)      │                   │
 │   └───────────────────┘        └─────────┬────────┘                   │
 │                                           │ OIDC / SAML2 / OAuth2       │
 │                                           ▼                            │
 │                              ┌──────────────────────────┐              │
 │                              │  iam-apps.home.arpa      │              │
 │                              │  10.40.40.16 (Minimox VM) │              │
 │                              │                           │              │
 │                              │  Grafana   (OIDC)         │              │
 │                              │  BookStack (SAML2)        │              │
 │                              │  Portainer CE (OAuth2)    │              │
 │                              └──────────────────────────┘              │
 │                                                                        │
 └───────────────────────────────┬────────────────────────────────────────┘
                                  │ Keycloak event listener
                                  ▼
                     wazuh.home.arpa (10.40.40.12) — Splunk
                     (from the SOC build — identity events land
                      alongside Wazuh/EDR alerts for one unified
                      security picture, see Phase 7)
```

---

## Phase 0 — Network Prep

**Goal:** Reservations and DNS exist before any VM is created.
**Estimated time:** 1 hour

### Checklist

- [ ] Add DHCP reservations for `idm` (10.40.40.14), `iam` (10.40.40.15), and
      `iam-apps` (10.40.40.16) on VLAN 40, following the same MAC-after-first-boot
      pattern used for the Wazuh/OpenEDR VMs
- [ ] Add Local DNS records for all three hostnames
- [ ] Confirm resolution from the desktop (not from the VM itself):
      ```
      nslookup idm.home.arpa
      nslookup iam.home.arpa
      nslookup iam-apps.home.arpa
      ```
- [ ] Host assignment already finalized (Keycloak → Bigmox; FreeIPA + demo-apps →
      Minimox, per `Homelab-Server-VM-Specs.md`) — re-check actual free RAM on both
      hosts right before creating each VM in case anything else has changed since

---

## Phase 1 — FreeIPA Identity Backend

**Goal:** FreeIPA running with a small realistic org structure (users, groups, OUs) that
Keycloak will later federate against.
**Estimated time:** 3–4 hours
**VM:** `idm.home.arpa` (10.40.40.14, on Minimox)

### Checklist

- [ ] Install Rocky Linux 9 (minimal) on the VM
- [ ] Set a static hostname matching the DNS record: `hostnamectl set-hostname idm.home.arpa`
- [ ] Install and configure the FreeIPA server:
      ```bash
      sudo dnf install -y freeipa-server
      sudo ipa-server-install
      ```
      Follow prompts for realm name (e.g. `HOMELAB.ARPA`) and domain (`home.arpa`).
- [ ] Confirm services are up:
      ```bash
      sudo ipactl status
      ```
- [ ] Log into the FreeIPA web UI (`https://idm.home.arpa`) and create a sample org
      structure to federate later:
      - Groups: `it-admins`, `engineers`, `recruiters-demo`
      - 3–5 sample users spread across those groups
- [ ] Confirm Kerberos ticketing works: `kinit <user>` then `klist`
- [ ] Confirm LDAP search works from the CLI:
      ```bash
      ldapsearch -x -H ldap://idm.home.arpa -b "dc=home,dc=arpa" -D "cn=Directory Manager" -W
      ```

### Sources
- FreeIPA documentation: https://freeipa.readthedocs.io/
- FreeIPA server install guide: https://freeipa.org/page/Documentation
- Red Hat FreeIPA/Keycloak federation reference: https://access.redhat.com/solutions/3010401

---

## Phase 2 — Keycloak Core Install & LDAP Federation

**Goal:** Keycloak running with its own realm, federated to FreeIPA so FreeIPA users can
log into Keycloak without being re-created.
**Estimated time:** 3–5 hours
**VM:** `iam.home.arpa` (10.40.40.15, on Bigmox)

### Checklist

**Installation**

- [ ] Install Docker + Docker Compose on the VM
- [ ] Deploy Keycloak + Postgres via `docker-compose.yml` (Postgres as the persistent
      backing store rather than Keycloak's dev-mode embedded database)
- [ ] Access the admin console at `https://iam.home.arpa:8443` (or your chosen port),
      accept the self-signed cert, and set the initial admin password
- [ ] Create a dedicated realm for this project (do not build in the `master` realm)

**LDAP User Federation (connect to FreeIPA)**

- [ ] Realm → User Federation → Add provider → `ldap`
- [ ] Connection URL: `ldap://idm.home.arpa:389` (or `ldaps://` on 636 once FreeIPA's
      CA cert is trusted — prefer this for anything beyond a quick lab test)
- [ ] Bind DN / credentials: FreeIPA's Directory Manager or a dedicated bind service
      account (create one in FreeIPA rather than reusing Directory Manager long-term)
- [ ] Users DN: `cn=users,cn=accounts,dc=home,dc=arpa` (FreeIPA's default user tree)
- [ ] Run "Synchronize all users" and confirm the FreeIPA users/groups created in
      Phase 1 appear under Keycloak → Users
- [ ] Test login as a synced FreeIPA user through Keycloak's account console

### Sources
- Keycloak documentation home: https://www.keycloak.org/documentation
- Keycloak LDAP/User Storage Federation: https://www.keycloak.org/docs/latest/server_admin/#_ldap
- Keycloak Docker guide: https://www.keycloak.org/getting-started/getting-started-docker

---

## Phase 3 — SSO Protocol Coverage (Three Apps, Three Protocols)

**Goal:** One real login flow each for OIDC, SAML2, and OAuth2 — deliberately using a
different protocol per app so the finished project demonstrates breadth, not just one
integration repeated three times.
**Estimated time:** 4–6 hours
**VM:** `iam-apps.home.arpa` (10.40.40.16, on Minimox) — all three apps via Docker Compose

### 3a. Grafana — OIDC

- [ ] Deploy Grafana (OSS) via Docker
- [ ] In Keycloak: create a Client → Client type `OpenID Connect` → confidential access
- [ ] Configure Grafana's `grafana.ini` (or env vars) generic OAuth section pointing at
      Keycloak's OIDC discovery endpoint:
      `https://iam.home.arpa/realms/<realm>/.well-known/openid-configuration`
- [ ] Log into Grafana via "Sign in with Keycloak" and confirm the FreeIPA-sourced user
      identity carries through

### 3b. BookStack — SAML2

- [ ] Deploy BookStack via Docker
- [ ] In Keycloak: create a Client → Client type `SAML` → set BookStack's ACS URL and
      Entity ID
- [ ] Export Keycloak's realm SAML metadata/IdP certificate, import into BookStack's
      `.env` SAML2 config block
- [ ] Confirm SP-initiated login from BookStack redirects to Keycloak and back
      successfully

### 3c. Portainer CE — OAuth2

- [ ] Deploy Portainer Community Edition via Docker
- [ ] In Keycloak: create a Client for OAuth2 (public or confidential depending on
      Portainer's flow requirements)
- [ ] Settings → Authentication → OAuth in Portainer, point at Keycloak's
      authorization/token/userinfo endpoints
- [ ] Confirm login via Keycloak and that Portainer correctly reflects the group
      membership pulled from FreeIPA (ties into Phase 5 RBAC mapping)

### Sources
- Grafana generic OAuth (OIDC) setup: https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-authentication/generic-oauth/
- BookStack SAML2 auth docs: https://www.bookstackapp.com/docs/admin/saml2-auth/
- Portainer OAuth settings docs: https://docs.portainer.io/admin/settings/authentication/oauth
- OpenID Connect Core spec: https://openid.net/specs/openid-connect-core-1_0.html
- SAML 2.0 Core spec: https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf

---

## Phase 4 — MFA & Authentication Flows

**Goal:** TOTP-based MFA enforced for at least one user group, plus a conditional/
step-up flow so the admin console itself requires stronger auth than a demo app login.
**Estimated time:** 2–3 hours

### Checklist

- [ ] Realm → Authentication → set OTP Policy (algorithm, digits, look-ahead window)
- [ ] Require TOTP for the `it-admins` group specifically (not realm-wide) via a
      Conditional OTP flow — demonstrates group-scoped MFA, not just an on/off switch
- [ ] Enroll a test user with an authenticator app (Google Authenticator/Authy/Bitwarden
      all support standard TOTP) and confirm second-factor prompt on next login
- [ ] Build a step-up authentication flow: browsing to the Keycloak admin console
      itself demands OTP even if the initial app login didn't
- [ ] Document the difference between "required" and "conditional" auth flow
      executions — this distinction is a common interview question in IAM roles

### Sources
- Keycloak authentication flows: https://www.keycloak.org/docs/latest/server_admin/#_authentication-flows
- Keycloak OTP policies: https://www.keycloak.org/docs/latest/server_admin/#otp-policies

---

## Phase 5 — RBAC & Fine-Grained Authorization

**Goal:** Real role-based access control mapped from FreeIPA groups, plus at least one
resource-based (attribute/policy-driven) authorization example using Keycloak's
Authorization Services (UMA2) — this is the piece that separates "I set up SSO" from
"I understand access control," which is what recruiters are actually screening for.
**Estimated time:** 3–5 hours

### Checklist

- [ ] Map FreeIPA groups (`it-admins`, `engineers`, `recruiters-demo`) to Keycloak
      realm roles via the LDAP federation's group mapper
- [ ] Assign different Grafana/BookStack/Portainer permissions per role (e.g.
      `it-admins` = full admin in all three, `engineers` = edit-only in BookStack,
      `recruiters-demo` = read-only viewer everywhere)
- [ ] Enable Keycloak Authorization Services (UMA2) on the Grafana or Portainer client
- [ ] Build one resource-based policy — e.g., a specific Grafana dashboard resource
      that only `it-admins` can access regardless of their general Grafana role —
      to demonstrate attribute/resource-scoped authorization, not just coarse RBAC
- [ ] Test the policy with a non-admin user and confirm the fine-grained deny works
      independent of the coarse role

### Sources
- Keycloak Authorization Services (UMA2) guide: https://www.keycloak.org/docs/latest/authorization_services/
- Keycloak group/role LDAP mappers: https://www.keycloak.org/docs/latest/server_admin/#_ldap_mappers

---

## Phase 6 — Provisioning & Deprovisioning Lifecycle

**Goal:** Prove the full identity lifecycle works end-to-end — not just "user logs in"
but "user is created once in FreeIPA, appears everywhere downstream, and disabling them
in one place actually locks them out everywhere."
**Estimated time:** 2–4 hours

### Checklist

- [ ] Create a brand-new user in FreeIPA only — confirm it appears in Keycloak after
      the next federation sync (or trigger Just-In-Time provisioning on first login,
      if configured, rather than waiting for a full sync)
- [ ] Log the new user into all three demo apps to confirm downstream provisioning
      works without manual account creation in any of them
- [ ] Disable the user in FreeIPA and confirm login is rejected at Keycloak — then
      confirm any active sessions in the demo apps are actually terminated, not just
      blocked on next login (Keycloak session management / global logout)
- [ ] **SCIM (stretch goal, treat as a discovery step, not a known quantity):**
      Keycloak does not ship a built-in SCIM server — research current community SCIM
      extensions/connectors for Keycloak before committing to this, since plugin
      compatibility with your installed Keycloak version needs to be verified at build
      time. If a working option is found, wire up SCIM push from Keycloak to one demo
      app and document the provisioning/deprovisioning payloads observed.

### Sources
- Keycloak sessions and logout docs: https://www.keycloak.org/docs/latest/server_admin/#_sessions
- SCIM protocol spec (for reference regardless of which connector is used): https://datatracker.ietf.org/doc/html/rfc7644

---

## Phase 7 — Identity Audit Logging & SIEM Integration

**Goal:** Keycloak login/logout/admin events flow into the existing Wazuh/Splunk stack
from the SOC build, so identity events sit alongside EDR/SIEM alerts in one place —
this is the cross-project piece that ties the IAM build and the SOC build together into
a single, more impressive portfolio story.
**Estimated time:** 3–4 hours
**Depends on:** `wazuh.home.arpa` / Splunk already being up from the SOC build

### Checklist

- [ ] Realm → Events → enable both Login Events and Admin Events, set an appropriate
      retention (Keycloak's in-DB event store isn't meant for long-term retention —
      this is why shipping to Splunk matters)
- [ ] Enable Keycloak's event listener SPI to emit events to a log file or syslog
      target that a Splunk Universal Forwarder (already used in the SOC build) can tail
- [ ] Create a dedicated `keycloak-events` index in Splunk (same pattern as
      `wazuh-alerts` and `windows-events` from the SOC build)
- [ ] Confirm a test login/logout/failed-login shows up in Splunk:
      ```spl
      index="keycloak-events"
      ```
- [ ] Build one Splunk panel correlating a failed Keycloak login with a corresponding
      Wazuh alert if the failed attempts came from a monitored endpoint — this is the
      "unified security picture" payoff mentioned in the SOC doc's Phase 7 capstone

### Sources
- Keycloak events documentation: https://www.keycloak.org/docs/latest/server_admin/#_events
- Splunk Universal Forwarder docs (reused from SOC build): https://docs.splunk.com/Documentation/Splunk/latest/Forwarding/Aboutforwardingandreceivingdata

---

## Phase 8 — Validation & Recruiter Packaging

**Goal:** Turn the working lab into something a recruiter or hiring manager can actually
evaluate in under 5 minutes — screenshots, a clear README, and a short demo script.
**Estimated time:** 2–3 hours

### Checklist

- [ ] Run and screenshot each of: OIDC login (Grafana), SAML login (BookStack), OAuth2
      login (Portainer), MFA challenge, RBAC deny (Phase 5 test), and a deprovisioned
      user being locked out (Phase 6 test)
- [ ] Export the architecture diagram (the ASCII diagram above, cleaned up in a proper
      diagram tool) as an image for the README
- [ ] Write a project README covering: what was built, why Keycloak + FreeIPA were
      chosen, the three protocols demonstrated, and the identity lifecycle proven —
      link back to this planning doc's phases as the structure
- [ ] Draft 2–3 resume bullet points from the finished project (e.g., "Deployed
      self-hosted Keycloak IdP federated with FreeIPA LDAP, implementing SSO across
      OIDC/SAML2/OAuth2 for three applications with group-based RBAC, TOTP MFA, and
      centralized identity audit logging into Splunk")
- [ ] Decide whether to publish configs (docker-compose files, realm export JSON with
      secrets stripped) to a public GitHub repo — a public repo is what most recruiters
      actually check, more than the write-up itself

---

## Key Ports (For Reference)

```
Port  | Protocol | Direction              | Purpose
------|----------|------------------------|------------------------------------
389   | TCP      | Keycloak → FreeIPA      | LDAP (use 636/LDAPS beyond quick testing)
636   | TCP      | Keycloak → FreeIPA      | LDAPS
88    | TCP/UDP  | Clients → FreeIPA       | Kerberos
443   | TCP      | Browser → FreeIPA UI    | FreeIPA web console
8443  | TCP      | Browser → Keycloak      | Keycloak admin console / login (adjust to
      |          |                         | your actual compose port mapping)
```

---

## Roadmap / Future Work

- [ ] Stand up Authentik as a second IdP purely for comparison/portfolio depth (lighter
      footprint, visual flow editor) — write a short "Keycloak vs. Authentik, hands-on"
      comparison once both have been run for real
- [ ] Revisit SCIM once a verified-compatible connector is found (see Phase 6 note)
- [ ] Explore Keycloak's Organizations feature for a simple multi-tenant IAM demo
- [ ] Consider placing FreeIPA/Keycloak traffic under the future VLAN 50 (Cyber Lab)
      zone once that's built, to practice attacking/defending the IAM stack itself
      without touching the production SOC VMs
- [ ] Cross-link this project with the SOC build's Phase 7 unified dashboard so identity
      events, EDR alerts, and Windows events all live on one Splunk view

---

## Troubleshooting Log

*Append to this as issues are encountered. Same format as the network doc and SOC doc.*

---

## Sources & References

**Keycloak**
- Documentation home: https://www.keycloak.org/documentation
- LDAP/User Storage Federation: https://www.keycloak.org/docs/latest/server_admin/#_ldap
- Authorization Services (UMA2): https://www.keycloak.org/docs/latest/authorization_services/
- Authentication flows: https://www.keycloak.org/docs/latest/server_admin/#_authentication-flows
- Events / audit logging: https://www.keycloak.org/docs/latest/server_admin/#_events
- Getting started with Docker: https://www.keycloak.org/getting-started/getting-started-docker

**FreeIPA**
- Documentation: https://freeipa.readthedocs.io/
- Project documentation index: https://freeipa.org/page/Documentation
- Red Hat: Federation with Keycloak and FreeIPA: https://access.redhat.com/solutions/3010401

**Demo Applications**
- Grafana generic OAuth/OIDC: https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-authentication/generic-oauth/
- BookStack SAML2 auth: https://www.bookstackapp.com/docs/admin/saml2-auth/
- Portainer OAuth settings: https://docs.portainer.io/admin/settings/authentication/oauth

**Standards**
- OpenID Connect Core 1.0: https://openid.net/specs/openid-connect-core-1_0.html
- SAML 2.0 Core: https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf
- OAuth 2.0 (RFC 6749): https://datatracker.ietf.org/doc/html/rfc6749
- SCIM Protocol (RFC 7644): https://datatracker.ietf.org/doc/html/rfc7644

**Tool Comparison (background reading used to choose the stack)**
- Keycloak vs Authentik comparison: https://skycloak.io/blog/keycloak-vs-authentik-comparison/
- Authentik vs Authelia vs Keycloak (2026): https://aicybr.com/blog/authentik-vs-authelia-vs-keycloak-sso-guide
- FreeIPA vs Keycloak comparison: https://stackshare.io/stackups/freeipa-vs-keycloak

---

*Document started: July 2026.*
*Cross-references: Homelab-Network-Documentation.md, Homelab-SOC-Build-Plan.md, Homelab-Server-VM-Specs.md*
