# School Computer Laboratory Network Deployment Guide
### Zentyal Server (Community/Development Edition, Samba4 AD DC) + pfSense Firewall/Filtering + Windows 11 Clients — 4-Room Lab Rollout

**Prepared for:** A developer/programmer moving into a systems administration role
**Scope:** Stand up a free, Linux-based Active Directory domain on top of a hypervisor already provisioned on the CCSE network, add pfSense to handle routing and web filtering (a gap in Zentyal's free edition), enforce strict student desktop restrictions, and back up user data to OneDrive/Microsoft 365.

> **How to read this guide if you're coming from software, not sysadmin work:** Most of what follows maps to concepts you already know. A **Domain Controller** is basically a centralized auth server (like an identity provider/OAuth server, but for Windows logins). A **GPO (Group Policy Object)** is Windows' built-in configuration-management system — think Ansible/Puppet, except it's pushed automatically to any machine that's a member of the right "group" (OU) every time it boots or every ~90 minutes. An **OU (Organizational Unit)** is just a folder/namespace you place user and computer accounts into so policies can target them. **Kerberos** is the token-based auth protocol AD runs on — same ideas as OAuth2 tokens with expiry, just older and UDP/TCP-native instead of HTTP. Keep these mappings in mind; they'll make the GUI screens much less mysterious. Throughout the guide, boxes like this one call out concepts worth pausing on.

---

## Architecture Overview

| Component | Choice | Reason |
|---|---|---|
| Hypervisor | Existing dedicated host at **192.168.4.4** on the CCSE network (assumed Proxmox VE — free, open-source; substitute your actual hypervisor if different) | Already provisioned for you; both pfSense and Zentyal will run as VMs on it |
| Firewall / Router / Web Filter | **pfSense CE (Community Edition)**, as a VM on the hypervisor | Zentyal's **Community/Development Edition** has no built-in content/web filtering (that's a paid Zentyal "Gateway" add-on) — pfSense fills that gap for free and also becomes your inter-VLAN router |
| Domain Controller | **Zentyal Server, Community/Development Edition only** (Ubuntu Server 22.04/24.04 LTS base) | Free forever, no subscription required, Samba4 AD-compatible, has a web GUI for the AD/file-sharing pieces we actually need |
| Directory Service | Samba4 AD DC (behind Zentyal's "Domain Controller" module) | Fully interoperable with Windows 11 domain-join, GPOs, Kerberos |
| Internal DNS | Zentyal-managed BIND9, integrated with Samba AD | Required for Kerberos/AD to function — this is non-negotiable, not optional |
| Clients | Windows 11 **Pro** or **Education** (Home edition **cannot** domain-join) | Domain join requires Pro/Education |
| GPO Management | RSAT on an admin workstation | Zentyal's web GUI edits Samba/AD *objects* (users, groups, DNS) but does **not** edit GPOs — those are Windows-native objects stored in SYSVOL |
| Cloud Backup | Microsoft OneDrive (Known Folder Move + Files On-Demand) | Native to your Microsoft 365 tenant, no extra backup server needed |

### A note on Zentyal editions (read this before downloading anything)

Zentyal ships in a few naming variants depending on when you look:

- **Zentyal Server Development Edition** — the free, rolling-release version, community-supported, no subscription.
- **Zentyal Server (Community Edition)** — same idea under newer branding on some download pages.
- **Zentyal Commercial/Server Edition with a subscription** — paid tier with vendor security-patch feeds, remote monitoring, and a few extra modules (including a commercial content-filter/UTM module).

**Use the free Community/Development edition only.** This guide never assumes the paid subscription features exist. The tradeoff you're accepting: you patch the underlying Ubuntu packages yourself via plain `apt`, on your own schedule — there's no vendor-curated update feed nagging you, but there's also no safety net. Put `apt update && apt list --upgradable` on a recurring calendar reminder (or better, in the Ansible playbook in Module 7).

Because the free edition has no content-filtering module, **web filtering moves to pfSense** in this design — see Module 1.

### Network Plan

```
CCSE Network (192.168.4.0/24, campus-managed)
        │
        │  physical hypervisor host: 192.168.4.4
        ▼
┌───────────────────────────────────────────────────────────┐
│  Hypervisor (e.g. Proxmox VE) — 192.168.4.4                │
│                                                             │
│   ┌───────────────┐        ┌───────────────────────────┐  │
│   │ pfSense VM    │        │ Zentyal AD DC VM           │  │
│   │ WAN: 192.168. │        │ 10.0.0.2 (static)           │  │
│   │  4.5 (example,│        │ Samba4 + BIND9 + web GUI    │  │
│   │  confirm with │        │ :8443                       │  │
│   │  network team)│        │                             │  │
│   │ LAN: 10.0.0.1 ├────────┤ Gateway/DNS points to        │  │
│   │  (router +    │        │ 10.0.0.1                    │  │
│   │  DHCP + web   │        │                             │  │
│   │  filter)      │        └───────────────────────────┬─┘  │
│   └───────┬───────┘                                    │    │
└───────────┼──────────────────────────────────────────────────┘
            │ (802.1Q trunk down to the lab switch)
   ┌────────┴─────────┬─────────────┬─────────────┐
Room 1 (VLAN 10)  Room 2 (VLAN 20) Room 3 (VLAN 30) Room 4 (VLAN 40)
10.0.10.0/24       10.0.20.0/24     10.0.30.0/24      10.0.40.0/24
```

Key differences from a "flat" design:

- **pfSense is the router and DHCP server** for all four room VLANs. Zentyal is *not* your DHCP server in this design (it can technically do DHCP, but centralizing it in pfSense keeps one tool responsible for one job — easier to reason about, easier to troubleshoot, and it's where the web filter needs to sit anyway to see all outbound traffic).
- **Zentyal only needs to be reachable, not routing.** It sits on the LAN side (10.0.0.0/24) doing AD + DNS + file sharing.
- Ports that must be allowed **from every room VLAN to Zentyal (10.0.0.2)** through pfSense's inter-VLAN firewall rules: 88 (Kerberos), 389/636 (LDAP/LDAPS), 445 (SMB), 53 (DNS), 123 (NTP). Write one firewall rule per port per VLAN (or one alias covering all four VLANs → one rule per port) — don't skip this, a domain join will fail in confusing ways if any one of these is silently blocked.
- The **WAN-side IP for the pfSense VM** (192.168.4.5 in the diagram) is a placeholder. Since the hypervisor itself already occupies 192.168.4.4 on the CCSE network, you'll need a second free address on that same segment for pfSense's WAN interface — request one from whoever manages the CCSE network rather than guessing.

---

# MODULE 1 — Hypervisor VMs: pfSense and Zentyal

> **Why this module exists that didn't exist before:** Because you already have a hypervisor dedicated at 192.168.4.4, both the firewall and the domain controller are just VMs you create on it, rather than separate physical boxes. If you're used to spinning up cloud instances (EC2, a DigitalOcean droplet, etc.), this is the same mental model — you're just doing it against local hardware instead of an API, at least until Module 7 where we script it.

## 1.1 Confirm Hypervisor Networking Before Creating Anything

1. Log into the hypervisor's own management UI (e.g., Proxmox's web console at `https://192.168.4.4:8006`).
2. Confirm there are (or create) **two virtual networks/bridges**:
   - One bridged directly to the physical CCSE network NIC (this is your "WAN" bridge — pfSense's WAN interface plugs into this).
   - One **internal-only** virtual switch/bridge, not connected to any physical NIC, which will carry `10.0.0.0/24` plus the four room VLANs (10, 20, 30, 40) as 802.1Q tags on the same bridge. In Proxmox this is a Linux Bridge with VLAN awareness enabled; other hypervisors call this a "private" or "internal" vSwitch with VLAN trunking.
3. Write down the exact bridge/vSwitch names — you'll reference them by name when creating both VMs and later in Ansible/Terraform.

## 1.2 Create the pfSense VM

1. Download the **pfSense CE (Community Edition)** ISO from the official Netgate site (free).
2. Create a new VM on the hypervisor with:
   - 2 vCPU, 2–4 GB RAM, 20 GB disk (pfSense is light; this is generous headroom)
   - **Two virtual NICs**: NIC1 attached to the WAN bridge (physical/CCSE-facing), NIC2 attached to the internal LAN bridge (trunked for VLANs 10/20/30/40 plus untagged 10.0.0.0/24)
3. Boot from the ISO and run the pfSense installer (accept defaults; it auto-detects the two NICs).
4. At first boot, pfSense will ask you to assign WAN and LAN interfaces to the two NICs — match them to what you wired in step 2.
5. Set:
   - **WAN**: static IP on the CCSE segment (the address you requested from your network team, e.g. `192.168.4.5/24`), gateway = the CCSE network's existing gateway.
   - **LAN**: `10.0.0.1/24` (this becomes the gateway address for everything behind pfSense, including Zentyal).

## 1.3 Configure VLANs and DHCP for the Four Rooms in pfSense

1. From the pfSense web GUI (`https://10.0.0.1`, reachable once you're on the internal network), go to **Interfaces → Assignments → VLANs** and create four VLANs on the LAN NIC: tags 10, 20, 30, 40.
2. Assign each VLAN to a new interface (`OPT1`…`OPT4`), name them `ROOM1`…`ROOM4`, and give each a static IP acting as that room's gateway:
   - ROOM1: `10.0.10.1/24`
   - ROOM2: `10.0.20.1/24`
   - ROOM3: `10.0.30.1/24`
   - ROOM4: `10.0.40.1/24`
3. Under **Services → DHCP Server**, enable DHCP on each ROOM interface with a scope covering most of the /24 (leave room at the bottom for static reservations for lab PCs if you want predictable hostnames later), and set:
   - **DNS servers pushed to clients:** `10.0.0.2` (Zentyal) — this is mandatory, exactly as it would be in a flat network. Do **not** push a public DNS server here.
   - **Gateway:** the ROOM interface's own IP (pfSense handles this automatically).
4. Under **Firewall → Rules**, on each ROOM interface, add explicit **allow** rules to `10.0.0.2` for ports 53, 88, 123, 389, 636, 445 (create a firewall alias called `ZentyalDC` with the single host `10.0.0.2` to keep this to one rule per port instead of four). Everything else outbound to the internet can stay on whatever your default room-to-WAN policy is; the point here is not blocking the domain-join traffic between rooms and the DC.

## 1.4 Web Filtering in pfSense (this replaces Zentyal's missing content-filter module)

You have two common free approaches. Pick one:

**Option A — pfBlockerNG-devel (DNS-based, simplest, no proxy):**
1. **System → Package Manager → Available Packages**, install `pfBlockerNG-devel`.
2. Configure it in **DNSBL** mode — it works as a DNS sinkhole for categories/blocklists you subscribe it to (adult content, malware, ad networks, etc.).
3. Point client DNS at pfSense's DNSBL listener for filtered categories, or leave DNS pointed at Zentyal (10.0.0.2) and configure Zentyal's BIND to forward *unresolved external* queries to pfSense's DNSBL resolver instead of straight to 8.8.8.8/1.1.1.1 — this keeps internal AD DNS untouched while filtering everything that leaves the network. Test carefully; this forwarder chain is the one place internal DNS resolution and web filtering interact.
4. Advantage: transparent, no client proxy config, low overhead. Disadvantage: category granularity is blocklist-quality, not truly content-aware.

**Option B — Squid + SquidGuard (proxy-based, more granular, supports per-room/per-user policy):**
1. Install `squid` and `squidGuard` packages via the same Package Manager.
2. Configure Squid as a **transparent proxy** on each ROOM interface (so students don't need per-browser proxy settings) or as an explicit proxy pushed via GPO (Module 5) if you want per-user/per-group category rules tied to AD group membership later.
3. SquidGuard gives you category blacklists (subscribe to a free list like Shalla's or UT1) with per-time-of-day or per-source-network overrides — e.g., allow YouTube in Room 3 (media lab) but not Rooms 1/2/4.
4. Advantage: real category granularity, usage logging per source IP. Disadvantage: more moving parts, transparent-proxy interception of HTTPS needs its own SSL-bump configuration and a trusted root cert pushed to clients via GPO — treat this as a follow-up project once DNS-based filtering (Option A) is stable, not a day-one requirement.

Start with Option A. It solves "we have no web filter at all" immediately; move to Option B only if a specific room/policy need (e.g., time-of-day rules) actually shows up.

## 1.5 Create the Zentyal VM

1. Same hypervisor, new VM: 2+ vCPU, 4+ GB RAM (Samba AD + BIND are not heavy, but give it room), 40+ GB disk.
2. **Single virtual NIC**, attached to the internal LAN bridge, **untagged** (i.e., on `10.0.0.0/24`, not one of the room VLANs — Zentyal serves all rooms centrally, it doesn't belong to one).
3. Continue to Module 2 for the actual Zentyal install — everything from here on is unchanged from a bare-metal deployment, just running inside this VM.

---

# MODULE 2 — Zentyal Server Installation & Web GUI Setup

## 2.1 Download and Install Zentyal Server

1. Download the **Zentyal Server Development/Community Edition ISO** (free) from the official Zentyal site — confirm you're on the free download page, not a "buy subscription" page. It ships as a customized Ubuntu Server LTS installer.
2. Attach the ISO to the Zentyal VM created in Module 1.5 and boot from it.
3. Proceed through the standard Ubuntu-based installer:
   - You can leave networking on DHCP for the install itself; you'll set the permanent static IP from the web GUI in step 2.3 below. (If your hypervisor's internal bridge has no DHCP server yet, just set `10.0.0.2/24`, gateway `10.0.0.1`, manually at this stage — either path works.)
   - Set hostname, e.g. `dc1`.
   - Create the initial local admin (Linux) user — this account logs into the web interface.
   - Let the installer finish disk partitioning and base package installation.
4. Reboot when prompted. Zentyal boots into a **console notice** showing the URL for the web GUI, typically:
   ```
   https://<server-ip>:8443/
   ```

## 2.2 First Web GUI Login & Module Selection

1. From an admin workstation on the internal `10.0.0.0/24` network, browse to `https://<server-ip>:8443/`.
2. Accept the self-signed certificate warning (swap in a real cert later if you want; not required for AD to function).
3. Log in with the Linux admin credentials from the install.
4. On first login, Zentyal presents a **package/module selection wizard**. Select:
   - **Domain Controller and File Sharing**
   - **DNS Server**
   - Leave **DHCP Server** and any content-filter modules **unchecked** — DHCP now lives in pfSense (Module 1.3), and content filtering lives there too (Module 1.4). Installing them here as well just creates two systems fighting over the same job.
5. Click **Install** — Zentyal pulls `samba`, `bind9`, `heimdal`/MIT Kerberos, etc. automatically via apt.

## 2.3 Configure Static IP and Network Interface

1. Left sidebar → **Network → Interfaces**. Select the single NIC, set method to **Static**:
   - IP address: `10.0.0.2`
   - Netmask: `255.255.255.0`
   - Gateway: `10.0.0.1` (the pfSense LAN interface — this is the important change from a design without pfSense: Zentyal's default gateway is pfSense, not a generic router)
2. Go to **Network → DNS** (or **DNS Resolver**, depending on version) and add **upstream DNS forwarders** for anything Zentyal's own BIND can't answer internally — point these at pfSense's LAN IP (`10.0.0.1`) if you set up the pfBlockerNG DNS chain in Module 1.4, or at `8.8.8.8`/`1.1.1.1` directly if you haven't yet. Internal AD queries never leave Zentyal's own BIND regardless of this setting.
3. Click **Save Changes** (top-right green button) — Zentyal batches config changes and only applies them on this click, which trips people up the first time.

## 2.4 Configure the Domain Controller Module

1. Navigate to **Users and Computers** (or **Domain**, depending on version) — this triggers the **initial AD provisioning wizard** if it hasn't run yet.
2. Choose **"Provision as the first Domain Controller of a new domain."**
3. Set:
   - **Domain/Realm name (FQDN):** `lab.school.local`
   - **NetBIOS domain name:** `LABSCHOOL` (auto-suggested; 15-char max, no dots)
   - **Server role:** Domain Controller
4. Set the **Directory Manager (DS Restore Mode) password** and store it somewhere safe — this is your AD disaster-recovery password, the equivalent of a root/break-glass credential.
5. Click **Provision**. Zentyal will:
   - Install `samba-ad-dc`
   - Configure Kerberos (`/etc/krb5.conf`)
   - Set up the internal BIND9 zone for `lab.school.local`
   - Create the default AD structure (Users, Computers, Domain Controllers OUs)
6. Confirm success under **Dashboard → Domain Controller widget**, which should read "Running."

## 2.5 Best Practices for Naming & DNS

- **Never** use a public TLD you don't own for your internal domain (avoid `.com`/`.net`); `.local` or a subdomain like `lab.school.local` is Microsoft's recommended safe pattern for internal AD.
- Keep the NetBIOS name ≤15 characters, uppercase, no special characters.
- Set Zentyal's own DNS resolution to point at **itself** (`127.0.0.1` or its static IP) as primary, with pfSense/public DNS configured only as *forwarders* for names outside the domain — this is critical, otherwise Kerberos/SRV record lookups fail intermittently and you'll chase a ghost bug for an afternoon.
- Verify DNS is serving AD SRV records:
  ```bash
  dig _ldap._tcp.lab.school.local SRV
  dig _kerberos._udp.lab.school.local SRV
  ```
  Both should return the DC's hostname and port. If they don't, stop here and fix DNS before touching anything else — nothing downstream works without this.

---

# MODULE 3 — User & Group Architecture

## 3.1 Create Organizational Units (OUs) via Web GUI

> **Translator's note:** an OU is a namespace, like a folder or a Kubernetes namespace — it exists so you can point a policy or a permission at "everything in here" instead of listing every object individually.

1. Go to **Users and Computers** in the left sidebar.
2. Right-click (or use the "+" action menu) on the domain root (`lab.school.local`) → **New → Organizational Unit**.
3. Create:
   - `Students` (top-level OU)
   - `Lab-Admins` (top-level OU)
4. Optionally nest sub-OUs per room for granular GPO targeting later:
   ```
   Students
   ├── Room1
   ├── Room2
   ├── Room3
   └── Room4
   ```
   This lets you apply room-specific policy (different lab software, different filtering exceptions) without duplicating logic elsewhere.

## 3.2 Create Student Accounts as Standard (Non-Admin) Users

1. Right-click the `Students` OU (or the relevant room sub-OU) → **New → User**.
2. Fill in: first name, surname, **username** (becomes the `sAMAccountName`), and a temporary password.
3. **Critical setting:** leave the account as a plain **Domain User**. Do **not** add it to `Domain Admins`, `Administrators`, or any local Administrators group.
4. Check **"User must change password at next logon"** for first-time onboarding.
5. Repeat per student, or use **bulk import**: Zentyal supports **CSV/LDIF import** under **Users and Computers → Import/Export** — much faster for 100+ students across four rooms.
   - Prepare a CSV with columns like `username,firstname,lastname,password,ou`.
   - Import via the GUI's file upload tool.
6. **Why this enforces restriction automatically:** on Windows 11, a domain user who is *not* a member of the local `Administrators` group (or `Domain Admins`) is automatically treated as a **Standard User** — no local admin rights, no installing software, no changing system settings, no bypassing UAC prompts. This one fact is the foundation Module 6's AppLocker/GPO restrictions build on top of.

## 3.3 Create the Lab-Admins Group and an Onboarding Template

1. Under **Users and Computers → Groups**, create a security group `Lab-Admin-Staff`. Add it (or individual IT staff accounts) to the built-in `Domain Admins` group *only* if they need full domain rights — otherwise use **Delegation** (3.5) for least-privilege, the same principle as scoping an IAM role narrowly instead of handing out an admin key.
2. For fast student onboarding, create a **template user** (e.g., `_StudentTemplate`, disabled) with correct OU membership, group memberships, home-directory path, and login-script path pre-set.
3. When creating new students, use Zentyal's **"Copy user"** function (right-click the template → Copy) to clone settings — fill in name/username/password and you're done. This is the closest thing here to a "scaffold" or project template in the way you'd think of it as a developer.

## 3.4 Faculty OU & Elevated Software-Install Rights

Faculty need to install course-specific software on lab PCs (IDEs, CAD tools, subject plugins) without being full Domain Admins and without you touching every machine by hand. The model: **faculty are local admins on lab PCs only, never on the domain itself** — the equivalent of scoping a service account to one resource instead of granting org-wide permissions.

### 3.4.1 Create the Faculty OU and Security Group

1. In **Users and Computers**, create a top-level OU `Faculty` (sibling to `Students` and `Lab-Admins`).
2. Create faculty accounts inside it the same way as students (3.2) — plain **Domain Users**, no domain-level elevation.
3. Create a security group `Lab-Faculty` (**Users and Computers → Groups → New → Security Group**).
4. Add every faculty account to `Lab-Faculty`. This one group is what GPOs will target — never add faculty individually to local admin groups on each PC; that doesn't scale and you'll forget who has what.

```bash
# On the Zentyal server
samba-tool group add Lab-Faculty
samba-tool group addmembers Lab-Faculty jsmith,mgarcia,rwong
```

### 3.4.2 Grant Local Admin on Lab PCs Only (Group Policy Preferences)

This is the key step: faculty get admin rights **on the lab computers**, not the domain, not servers, not via a domain-wide privileged group.

1. In `gpmc.msc`, create a GPO `Faculty-LocalAdmin-LabPCs`, linked **only** to the OU containing the lab computer objects (e.g., `Computers\Room1-PCs` … `Room4-PCs` — link to all four, not to `Students` or `Faculty` user OUs, since this is a *computer-side* policy).
2. Edit the GPO → **Computer Configuration → Preferences → Control Panel Settings → Local Users and Groups**.
3. Right-click → **New → Local Group**:
   - Group name: **Administrators (built-in)**
   - Action: **Update**
   - Members → Add → `LABSCHOOL\Lab-Faculty` → check **Add to this group**.
4. This uses item-level targeting, so `Lab-Faculty` gets merged into the local `Administrators` group on every lab PC the GPO applies to, on next `gpupdate`/reboot. Students are unaffected, and faculty gain zero rights on the DC, pfSense, or non-lab machines.
5. Verify on a test client:
   ```cmd
   net localgroup administrators
   ```
   should list `LABSCHOOL\Lab-Faculty` alongside the built-in `Administrator`.

**Why not the older "Restricted Groups" node instead?** Restricted Groups *overwrites* the entire local Administrators membership on every refresh, which can silently strip the built-in local `Administrator` account or other needed entries. Group Policy Preferences with **"Update"** is additive and safer here — use Restricted Groups only if you deliberately want a fixed, exact membership list.

### 3.4.3 Scope AppLocker So It Doesn't Block Faculty (see Module 6.2)

Local admin rights alone don't bypass AppLocker — AppLocker rules apply regardless of local admin status unless you explicitly scope a rule to the `Lab-Faculty` group. Handled in **Module 6.2**.

### 3.4.4 Faculty Software-Install Workflow (Day-to-Day)

With this setup, a teacher can, at the start of term:
1. Log into any lab PC in their room with their normal faculty domain account.
2. Right-click an installer → **Run as administrator** (UAC prompts but succeeds, since `Lab-Faculty` is a local admin) — no separate elevated credential needed.
3. Install the course software once per room, or push it via Module 6.3's GPO Software Installation / Chocolatey method if it needs to go to all rooms centrally.

For anything touching **multiple rooms**, or anything that should be standardized, prefer the admin-pushed methods in Module 6.3 over relying on individual faculty installs — it keeps configuration consistent and auditable, the same reasoning you'd apply to "don't let every service hand-configure itself, use one deploy pipeline."

## 3.5 Delegate Limited Admin Rights (Optional, Recommended)

Instead of making lab teachers full `Domain Admins`, delegate only what they need (e.g., password resets for the `Students` OU):
```bash
samba-tool ou list
samba-tool delegation add-service-account <lab-admin-user> "Reset Password" --ou="OU=Students,DC=lab,DC=school,DC=local"
```
This limits blast radius if a lab-admin account is compromised — the AD equivalent of a narrowly-scoped IAM policy instead of a wildcard `*` grant.

---

# MODULE 4 — Windows 11 Client Provisioning

## 4.1 Network Adapter / DNS Configuration

1. On each Windows 11 machine: **Settings → Network & Internet → Ethernet/Wi-Fi → Edit DNS settings** (or legacy Control Panel → Network Adapter → Properties → IPv4).
2. Set:
   - **Preferred DNS:** `10.0.0.2` (Zentyal) — mandatory; Windows cannot locate the domain via SRV records otherwise.
   - **Alternate DNS:** leave blank, or a secondary DC's IP if you build one later. Do **not** put pfSense's WAN-facing or public DNS here, or domain lookups will silently fail.
   - Since this machine is on a room VLAN behind pfSense, its **gateway** should already be correct automatically via the pfSense DHCP scope for that room (10.0.10.1, 10.0.20.1, etc.) — you shouldn't need to touch this manually if pfSense DHCP is working.
3. Confirm resolution:
   ```cmd
   nslookup lab.school.local
   ping dc1.lab.school.local
   ```

## 4.2 Join the Domain

1. **Settings → Accounts → Access work or school → Connect** → **"Join this device to a local Active Directory domain"** (Windows 11 **Pro/Education** only — not Home).
2. Enter the domain name: `lab.school.local`.
3. Authenticate with a domain account that has rights to join computers (a Lab-Admin or Domain Admin account — **not** a student account).
4. Choose which OU the computer object lands in (or accept the default `Computers` OU and move it later via the Zentyal GUI or `samba-tool computer move`).
5. Restart when prompted.
6. At the new logon screen, click **"Other user"** and log in as `LABSCHOOL\studentusername`.

## 4.3 Verification & Troubleshooting Commands (Client-Side)

Run these in an elevated Command Prompt or PowerShell on the Windows 11 client:

```cmd
:: Confirm domain membership
echo %USERDOMAIN%
systeminfo | findstr /B /C:"Domain"

:: Verify secure channel to the DC
nltest /sc_query:lab.school.local

:: Force Kerberos time sync (fixes "clock skew" / KRB_AP_ERR_SKEW errors)
w32tm /config /manualpeerlist:"dc1.lab.school.local" /syncfromflags:manual /reliable:yes /update
w32tm /resync /force

:: Flush and re-register DNS
ipconfig /flushdns
ipconfig /registerdns

:: Confirm Kerberos ticket issuance
klist
```

**Important:** Kerberos requires client and DC clocks to be within **5 minutes** of each other by default. If joins fail with time-sync errors, fix `w32tm` **before** retrying the domain join, not after — this is the single most common "it just doesn't work" support ticket you'll get.

---

# MODULE 5 — OneDrive Cloud Backup & Storage Configuration

## 5.1 Install RSAT to Manage GPOs

Zentyal's web GUI manages Samba/AD objects (users, groups, DNS) but has **no native GPO editor** — you manage Group Policy from a Windows machine using RSAT, same as with a traditional Windows Server AD DC (Samba4 stores GPOs in SYSVOL in Windows-compatible format).

1. On an admin Windows 11 Pro/Education workstation (already domain-joined): **Settings → Optional Features → Add a feature**.
2. Install:
   - **RSAT: Group Policy Management Tools**
   - **RSAT: Active Directory Domain Services and Lightweight Directory Tools**
3. Launch **Group Policy Management** (`gpmc.msc`) — it should connect to `lab.school.local` automatically since the machine is domain-joined.

## 5.2 Add OneDrive Administrative Templates to the Central Store

1. Download the latest **OneDrive Administrative Templates** package from Microsoft (`OneDrive.admx` / `OneDrive.adml`).
2. Create the **Central Store** if it doesn't exist yet (a SYSVOL folder structure Samba4 already replicates):
   ```
   \\lab.school.local\SYSVOL\lab.school.local\Policies\PolicyDefinitions\
   \\lab.school.local\SYSVOL\lab.school.local\Policies\PolicyDefinitions\en-US\
   ```
3. Copy:
   - `OneDrive.admx` → `...\PolicyDefinitions\`
   - `OneDrive.adml` → `...\PolicyDefinitions\en-US\`
4. Reopen **Group Policy Management Editor** — under **Computer Configuration → Administrative Templates**, you should now see a **OneDrive** folder.

## 5.3 GPO: Silent Account Configuration

1. In `gpmc.msc`, right-click the `Students` OU → **Create a GPO in this domain, and Link it here**. Name it `OneDrive-SilentConfig`.
2. Edit → **Computer Configuration → Policies → Administrative Templates → OneDrive**:
   - Enable **"Silently move Windows known folders to OneDrive"** *(fully covered in 5.4)*
   - Enable **"Silently sign in users to the OneDrive sync app with their Windows credentials"**
   - Enable **"Set the default location for the OneDrive folder"** if desired.
3. Enable and configure **"Allow silent account configuration"**, entering your school's **Microsoft 365 Tenant ID** (Microsoft 365 Admin Center → Settings → Org Settings → Organization Profile, or `Get-MgOrganization` in Microsoft Graph PowerShell).
4. This makes OneDrive auto-configure using the signed-in Windows credential — no manual setup wizard for students. (Since these accounts are on-prem AD, not Azure AD-joined, students will still see one M365 login prompt unless you've deployed Entra Connect Sync — see the note below.)

> **Hybrid-Identity Note:** Pure Samba4 AD accounts aren't automatically known to Microsoft 365/Entra ID. For OneDrive Silent Config and SSO to work seamlessly you need either (a) matching UPNs plus a **directory sync** solution (Azure AD Connect classically requires Windows Server — as a free alternative, run **Entra Connect Sync** from a small dedicated Windows VM on the same hypervisor), or (b) students entering their school M365 email/password once at first OneDrive login. Plan identity-matching (same username = same email prefix) during Module 3 account creation to simplify this later.

## 5.4 GPO: Known Folder Move (KFM) + Files On-Demand

1. In the same or a new GPO linked to `Students`, go to **Computer Configuration → Administrative Templates → OneDrive**:
   - Enable **"Silently move Windows known folders to OneDrive"** → enter your **Tenant ID**.
   - Enable **"Prevent users from redirecting their Windows known folders to their PC"** (stops students reversing the redirect).
2. Enable **Files On-Demand**: set to **Enabled** (keeps files as online-only placeholders until opened, saving local disk space — ideal for shared lab PCs with small SSDs).
3. Run `gpupdate /force` on a test client, sign in, and confirm:
   - `Desktop`, `Documents`, `Pictures` show the OneDrive cloud icon and redirect to `%USERPROFILE%\OneDrive - School\...`.
   - Files show cloud/check icons (Files On-Demand working).
4. **Recommended companion setting:** leave **"Block monitoring known folders..."** *Disabled* (i.e., leave monitoring on) so KFM notifications work correctly, and apply **"Use OneDrive Files On-Demand"** at the Computer Configuration level so it takes effect before any user profile loads (avoids race conditions on shared lab machines).

---

# MODULE 6 — Desktop Lockdown & Application Restrictions

> Web/content filtering is handled entirely by pfSense now (Module 1.4). This module is only about what runs **locally** on the Windows machine — what a student can execute, not what sites they can reach.

## 6.1 Block Execution from Flash Drives / Downloads

1. Create a GPO `Student-ExecutionLockdown`, linked to the `Students` OU.
2. Prefer **AppLocker** (6.2) over the legacy **Software Restriction Policies** node — SRPs are largely superseded.
3. If you additionally want to hard-block **removable media execution** regardless of AppLocker:
   - **Computer Configuration → Administrative Templates → System → Removable Storage Access** → Enable **"All Removable Storage classes: Deny all access"**, or more surgically, **"Removable Disks: Deny execute access."**
4. Combined with AppLocker, this ensures a student can't run a portable `.exe` from a USB stick or a browser-downloaded file even if they copy it locally.

## 6.2 AppLocker Path-Rule Whitelist

1. **Computer Configuration → Policies → Windows Settings → Security Settings → Application Control Policies → AppLocker**.
2. Right-click **Executable Rules** → **Create Default Rules** (baseline allow rules for `Program Files` and `Windows`), then tighten to exactly:
   - **Allow:** Path = `%PROGRAMFILES%\*`
   - **Allow:** Path = `%WINDIR%\*`
   - **Delete** or **exclude** any default rule whitelisting `%OSDRIVE%\*` broadly, or "all files" for `BUILTIN\Users` — that's the rule that would defeat the whole lockdown.
3. Configure **Windows Installer Rules**, **Script Rules**, and **Packaged app Rules** the same way, so `.msi`, `.ps1`, `.bat`, and UWP sideloading are equally restricted.
4. Enable the AppLocker service: same GPO node → **AppLocker → Properties** → check **"Configured"** for each rule type.
5. Ensure the **Application Identity service** starts automatically (required for enforcement):
   - **Computer Configuration → Preferences → Control Panel Settings → Services**, add `AppIDSvc` set to Automatic, or push via startup script: `sc config AppIDSvc start=auto`.
6. Set enforcement to **"Enforce rules"** (not "Audit only") once testing confirms the whitelist doesn't block legitimate lab software.
7. Test on one machine per room before wide rollout — misconfigured AppLocker can lock out portable STEM/coding tools that don't install into `Program Files`; either reinstall them properly or add a narrowly-scoped publisher/hash rule.

### 6.2.1 Faculty Exception (allow installs without opening the whitelist to students)

Local admin rights (Module 3.4) let faculty run an installer's UAC prompt, but AppLocker still evaluates every executable regardless of who's running it — so without an explicit rule, faculty are blocked too. Scope a dedicated rule to `Lab-Faculty` only:

1. In the same GPO (or a separate one linked only to the lab computer OUs — AppLocker is computer-scoped, so the *user* condition inside the rule is what limits it to faculty), right-click **Executable Rules** → **Create New Rule**.
2. **Permissions:** Action = **Allow**, User or group = `LABSCHOOL\Lab-Faculty` (not "Everyone").
3. **Conditions:** a **Path** rule → `%OSDRIVE%\*` (allow execution anywhere on the drive), or more conservatively, scope it to a dedicated `C:\FacultyInstalls\` staging folder if you want installers copied there first.
4. Repeat for **Windows Installer Rules** (so faculty can run `.msi`) and **Script Rules** if faculty need setup scripts.
5. Because AppLocker matches by user/group SID, students (not in `Lab-Faculty`) still only match the restrictive `Program Files`/`Windows` rules from 6.2 — this new rule just adds a second, wider allow path that only resolves for faculty logons.
6. Confirm on a test lab PC: student login → random `.exe` in Downloads fails to launch (AppLocker block dialog). Faculty login → same file runs.

## 6.3 Pushing Software Updates While Staying Locked Down

Because students can't install anything, all software deployment must be **admin-pushed**. Free options:

**Option A — GPO Software Installation (native, simplest):**
1. Place `.msi` installers on a network share (e.g., `\\dc1\Software$`) readable by Domain Computers.
2. In a GPO linked to your **Computers** OU (not Students — this targets machines): **Computer Configuration → Policies → Software Settings → Software Installation → New → Package**, point to the `.msi` on the UNC path, choose **Assigned**.
3. Software installs automatically at next machine boot — no student interaction, no admin rights needed on the student's session.

**Option B — Free Deployment Tooling (for non-MSI / bulk app management):**
- **Chocolatey** (free, open-source): push via a **startup script GPO** running as `SYSTEM`:
  ```powershell
  Set-ExecutionPolicy Bypass -Scope Process -Force
  iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
  choco upgrade all -y
  ```
  Deploy as a **Computer Startup Script** (GPO: Computer Config → Windows Settings → Scripts → Startup) so it runs with SYSTEM privileges before student logon — SYSTEM-run scripts aren't subject to the same lockdown as the student session.
- **PDQ Deploy (Free tier)** — GUI-based, good for pushing `.exe`/`.msi` on a schedule across all four rooms from one console.
- **WSUS** (free, runs on Windows, or `wsus-offline` as a Linux-hosted front-end) for Windows Update management specifically, keeping OS patches consistent without giving students Windows Update access.

---

# MODULE 7 — Infrastructure as Code (Terraform + Ansible)

> **Why this module changed the most:** the original bare-metal design had nothing for Terraform to declare — there was no VM resource, just physical hardware. Now that everything (pfSense, Zentyal) lives on a hypervisor, Terraform has a real, useful job: **provisioning the VMs themselves.** Ansible still owns everything *inside* those VMs (Samba/AD object management, GPO-equivalent registry pushes, Windows client config). Think of it the way you'd split `terraform apply` (creates the EC2 instance) from a config-management run (installs and configures software on it) — same split here, just against your own hypervisor instead of AWS.

## 7.0 Honest Caveats Before You Commit to This

- **Zentyal's GUI is not the source of truth once you do this.** Zentyal stores its config in a Redis-backed model behind the web GUI; there's no clean, stable API/CLI for "config as code" against Zentyal itself. Treat **Samba4 (`samba-tool`) as the real source of truth** and use Zentyal's GUI purely as an optional read/monitoring dashboard. Changes made *through* the GUI after this point won't show up in your Ansible state unless someone remembers to update the playbooks too — pick one source of truth and stick to it, or you'll get drift.
- **`samba-tool` commands are not natively idempotent** (e.g., `samba-tool user create` fails loudly if the user exists). Every task below wraps calls with existence checks so re-running the playbook is safe — the same discipline you'd apply to any shell-based provisioner.
- **GPOs are semi-opaque as code.** GPOs live in SYSVOL partly as binary `registry.pol` and partly as XML (Preferences/AppLocker). Two viable strategies:
  1. **Backup/Restore roundtrip** — build the GPO once by hand in `gpmc.msc`, `Backup-GPO` it to a folder, commit that folder to git, `Import-GPO` it onto fresh environments. Simple, but git diffs on the binary blobs are unreadable.
  2. **Granular registry-value management** — for the specific OneDrive/AppLocker settings this guide uses, push the exact registry values Ansible can read/diff. More "real IaC"; shown below for OneDrive. AppLocker specifically is better handled via its dedicated XML export/import cmdlets (`Export-AppLockerPolicy` / `Set-AppLockerPolicy`), also shown below.
- **WinRM chicken-and-egg problem.** Ansible needs WinRM enabled to manage a Windows 11 client, but a freshly domain-joined PC doesn't have WinRM enabled by default. You need *one* bootstrap mechanism before Ansible can take over — either a GPO that enables the WinRM listener + firewall rule domain-wide (one-time, via `gpmc.msc`/RSAT, not Ansible), or Microsoft's `ConfigureRemotingForAnsible.ps1` run once per machine during imaging.
- **pfSense config isn't Ansible-native either.** pfSense stores its entire config in one XML file (`/cfg/config.xml`). There's a community Ansible collection (`pfsensible.core`) that wraps some of this via the pfSense API/SSH, but it doesn't cover every screen (VLANs and package installs especially are still easiest done once by hand). Treat pfSense the same way as GPOs: script what the collection supports, and version-control an exported `config.xml` backup for the rest.

## 7.1 Repository Layout

```
lab-iac/
├── terraform/
│   ├── main.tf                    # provider (Proxmox/other) + VM resources for pfSense & Zentyal
│   ├── variables.tf               # hypervisor host (192.168.4.4), bridge names, IPs
│   └── outputs.tf
├── ansible.cfg
├── inventory/
│   ├── hosts.yml                  # dc1, pfsense, plus all lab PCs grouped by room
│   └── group_vars/
│       ├── all.yml                # domain name, tenant ID, etc.
│       └── windows_clients.yml    # winrm connection vars
├── data/
│   ├── students.yml               # source of truth for student accounts
│   ├── faculty.yml
│   └── computers.yml              # hostname -> room/OU mapping
├── roles/
│   ├── zentyal_samba_dc/
│   ├── ad_objects/
│   ├── pfsense_baseline/          # VLANs, DHCP scopes, firewall aliases/rules, pfBlockerNG
│   ├── windows_client_join/
│   ├── onedrive_policy/
│   ├── applocker_policy/
│   └── software_push/
├── gpo-backups/                   # only if using the backup/restore GPO strategy
│   └── OneDrive-SilentConfig/     # raw Backup-GPO output, committed as-is
├── pfsense-backups/
│   └── config.xml                 # exported pfSense config, committed for review/diff
└── site.yml
```

## 7.2 Terraform: Provisioning the VMs

If your hypervisor is Proxmox VE, the `bpg/proxmox` (or `Telmate/proxmox`) provider can create both VMs declaratively:

```hcl
# terraform/main.tf
terraform {
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.60"
    }
  }
}

provider "proxmox" {
  endpoint = "https://192.168.4.4:8006/"
  api_token = var.proxmox_api_token
  insecure  = true # self-signed cert on a LAN-only management IP
}

resource "proxmox_virtual_environment_vm" "pfsense" {
  name      = "pfsense-fw"
  node_name = "pve" # your hypervisor's node name
  cpu       { cores = 2 }
  memory    { dedicated = 4096 }

  network_device { bridge = var.wan_bridge }   # CCSE-facing
  network_device { bridge = var.lan_bridge, vlan_id = null } # trunk, VLAN tagging handled inside pfSense

  cdrom { file_id = "local:iso/pfSense-CE.iso" }
  disk  { interface = "scsi0", size = 20 }
}

resource "proxmox_virtual_environment_vm" "zentyal" {
  name      = "dc1"
  node_name = "pve"
  cpu       { cores = 2 }
  memory    { dedicated = 4096 }

  network_device { bridge = var.lan_bridge } # untagged, 10.0.0.0/24

  cdrom { file_id = "local:iso/zentyal-server.iso" }
  disk  { interface = "scsi0", size = 40 }
}
```

Run once to stand up empty VMs; the actual OS install still happens interactively off the ISO the first time (Terraform provisions the VM shell, not what happens inside the installer wizard) — after that, Ansible takes over everything configuration-related, same division of labor as before.

## 7.3 Control Node Prerequisites

On the Linux machine that will *run* Ansible (can be the Zentyal VM itself, or a separate admin box):

```bash
pip install --user ansible pywinrm requests-ntlm requests-kerberos
ansible-galaxy collection install ansible.windows community.windows microsoft.ad pfsensible.core
```

`microsoft.ad` provides modules like `microsoft.ad.user`, `microsoft.ad.ou`, `microsoft.ad.group` that speak native AD/LDAP against **any** AD-compatible DC, including Samba4 — generally a better fit than hand-rolling `samba-tool` shell calls where the collection covers your use case; fall back to raw `samba-tool` for anything it doesn't support (e.g., initial domain provisioning).

## 7.4 Data-Driven Source of Truth

`data/students.yml`:
```yaml
students:
  - username: jdoe
    first_name: John
    last_name: Doe
    ou: "OU=Room1,OU=Students,DC=lab,DC=school,DC=local"
    groups: [Students]
  - username: agarcia
    first_name: Ana
    last_name: Garcia
    ou: "OU=Room2,OU=Students,DC=lab,DC=school,DC=local"
    groups: [Students]
```

`data/faculty.yml`:
```yaml
faculty:
  - username: jsmith
    first_name: Jane
    last_name: Smith
    ou: "OU=Faculty,DC=lab,DC=school,DC=local"
    groups: [Lab-Faculty]
```

`data/computers.yml`:
```yaml
rooms:
  Room1: { ou: "OU=Room1-PCs,OU=Computers,DC=lab,DC=school,DC=local", hosts: [PC-ROOM1-01, PC-ROOM1-02] }
  Room2: { ou: "OU=Room2-PCs,OU=Computers,DC=lab,DC=school,DC=local", hosts: [PC-ROOM2-01, PC-ROOM2-02] }
```

Every account/computer this guide creates should come from these files, not manual GUI clicks, so `git diff` on this repo *is* your change log for who has access to what.

## 7.5 Role: `zentyal_samba_dc` (Server Bootstrap)

```yaml
# roles/zentyal_samba_dc/tasks/main.yml
- name: Install Zentyal core + Samba AD packages
  apt:
    name: [zentyal-samba, zentyal-dns, samba, krb5-user]
    state: present
    update_cache: true

- name: Check whether the domain is already provisioned
  command: samba-tool domain info 127.0.0.1
  register: domain_check
  ignore_errors: true
  changed_when: false

- name: Provision the AD domain (only if not already provisioned)
  command: >
    samba-tool domain provision
    --realm={{ ad_realm }}
    --domain={{ ad_netbios }}
    --server-role=dc
    --dns-backend=SAMBA_INTERNAL
    --adminpass={{ ad_admin_password }}
  when: domain_check.rc != 0

- name: Template static network config (gateway is now pfSense LAN IP)
  template:
    src: 01-netcfg.yaml.j2
    dest: /etc/netplan/01-netcfg.yaml
    mode: "0600"
  notify: apply netplan

- name: Ensure DNS forwarders point at pfSense
  lineinfile:
    path: /etc/samba/smb.conf
    regexp: '^\s*dns forwarder'
    line: "    dns forwarder = 10.0.0.1"
  notify: restart samba-ad-dc
```

`ad_realm`, `ad_netbios`, `ad_admin_password` live in `inventory/group_vars/all.yml` (vault-encrypt the password: `ansible-vault encrypt_string`).

## 7.6 Role: `pfsense_baseline` (VLANs, DHCP, Firewall Aliases, pfBlockerNG)

```yaml
# roles/pfsense_baseline/tasks/main.yml
- name: Ensure the ZentyalDC alias exists (used by per-room firewall rules)
  pfsensible.core.pfsense_alias:
    name: ZentyalDC
    type: host
    address: 10.0.0.2
    state: present

- name: Ensure a firewall rule per required port exists on each room interface
  pfsensible.core.pfsense_rule:
    name: "Allow {{ item.port_name }} to Zentyal DC"
    action: pass
    interface: "{{ room_iface }}"
    source: any
    destination: ZentyalDC
    destination_port: "{{ item.port }}"
    protocol: "{{ item.proto }}"
    state: present
  loop:
    - { port_name: "DNS",       port: 53,  proto: "tcp/udp" }
    - { port_name: "Kerberos",  port: 88,  proto: "tcp/udp" }
    - { port_name: "NTP",       port: 123, proto: "udp" }
    - { port_name: "LDAP",      port: 389, proto: "tcp" }
    - { port_name: "LDAPS",     port: 636, proto: "tcp" }
    - { port_name: "SMB",       port: 445, proto: "tcp" }
  loop_control: { loop_var: item }
  vars: { room_iface: "{{ item.room_iface | default(ansible_loop.index) }}" }
```

Because `pfsensible.core`'s coverage of every pfSense screen (VLAN creation, package installation like pfBlockerNG) is incomplete, treat **VLAN setup and package installs as one-time manual steps** (Module 1.3/1.4) and let this role manage only the ongoing, repetitive parts — firewall rules and aliases — that actually change as rooms/policies evolve.

## 7.7 Role: `ad_objects` (OUs, Groups, Users from Data Files)

```yaml
# roles/ad_objects/tasks/main.yml
- name: Ensure OUs exist
  command: samba-tool ou create "{{ item }}"
  loop: "{{ all_ous }}"
  register: ou_result
  failed_when: ou_result.rc != 0 and 'already exists' not in ou_result.stderr
  changed_when: "'already exists' not in ou_result.stderr"

- name: Ensure security groups exist
  command: samba-tool group add "{{ item }}"
  loop: [Students, Lab-Faculty, Lab-Admin-Staff]
  register: grp_result
  failed_when: grp_result.rc != 0 and 'already exists' not in grp_result.stderr
  changed_when: "'already exists' not in grp_result.stderr"

- name: Create student users
  command: >
    samba-tool user create {{ item.username }} {{ default_temp_password }}
    --given-name="{{ item.first_name }}" --surname="{{ item.last_name }}"
    --userou="{{ item.ou }}"
  loop: "{{ students }}"
  register: user_result
  failed_when: user_result.rc != 0 and 'already exists' not in user_result.stderr
  changed_when: "'already exists' not in user_result.stderr"

- name: Add students to their groups
  command: samba-tool group addmembers "{{ g }}" "{{ item.username }}"
  loop: "{{ students }}"
  loop_control: { loop_var: item }
  vars: { g: "Students" }
  changed_when: true
```

Repeat for `faculty.yml` targeting `Lab-Faculty`. Because every `command` task checks stderr for "already exists" before failing, re-running `ansible-playbook site.yml` after adding three new rows to `students.yml` only creates those three — safe to run every time you onboard students mid-term.

## 7.8 Role: `windows_client_join` (WinRM-Managed Clients)

`inventory/group_vars/windows_clients.yml`:
```yaml
ansible_connection: winrm
ansible_winrm_transport: kerberos
ansible_port: 5985
ansible_user: "LABSCHOOL\\lab-admins-svc"
ansible_password: "{{ vault_lab_admin_password }}"
```

```yaml
# roles/windows_client_join/tasks/main.yml
- name: Set static DNS to point at the DC
  ansible.windows.win_dns_client:
    adapter_names: "*"
    dns_servers: ["10.0.0.2"]

- name: Join the AD domain
  microsoft.ad.membership:
    dns_domain_name: lab.school.local
    domain_admin_user: "administrator@LAB.SCHOOL.LOCAL"
    domain_admin_password: "{{ vault_ad_admin_password }}"
    state: domain
  register: join_result

- name: Reboot if domain join changed state
  ansible.windows.win_reboot:
  when: join_result.reboot_required

- name: Force Kerberos time sync
  ansible.windows.win_command: >
    w32tm /config /manualpeerlist:"dc1.lab.school.local" /syncfromflags:manual /reliable:yes /update

- name: Flush and re-register DNS
  ansible.windows.win_command: ipconfig /flushdns

- name: Install RSAT features on admin workstations only
  ansible.windows.win_optional_feature:
    name:
      - Rsat.GroupPolicy.Management.Tools~~~~0.0.1.0
      - Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0
    state: present
  when: "'admin_workstations' in group_names"
```

## 7.9 OneDrive Policy as Granular Registry State (True IaC, No Binary GPO Blob)

```yaml
# roles/onedrive_policy/tasks/main.yml
- name: OneDrive silent account configuration + KFM (registry-level, git-diffable)
  ansible.windows.win_regedit:
    path: HKLM:\SOFTWARE\Policies\Microsoft\OneDrive
    name: "{{ item.name }}"
    data: "{{ item.data }}"
    type: "{{ item.type }}"
  loop:
    - { name: "KFMOptInWithWizard", data: "{{ m365_tenant_id }}", type: string }
    - { name: "KFMBlockOptOut",     data: 1, type: dword }
    - { name: "FilesOnDemandEnabled", data: 1, type: dword }
    - { name: "SilentAccountConfig", data: 1, type: dword }
```

`git blame` on this file tells you exactly who changed the tenant ID and when, which a GPO backup blob won't.

## 7.10 AppLocker as Exported XML Policy

```yaml
# roles/applocker_policy/tasks/main.yml
- name: Push the version-controlled AppLocker policy
  ansible.windows.win_copy:
    src: files/applocker-policy.xml   # generated once via Export-AppLockerPolicy -Xml, then hand-edited/reviewed
    dest: C:\Windows\Temp\applocker-policy.xml

- name: Apply AppLocker policy locally
  ansible.windows.win_shell: >
    Set-AppLockerPolicy -XmlPolicy C:\Windows\Temp\applocker-policy.xml -Merge

- name: Ensure Application Identity service is running (required for enforcement)
  ansible.windows.win_service:
    name: AppIDSvc
    start_mode: auto
    state: started
```

`files/applocker-policy.xml` encodes exactly the rules from Module 6.2/6.2.1 (Program Files/Windows allow for everyone, wider allow scoped to the `Lab-Faculty` SID). Regenerate it whenever rules change via `Export-AppLockerPolicy -Xml` from a reference machine, then commit the diff for review before rolling out.

## 7.11 Role: `software_push` (Faculty/Admin-Requested Course Software)

```yaml
# roles/software_push/tasks/main.yml
- name: Install course software via Chocolatey
  chocolatey.chocolatey.win_chocolatey:
    name: "{{ item }}"
    state: present
  loop: "{{ room_software_list }}"
```

`room_software_list` is defined per room in `inventory/group_vars/`, so Room 3 (CAD lab) and Room 1 (intro programming) carry different package lists while sharing every other role.

## 7.12 Running It & Keeping Drift in Check

```bash
terraform -chdir=terraform apply             # only needed when hardware/VM topology changes
ansible-playbook -i inventory/hosts.yml site.yml --vault-password-file .vault-pass
```

- Run the Ansible playbook after **any** change to `data/*.yml` — new student, new faculty member, room reassignment.
- Consider a nightly `cron` run in **check mode** (`--check --diff`) that emails you a drift report (e.g., someone manually added a local admin through a GUI) without applying changes automatically — a lightweight, free substitute for a full compliance/CI product.
- Export pfSense's `config.xml` (**Diagnostics → Backup & Restore → Download configuration**) on a schedule too, and commit it to `pfsense-backups/` — it's your only real "diff" on firewall/filtering changes until `pfsensible.core`'s coverage grows.
- If this grows beyond what cron + a shared terminal can manage, free self-hosted options for scheduled/triggered runs include **Semaphore UI** or **AWX** (upstream of Red Hat Ansible Automation Platform) — both installable on the same hypervisor as a small companion VM.

---

# ADMINISTRATOR'S CHEAT SHEET

## Samba/Zentyal `samba-tool` Essentials (run on the Zentyal VM, as root/sudo)

```bash
# Domain / DC health
samba-tool domain info 127.0.0.1
samba-tool dbcheck --cross-ncs
samba-tool drs showrepl

# User management
samba-tool user list
samba-tool user create jdoe P@ssw0rd123 --given-name=John --surname=Doe --userou="OU=Students,DC=lab,DC=school,DC=local"
samba-tool user setpassword jdoe
samba-tool user disable jdoe
samba-tool user delete jdoe

# Group management
samba-tool group list
samba-tool group addmembers Students jdoe
samba-tool group listmembers Domain Admins

# OU management
samba-tool ou list
samba-tool ou create "OU=Room1,OU=Students,DC=lab,DC=school,DC=local"

# Computer objects
samba-tool computer list
samba-tool computer move PC-ROOM1-05 "OU=Room1-PCs,DC=lab,DC=school,DC=local"

# GPO management (from Linux side, limited — prefer RSAT for editing)
samba-tool gpo listall
samba-tool gpo backup <GPO-GUID> --path=/tmp/gpo-backup

# Kerberos / time
samba-tool domain level show
kinit administrator@LAB.SCHOOL.LOCAL
klist
```

## pfSense Essentials (web GUI paths, and console/SSH where noted)

| Task | Where |
|---|---|
| View/edit firewall rules per room | Web GUI → **Firewall → Rules → [ROOM interface]** |
| View/edit VLAN assignments | Web GUI → **Interfaces → Assignments → VLANs** |
| View DHCP leases per room | Web GUI → **Status → DHCP Leases** |
| Check pfBlockerNG DNSBL stats | Web GUI → **Firewall → pfBlockerNG → DNSBL** |
| Export config for backup/IaC | Web GUI → **Diagnostics → Backup & Restore → Download configuration** |
| Console/SSH packet filter check | `pfctl -sr` (show active ruleset), `pfctl -ss` (show current states) |
| Restart the whole filter | Console menu option 8 (Shell), then `pfctl -f /tmp/rules.debug`, or just **Diagnostics → Reboot** for a clean reload |
| Check WAN connectivity from the console | Console menu option 1 (interfaces), or `ping -c4 <upstream-gateway>` from the shell |

## Network Verification Commands

| Purpose | Command (run on server or client as noted) |
|---|---|
| Verify AD DNS SRV records (server) | `dig _ldap._tcp.lab.school.local SRV` |
| Verify Kerberos SRV records (server) | `dig _kerberos._udp.lab.school.local SRV` |
| Test LDAP bind (server) | `ldapsearch -x -H ldap://localhost -b "dc=lab,dc=school,dc=local"` |
| Check Samba AD service status (server) | `systemctl status samba-ad-dc` |
| Client secure channel check (client) | `nltest /sc_query:lab.school.local` |
| Client domain join test (client) | `nltest /dsgetdc:lab.school.local` |
| Force time sync (client) | `w32tm /resync /force` |
| Flush DNS (client) | `ipconfig /flushdns && ipconfig /registerdns` |
| Kerberos ticket check (client) | `klist` |
| Force GPO refresh (client) | `gpupdate /force` |
| GPO application report (client) | `gpresult /h report.html` |
| Confirm which room VLAN a PC landed on | `ipconfig` on the client — check the IP's third octet matches the expected room subnet (10.0.**10**.x = Room1, etc.) |

## Troubleshooting Matrix — Common Errors

| Symptom / Error | Likely Cause | Fix |
|---|---|---|
| "An Active Directory Domain Controller could not be contacted" | Client DNS not pointing to Zentyal, or pfSense firewall rule for port 88/389/445 missing on that room's VLAN | Set client DNS to `10.0.0.2`; check pfSense **Firewall → Rules** on the room interface for the `ZentyalDC` alias rules (Module 1.3/7.6); `ipconfig /flushdns` |
| `KRB_AP_ERR_SKEW` / clock skew errors | Client/DC time difference >5 min | `w32tm /resync /force`; verify DC is an NTP source |
| "The trust relationship between this workstation and the primary domain failed" | Machine account password out of sync (common after re-imaging or long offline period) | Rejoin domain, or on client: `Reset-ComputerMachinePassword -Server dc1.lab.school.local` |
| Domain join dialog greyed out / not available | Windows 11 **Home** edition in use | Upgrade to Pro/Education — Home cannot join AD domains |
| Users can't log in with `DOMAIN\user` after join | NetBIOS name mismatch or DNS suffix not applied | Confirm NetBIOS name in Zentyal matches login prompt; **System → About → Rename this PC (domain)** should show correct domain |
| GPOs not applying to student machines | Computer object in wrong OU, or `gpupdate` not run/rebooted | `samba-tool computer move` to correct OU; `gpupdate /force` + reboot |
| AppLocker not blocking anything despite "Enforce" mode | `Application Identity` service (`AppIDSvc`) not running | `sc config AppIDSvc start=auto && sc start AppIDSvc` |
| OneDrive KFM not redirecting folders | Tenant ID missing/incorrect in GPO, or account not signed into OneDrive | Re-verify Tenant ID string; confirm sign-in (`OneDrive.exe /background`) |
| Students still show as local admin post-join | Account manually added to local `Administrators` group at some point, or `Domain Users` mistakenly nested into local admins | Check `net localgroup administrators` on the client; remove the entry |
| Faculty can't install software despite local admin | AppLocker still blocking — local admin doesn't bypass AppLocker | Verify the `Lab-Faculty`-scoped Allow rule (Module 6.2.1); confirm `AppIDSvc` is running |
| Faculty local-admin GPO not applying | GPO linked to wrong OU (e.g., linked to `Faculty` user OU instead of the lab **computer** OUs) | Re-link `Faculty-LocalAdmin-LabPCs` to the Room1–4 computer OUs, not the user OU; `gpupdate /force` + reboot |
| A whole room can reach the internet but not the DC | pfSense firewall rule for that room's VLAN missing one of the required ports (53/88/123/389/636/445) | Compare **Firewall → Rules** for the affected room against a working room; add the missing rule |
| Web filter blocks a legitimate education site | pfBlockerNG blocklist over-matched a domain, or SquidGuard category too broad | Add the domain to a pfBlockerNG allow-list ("Whitelist" tab), or add a SquidGuard destination override for that room |
| Students bypass web filtering by using a public DNS resolver directly | DHCP pushed Zentyal's DNS but nothing blocks students from hardcoding `8.8.8.8` in their own adapter settings | On the ROOM interfaces in pfSense, add a **NAT/firewall rule redirecting all outbound port 53 traffic** (except from Zentyal itself) to the internal resolver, so manual DNS overrides get forced back through the filter |
| SYSVOL/GPO not replicating between DCs (if you add a 2nd DC later) | Samba `sysvol` replication is manual/scripted, not FRS/DFSR by default | Use `samba-tool ntacl sysvolreset` and verify `rsync`-based SYSVOL sync scripts are running |
| Slow logons across the four rooms | DNS forwarder misconfigured, causing internal lookups to leak out through pfSense unnecessarily | Ensure Zentyal's BIND has `lab.school.local` as an authoritative zone, and only forwards genuinely external names to pfSense |

---

**Deployment Order Recap:** Module 1 (hypervisor VMs: pfSense + Zentyal, VLANs, DHCP, web filtering) → Module 2 (Zentyal domain provisioning) → Module 3 (accounts/OUs, incl. `Faculty` OU + `Lab-Faculty` group) → Module 4 (join all client PCs across four rooms) → Module 5 (OneDrive/backup policy) → Module 6 (local lockdown/AppLocker, incl. faculty local-admin + AppLocker exception) → Module 7 (wrap the VM layer in Terraform and everything else in Ansible, so re-running `site.yml` is how you onboard, patch, and audit going forward) → validate with the Cheat Sheet before handing the lab over to students and faculty.
