# School Computer Laboratory Network Deployment Guide
### Zentyal Server (Samba4 AD DC) + Windows 11 Clients — 4-Room Lab Rollout

**Prepared for:** Senior Systems Architect / IT Administrator
**Scope:** Replace Windows Server with a free Linux-based Active Directory Domain Controller, provide web-based management, enforce strict student desktop restrictions, and back up user data to OneDrive/Microsoft 365.

---

## Architecture Overview

| Component | Choice | Reason |
|---|---|---|
| Domain Controller | Zentyal Server 8.x (Development Edition), Ubuntu Server 22.04/24.04 LTS base | Free, Samba4 AD-compatible, full web GUI |
| Directory Service | Samba4 AD DC (behind Zentyal's "Domain Controller" module) | Fully interoperable with Windows 11 domain-join, GPOs, Kerberos |
| DNS | Zentyal-managed BIND9 (internal), integrated with Samba AD | Required for Kerberos/AD to function |
| Clients | Windows 11 Pro or Education (Home edition **cannot** domain-join) | Required for domain join |
| GPO Management | RSAT on an admin workstation | Zentyal's GUI does not edit GPOs directly; GPOs are Windows-side objects stored in SYSVOL |
| Cloud Backup | Microsoft OneDrive (KFM + Files On-Demand) | Native to Microsoft 365 tenant, no extra backup server needed |

**Network Plan (4 rooms, single subnet recommended for simplicity):**

```
Room 1 (VLAN 10) ┐
Room 2 (VLAN 20) ├── Core Switch/Router ── Zentyal AD DC (10.0.0.2) ── Internet/M365
Room 3 (VLAN 30) │        (10.0.0.1 gateway)
Room 4 (VLAN 40) ┘
```

If you segment rooms into VLANs, make sure DHCP relay or a DHCP server on each VLAN points clients' DNS to the Zentyal server's static IP, and that inter-VLAN routing allows ports 88 (Kerberos), 389/636 (LDAP), 445 (SMB), 53 (DNS), and 123 (NTP) to reach the DC.

---

# MODULE 1 — Server Installation & Web GUI Setup

## 1.1 Download and Install Zentyal Server

1. Download the **Zentyal Server Development Edition ISO** (free, community-supported) from the official Zentyal site — it ships as a customized Ubuntu Server LTS installer.
2. Boot the target server hardware/VM from the ISO.
3. Proceed through the standard Ubuntu-based installer:
   - Choose **manual/static network configuration** later (Zentyal's installer usually sets this up post-install via the web GUI, but you can also pre-set a static IP at this stage).
   - Set hostname, e.g. `dc1`.
   - Create the initial local admin (Linux) user — this account will log into the web interface.
   - Allow the installer to complete disk partitioning and base package installation.
4. Reboot when prompted. Zentyal will boot into a **console notice** displaying the URL to reach the Web GUI, typically:
   ```
   https://<server-ip>:8443/
   ```

## 1.2 First Web GUI Login & Module Selection

1. From an admin workstation on the same network, browse to `https://<server-ip>:8443/`.
2. Accept the self-signed certificate warning (replace with a real cert later if desired).
3. Log in with the Linux admin credentials created during install.
4. On first login, Zentyal presents a **package/module selection wizard**. Select:
   - **Domain Controller and File Sharing**
   - **DNS Server**
   - (Optional) **DHCP Server**, **Firewall**, **Antivirus** if you want Zentyal to manage those too.
5. Click **Install** — Zentyal pulls the required packages (`samba`, `bind9`, `heimdal/MIT Kerberos`, etc.) automatically via apt.

## 1.3 Configure Static IP and Network Interfaces

1. In the left sidebar, go to **Network → Interfaces**.
2. Select the LAN-facing NIC (e.g., `eth0`), set method to **Static**:
   - IP address: `10.0.0.2`
   - Netmask: `255.255.255.0`
   - Gateway: `10.0.0.1` (your router/firewall)
3. Go to **Network → DNS** (or **DNS Resolver** depending on version) and add your **upstream DNS forwarders** (e.g., `8.8.8.8`, `1.1.1.1`, or your ISP's DNS) — these are used for internet resolution; internal AD queries stay on Zentyal's own BIND.
4. Click **Save Changes** (Zentyal batches changes and applies them only when you click the green "Save Changes" button top-right).

## 1.4 Configure the Domain Controller Module

1. Navigate to **Users and Computers** (or **Domain** depending on version) → this triggers the **initial AD provisioning wizard** if not already run.
2. Choose **"Provision as the first Domain Controller of a new domain"**.
3. Set:
   - **Domain/Realm name (FQDN):** `lab.school.local`
   - **NetBIOS domain name:** `LABSCHOOL` (auto-suggested, 15-char max, no dots)
   - **Server role:** Domain Controller
4. Set the **Directory Manager (DS Restore Mode) password** — store this securely; it's your AD disaster-recovery password.
5. Click **Provision**. Zentyal will:
   - Install `samba-ad-dc`
   - Configure Kerberos (`/etc/krb5.conf`)
   - Set up the internal BIND9 zone for `lab.school.local`
   - Create the default AD structure (Users, Computers, Domain Controllers OUs)
6. Confirm success under **Dashboard → Domain Controller widget**, which should show "Running" status.

## 1.5 Best Practices for Naming & DNS

- **Never** use a public TLD you don't own for your internal domain (avoid `.com`/`.net`); `.local` or a subdomain like `lab.school.local` is the Microsoft-recommended safe pattern for internal AD.
- Keep the NetBIOS name ≤15 characters, uppercase, no special characters.
- Set the Zentyal server's own `/etc/resolv.conf` (or Zentyal's DNS module) to point to **itself** (`127.0.0.1` or its static IP) as primary DNS, with your forwarders configured only as *forwarders*, not primary resolvers — this is critical, otherwise Kerberos/SRV record lookups will fail intermittently.
- Verify DNS is serving AD SRV records:
  ```bash
  dig _ldap._tcp.lab.school.local SRV
  dig _kerberos._udp.lab.school.local SRV
  ```
  Both should return the DC's hostname and port.

---

# MODULE 2 — User & Group Architecture

## 2.1 Create Organizational Units (OUs) via Web GUI

1. Go to **Users and Computers** in the left sidebar.
2. Right-click (or use the "+" action menu) on the domain root (`lab.school.local`) → **New → Organizational Unit**.
3. Create:
   - `Students` (top-level OU)
   - `Lab-Admins` (top-level OU)
4. Optionally nest sub-OUs per room for granular GPO targeting later, e.g.:
   ```
   Students
   ├── Room1
   ├── Room2
   ├── Room3
   └── Room4
   ```
   This lets you apply room-specific policies (e.g., different lab software) without duplicating logic.

## 2.2 Create Student Accounts as Standard (Non-Admin) Users

1. Right-click the `Students` OU (or the relevant room sub-OU) → **New → User**.
2. Fill in: First name, Surname, **Username** (this becomes the `sAMAccountName`), and a temporary password.
3. **Critical setting:** Leave the account as a plain **Domain User**. Do **not** add it to `Domain Admins`, `Administrators`, or any local Administrators group.
4. Check **"User must change password at next logon"** for first-time onboarding.
5. Repeat per student, or use **bulk import**: Zentyal supports **CSV/LDIF import** under **Users and Computers → Import/Export**, which is far faster for 100+ student accounts across 4 rooms.
   - Prepare a CSV with columns like `username,firstname,lastname,password,ou`.
   - Import via the GUI's file upload tool.
6. **Why this enforces restriction automatically:** In Windows 11, when a domain user who is *not* a member of the local `Administrators` group (or `Domain Admins`) logs on, Windows automatically treats them as a **Standard User** — no local admin rights, no ability to install software, change system settings, or bypass UAC prompts. This is the foundation that Module 5's AppLocker/GPO restrictions build on.

## 2.3 Create the Lab-Admins Group and Onboarding Template

1. Under **Users and Computers → Groups**, create a security group `Lab-Admin-Staff` and add it (or individual teacher/IT accounts) to the built-in `Domain Admins` group *only* if they need full domain rights — otherwise use **Delegation** (see 2.4) for least-privilege.
2. For **fast student onboarding**, create a **template user** (e.g., `_StudentTemplate`, disabled account) with:
   - Correct OU membership
   - Correct group memberships (e.g., `Students`, printer-access groups, lab-software groups)
   - Home directory / profile path pre-set
   - Login script path (if used)
3. When creating new students, use Zentyal's **"Copy user"** function (right-click the template → Copy) to clone group memberships and settings, then just fill in name/username/password — this saves significant time onboarding a full classroom.

## 2.4 Faculty OU & Elevated Software-Install Rights

Faculty need to install course-specific software on lab PCs (e.g., IDEs, CAD tools, subject plugins) without being full Domain Admins and without you having to touch every machine by hand. The cleanest model is: **Faculty are local admins on lab PCs only**, never on the domain itself.

### 2.4.1 Create the Faculty OU and Security Group

1. In **Users and Computers**, create a top-level OU: `Faculty` (sibling to `Students` and `Lab-Admins`).
2. Create faculty user accounts inside it the same way as students (Module 2.2) — they remain plain **Domain Users**, no domain-level elevation.
3. Create a **security group** `Lab-Faculty` (Users and Computers → Groups → New → Security Group).
4. Add every faculty account to `Lab-Faculty`. This single group is what you'll target in GPOs — never add faculty individually to local admin groups on each PC.

```bash
# On the Zentyal server
samba-tool group add Lab-Faculty
samba-tool group addmembers Lab-Faculty jsmith,mgarcia,rwong
```

### 2.4.2 Grant Local Admin on Lab PCs Only (Restricted Groups / Group Policy Preferences)

This is the key step: faculty get admin rights **on the lab computers**, not on the domain, not on servers, and not via a domain-wide privileged group.

1. In `gpmc.msc`, create a GPO `Faculty-LocalAdmin-LabPCs`, and link it **only to the OU containing the lab computer objects** (e.g., `Computers\Room1-PCs`, `Room2-PCs`, etc. — link to all four room-computer OUs, not to `Students` or `Faculty` user OUs, since this is a *computer-side* policy).
2. Edit the GPO → **Computer Configuration → Preferences → Control Panel Settings → Local Users and Groups**.
3. Right-click → **New → Local Group**:
   - Group name: **Administrators (built-in)**
   - Action: **Update**
   - Under **Members**, click **Add** → enter `LABSCHOOL\Lab-Faculty` → check **Add to this group**.
4. This uses Group Policy Preferences item-level targeting, so `Lab-Faculty` is merged into the local `Administrators` group on every lab PC the GPO applies to, on next `gpupdate`/reboot — students are unaffected, and faculty gain zero rights on the DC, other servers, or non-lab machines.
5. Verify on a test client:
   ```cmd
   net localgroup administrators
   ```
   should list `LABSCHOOL\Lab-Faculty` alongside the built-in `Administrator`.

**Why not use `Restricted Groups` (the older GPO node) instead?** Restricted Groups *overwrites* the entire local Administrators membership every refresh, which can silently strip out the built-in local `Administrator` account or other needed entries. Group Policy Preferences with **"Update"** action is additive and safer for this use case — use Restricted Groups only if you deliberately want to enforce an exact, fixed membership list.

### 2.4.3 Scope AppLocker So It Doesn't Block Faculty (see Module 5.2)

Local admin rights alone don't bypass AppLocker — AppLocker rules apply regardless of local admin status unless you explicitly scope a rule to the `Lab-Faculty` group. This is handled in **Module 5.2**, where a permissive AppLocker rule is added specifically for this security group.

### 2.4.4 Faculty Software-Install Workflow (Day-to-Day)

With this setup, a teacher can, at the start of term:
1. Log into any lab PC in their room with their normal faculty domain account.
2. Right-click an installer → **Run as administrator** (UAC will prompt but succeed, since `Lab-Faculty` is a local admin) — no separate elevated credential needed.
3. Install the course software once per room (or push it via Module 5.3's GPO Software Installation / Chocolatey method if you want it centrally deployed to all rooms instead of teacher-by-teacher).

For anything touching **multiple rooms** or that should be standardized, prefer the admin-pushed methods in Module 5.3 over relying on individual faculty installs — it keeps configuration consistent and auditable. Faculty local-admin is best reserved for one-off, room-specific, or last-minute course tools.

## 2.5 Delegate Limited Admin Rights (Optional, Recommended)

Instead of making lab teachers full `Domain Admins`, delegate only what they need (e.g., password resets for the `Students` OU) via `samba-tool`:
```bash
samba-tool ou list
samba-tool delegation add-service-account <lab-admin-user> "Reset Password" --ou="OU=Students,DC=lab,DC=school,DC=local"
```
This limits blast-radius if a lab-admin account is compromised.

---

# MODULE 3 — Windows 11 Client Provisioning

## 3.1 Network Adapter / DNS Configuration

1. On each Windows 11 machine: **Settings → Network & Internet → Ethernet/Wi-Fi → Edit DNS settings** (or via legacy Control Panel → Network Adapter → Properties → IPv4).
2. Set:
   - **Preferred DNS:** `10.0.0.2` (Zentyal server IP) — this is mandatory; Windows cannot locate the domain via SRV records otherwise.
   - **Alternate DNS:** leave blank or point to a secondary DC if you build one later. Do **not** put your ISP/public DNS here, or domain lookups will silently fail.
3. Confirm the machine can resolve the domain:
   ```cmd
   nslookup lab.school.local
   ping dc1.lab.school.local
   ```

## 3.2 Join the Domain

1. **Settings → Accounts → Access work or school → Connect** → choose **"Join this device to a local Active Directory domain"** (this option appears only on Windows 11 **Pro/Education**, not Home).
2. Enter the domain name: `lab.school.local`.
3. When prompted, authenticate with a domain account that has rights to join computers (a Lab-Admin or Domain Admin account — **not** the student account).
4. Choose which OU the computer object lands in (or accept default `Computers` OU, then move it later via the Zentyal GUI or `samba-tool computer move`).
5. Restart when prompted.
6. At the new logon screen, click **"Other user"** and log in as `LABSCHOOL\studentusername`.

## 3.3 Verification & Troubleshooting Commands (Client-Side)

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

**Important:** Kerberos requires client and DC clocks to be within **5 minutes** of each other by default. If joins fail with time-sync errors, fix `w32tm` **before** retrying the domain join, not after.

---

# MODULE 4 — OneDrive Cloud Backup & Storage Configuration

## 4.1 Install RSAT to Manage GPOs

Since Zentyal's web GUI manages Samba/AD objects (users, groups, DNS) but **does not** provide a native GPO editor, you manage Group Policy from a Windows machine using RSAT, exactly as you would with a traditional Windows Server AD DC (Samba4 stores GPOs in SYSVOL in Windows-compatible format).

1. On an admin Windows 11 Pro/Education workstation (already domain-joined): **Settings → Optional Features → Add a feature**.
2. Search for and install:
   - **RSAT: Group Policy Management Tools**
   - **RSAT: Active Directory Domain Services and Lightweight Directory Tools**
3. Launch **Group Policy Management** (`gpmc.msc`) — it should connect to `lab.school.local` automatically since the machine is domain-joined.

## 4.2 Add OneDrive Administrative Templates to the Central Store

1. Download the latest **OneDrive Administrative Templates** package from Microsoft (contains `OneDrive.admx` and `OneDrive.adml`).
2. On the domain, create the **Central Store** if it doesn't exist yet (this is just a SYSVOL folder structure Samba4 already replicates):
   ```
   \\lab.school.local\SYSVOL\lab.school.local\Policies\PolicyDefinitions\
   \\lab.school.local\SYSVOL\lab.school.local\Policies\PolicyDefinitions\en-US\
   ```
3. Copy:
   - `OneDrive.admx` → `...\PolicyDefinitions\`
   - `OneDrive.adml` → `...\PolicyDefinitions\en-US\`
4. Reopen **Group Policy Management Editor** — under **Computer Configuration → Administrative Templates**, you should now see a **OneDrive** folder.

## 4.3 GPO: Silent Account Configuration

1. In `gpmc.msc`, right-click the `Students` OU → **Create a GPO in this domain, and Link it here**. Name it `OneDrive-SilentConfig`.
2. Edit the GPO → **Computer Configuration → Policies → Administrative Templates → OneDrive**:
   - Enable **"Silently move Windows known folders to OneDrive"** *(covered fully in 4.4)*
   - Enable **"Silently sign in users to the OneDrive sync app with their Windows credentials"**
   - Enable **"Set the default location for the OneDrive folder"** if desired.
3. Enable and configure **"Allow silent account configuration"**, entering your school's **Microsoft 365 Tenant ID** (found in the Microsoft 365 Admin Center → Settings → Org Settings → Organization Profile, or via `Get-MgOrganization` in Microsoft Graph PowerShell).
4. This forces OneDrive to auto-configure using the signed-in Windows/Azure AD credential — **no manual OneDrive setup wizard** appears for students. (Note: since accounts here are on-prem AD, not Azure AD-joined, students will need Microsoft 365 login prompts unless you've deployed Azure AD Connect/Entra Connect to sync identities — see note below.)

> **Hybrid-Identity Note:** Pure Samba4 AD accounts are not automatically known to Microsoft 365/Entra ID. For OneDrive Silent Config and SSO to work seamlessly, you need either (a) matching UPNs and a **directory sync** solution (e.g., Azure AD Connect requires Windows Server — as a free alternative, Microsoft 365 accounts can instead be provisioned/matched manually or via **Entra Connect Sync** running from a lightweight Windows VM), or (b) students entering their school M365 email/password once at first OneDrive login. Plan identity-matching (same username = same email prefix) during Module 2 account creation to simplify this.

## 4.4 GPO: Known Folder Move (KFM) + Files On-Demand

1. In the same or a new GPO linked to `Students`, go to **Computer Configuration → Administrative Templates → OneDrive**:
   - Enable **"Silently move Windows known folders to OneDrive"** → enter your **Tenant ID** in the provided field.
   - Enable **"Prevent users from redirecting their Windows known folders to their PC"** (prevents students reversing the redirect).
2. Enable **Files On-Demand**:
   - Policy: **"Files On-Demand"** → set to **Enabled** (this keeps files as online-only placeholders until opened, saving local disk space — ideal for shared lab PCs with small SSDs).
3. Run `gpupdate /force` on a test client, sign in, and confirm:
   - `Desktop`, `Documents`, and `Pictures` folders now show the OneDrive cloud icon and redirect to `%USERPROFILE%\OneDrive - School\...`.
   - Files show cloud/check icons (Files On-Demand working).
4. **Recommended companion policy:** Enable **"Block monitoring known folders to detect files that should be moved to a OneDrive account"** = *Disabled* (i.e., leave monitoring on) so KFM notifications work correctly, and set **"Use OneDrive Files On-Demand"** at the Computer Configuration level so it applies before any user profile loads (avoids race conditions on shared lab machines).

---

# MODULE 5 — Desktop Lockdown & Application Restrictions

## 5.1 Block Execution from Flash Drives / Downloads

1. Create a GPO `Student-ExecutionLockdown`, linked to `Students` OU.
2. Under **Computer Configuration → Policies → Windows Settings → Security Settings → Software Restriction Policies** (legacy option) **or**, preferably, use **AppLocker** (see 5.2) since Software Restriction Policies are largely superseded.
3. If you additionally want to hard-block **removable media execution** regardless of AppLocker:
   - **Computer Configuration → Administrative Templates → System → Removable Storage Access** → Enable **"All Removable Storage classes: Deny all access"**, or more surgically, **"Removable Disks: Deny execute access"**.
4. This, combined with AppLocker, ensures a student cannot run a portable `.exe` from a USB stick or a browser-downloaded file even if they copy it locally.

## 5.2 AppLocker Path-Rule Whitelist

1. In the same or a new GPO, go to **Computer Configuration → Policies → Windows Settings → Security Settings → Application Control Policies → AppLocker**.
2. Right-click **Executable Rules** → **Create Default Rules** first (creates baseline allow rules for `Program Files` and `Windows`), then review/tighten them to exactly:
   - **Allow:** Path = `%PROGRAMFILES%\*` (covers `C:\Program Files\` and `C:\Program Files (x86)\`)
   - **Allow:** Path = `%WINDIR%\*` (covers `C:\Windows\`)
   - **Delete** or **do not include** any default rule that whitelists `%OSDRIVE%\*` broadly or "all files" for `BUILTIN\Users` — this is the rule that would defeat your lockdown.
3. Also configure **Windows Installer Rules**, **Script Rules**, and **Packaged app Rules** similarly (limit to Program Files/Windows-signed sources) so `.msi`, `.ps1`, `.bat`, and UWP sideloading are equally restricted.
4. **Enable the AppLocker service** — under the same GPO node, right-click **AppLocker** → **Properties**, and check **"Configured"** = Enabled for each rule type.
5. Ensure the **Application Identity service** is set to **Automatic start** (required for AppLocker to actually enforce):
   - **Computer Configuration → Preferences → Control Panel Settings → Services**, add `AppIDSvc` set to Automatic, or push via a startup script: `sc config AppIDSvc start=auto`.
6. Set enforcement mode to **"Enforce rules"** (not "Audit only") once testing confirms the whitelist doesn't block legitimate lab software.
7. Test thoroughly on one machine per room before wide rollout — misconfigured AppLocker can lock out legitimate education software installed outside `Program Files` (some portable STEM/coding tools do this; either reinstall them properly into `Program Files` or add a narrowly-scoped publisher/hash rule).

### 5.2.1 Faculty Exception (allow installs without opening the whitelist to students)

Local admin rights (Module 2.4) let faculty run an installer's UAC prompt, but AppLocker still evaluates every executable regardless of who's running it — so without an explicit rule, faculty would be blocked too. Scope a dedicated rule to the `Lab-Faculty` group only:

1. In the same GPO (or a separate GPO linked only to the lab computer OUs — AppLocker is computer-scoped, so the *user* condition inside the rule is what limits it to faculty), right-click **Executable Rules** → **Create New Rule**.
2. **Permissions:** Action = **Allow**, User or group = browse to `LABSCHOOL\Lab-Faculty` (not "Everyone").
3. **Conditions:** choose **Path** rule → `%OSDRIVE%\*` (i.e., allow execution from anywhere on the drive) — or, more conservatively, scope it to a dedicated `C:\FacultyInstalls\` staging folder if you want installers copied there first rather than run from Downloads.
4. Repeat as needed for **Windows Installer Rules** (so faculty can run `.msi` packages) and **Script Rules** if faculty need to run setup scripts.
5. Because AppLocker evaluates rules by matching user/group SID, students (not in `Lab-Faculty`) still only match the restrictive `Program Files`/`Windows` allow rules from 5.2 — this new rule simply adds a second, wider allow path that only resolves for faculty logons.
6. Confirm on a test lab PC: log in as a student → confirm a random `.exe` in Downloads still fails to launch (AppLocker block dialog appears). Log in as faculty → confirm the same file runs.

## 5.3 Pushing Software Updates While Staying Locked Down

Because students can't install anything, all software deployment must be **admin-pushed**. Options (all free):

**Option A — GPO Software Installation (native, simplest):**
1. Place `.msi` installers in a network share (e.g., `\\dc1\Software$`) that Domain Computers have read access to.
2. In a GPO linked to your **Computers** OU (not Students — this targets machines, not users), go to **Computer Configuration → Policies → Software Settings → Software Installation** → **New → Package**, point to the `.msi` on the UNC path, choose **Assigned**.
3. Software installs automatically at next machine boot — no student interaction, no admin rights needed on the student's session.

**Option B — Free Deployment Tooling (for non-MSI / bulk app management):**
- **Chocolatey** (free, open-source Windows package manager): push updates via a **scheduled task GPO** or **PowerShell script GPO** running as `SYSTEM`:
  ```powershell
  Set-ExecutionPolicy Bypass -Scope Process -Force
  iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
  choco upgrade all -y
  ```
  Deploy this as a **Computer Startup Script** (GPO: Computer Config → Windows Settings → Scripts → Startup) so it runs with SYSTEM privileges before student logon, bypassing the AppLocker/standard-user restriction entirely (SYSTEM-run scripts aren't subject to the same lockdown as the student session).
- **PDQ Deploy (Free tier)** — GUI-based, good for pushing `.exe`/`.msi` on a schedule across all 4 rooms from one console.
- **WSUS (optional, free, runs on Windows or via `wsus-offline` on Linux front-end)** for Windows Update management specifically, keeping OS patches consistent across the lab without giving students Windows Update access.

---

# MODULE 6 — Infrastructure as Code (Ansible)

**Context for this environment:** bare metal (no hypervisor/cloud API), so Terraform has little to do here — there's no VM resource for it to declare. Ansible carries almost the whole stack instead: OS-level server config, Samba/AD object management, and Windows 11 client configuration over WinRM. Terraform still has a *narrow, optional* role if you manage the Microsoft 365/Entra ID side (tenant settings, app registrations) via its `azuread` provider — not covered here since it's outside the Samba/AD scope, but worth knowing it exists if M365 config sprawls.

## 6.0 Honest Caveats Before You Commit to This

- **Zentyal's GUI is not the source of truth once you do this.** Zentyal stores its config in a Redis-backed model behind the web GUI; there isn't a clean, stable API/CLI for "config as code" against Zentyal itself. The practical approach is to treat **Samba4 (`samba-tool`) as the real source of truth** and use Zentyal purely as an optional read/monitoring dashboard on top. Provisioning the domain and creating OUs/users/groups via `samba-tool` (as this guide already does) is fully scriptable; changes made *through* the Zentyal GUI afterward won't be reflected in your Ansible state unless someone remembers to update the playbooks too — pick one source of truth and stick to it, or you'll get drift.
- **`samba-tool` commands are not natively idempotent** (e.g., `samba-tool user create` fails loudly if the user exists). Every task below wraps calls with existence checks so re-running the playbook is safe.
- **GPOs are semi-opaque as code.** Group Policy objects are stored in SYSVOL partly as binary `registry.pol` and partly as XML (for Preferences/AppLocker). Two viable IaC strategies:
  1. **Backup/Restore roundtrip** — build the GPO once by hand in `gpmc.msc`, `Backup-GPO` it to a folder, commit that folder to git, and `Import-GPO` it onto fresh domains/environments. Simple, but diffs in git are unreadable (binary blobs).
  2. **Granular registry-value management** — for the specific OneDrive/AppLocker settings this guide uses, push the exact registry values Ansible can read/diff. This is the more "real IaC" option and is what's shown below for OneDrive. AppLocker specifically is better handled via its dedicated XML export/import cmdlets (`Export-AppLockerPolicy` / `Set-AppLockerPolicy`), also shown below.
- **WinRM chicken-and-egg problem.** Ansible needs WinRM enabled to manage a Windows 11 client, but a freshly domain-joined PC doesn't have WinRM enabled by default. You need *one* bootstrap mechanism before Ansible can take over — either a GPO that enables WinRM listener + firewall rule domain-wide (one-time, done via `gpmc.msc`/RSAT, not Ansible), or Microsoft's `ConfigureRemotingForAnsible.ps1` script run once per machine during imaging.

## 6.1 Repository Layout

```
lab-iac/
├── ansible.cfg
├── inventory/
│   ├── hosts.yml                # dc1, plus all lab PCs grouped by room
│   └── group_vars/
│       ├── all.yml              # domain name, tenant ID, etc.
│       └── windows_clients.yml  # winrm connection vars
├── data/
│   ├── students.yml              # source of truth for student accounts
│   ├── faculty.yml
│   └── computers.yml             # hostname -> room/OU mapping
├── roles/
│   ├── zentyal_samba_dc/
│   ├── ad_objects/
│   ├── windows_client_join/
│   ├── onedrive_policy/
│   ├── applocker_policy/
│   └── software_push/
├── gpo-backups/                  # only if using the backup/restore GPO strategy
│   └── OneDrive-SilentConfig/    # raw Backup-GPO output, committed as-is
└── site.yml
```

## 6.2 Control Node Prerequisites

On the Linux machine that will *run* Ansible (can be the Zentyal server itself, or a separate admin box):

```bash
pip install --user ansible pywinrm requests-ntlm requests-kerberos
ansible-galaxy collection install ansible.windows community.windows microsoft.ad
```

The `microsoft.ad` collection (community-maintained) provides modules like `microsoft.ad.user`, `microsoft.ad.ou`, `microsoft.ad.group` that speak native AD/LDAP against **any** AD-compatible DC, including Samba4 — this is generally a better fit than hand-rolling `samba-tool` shell calls where the collection covers your use case; fall back to raw `samba-tool` for anything it doesn't support (e.g., initial domain provisioning).

## 6.3 Data-Driven Source of Truth

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

## 6.4 Role: `zentyal_samba_dc` (Server Bootstrap)

Run once against `dc1` (Ubuntu already installed manually on bare metal — Ansible picks up from a base OS):

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

- name: Template static network config
  template:
    src: 01-netcfg.yaml.j2
    dest: /etc/netplan/01-netcfg.yaml
    mode: "0600"
  notify: apply netplan

- name: Ensure DNS forwarders are set
  lineinfile:
    path: /etc/samba/smb.conf
    regexp: '^\s*dns forwarder'
    line: "    dns forwarder = 8.8.8.8 1.1.1.1"
  notify: restart samba-ad-dc
```

`ad_realm`, `ad_netbios`, `ad_admin_password` live in `inventory/group_vars/all.yml` (vault-encrypt the password: `ansible-vault encrypt_string`).

## 6.5 Role: `ad_objects` (OUs, Groups, Users from Data Files)

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

Repeat the same pattern for `faculty.yml` targeting `Lab-Faculty`. Because every `command` task checks stderr for "already exists" before failing, re-running `ansible-playbook site.yml` after adding three new rows to `students.yml` only creates those three — safe to run every time you onboard students mid-term.

## 6.6 Role: `windows_client_join` (WinRM-Managed Clients)

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

## 6.7 OneDrive Policy as Granular Registry State (True IaC, No Binary GPO Blob)

Rather than importing an opaque GPO backup, push the exact registry values the Administrative Templates would otherwise set, directly and git-diffably:

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

This achieves the same end state as Module 4's GPO but as plain Ansible tasks — `git blame` on this file tells you exactly who changed the tenant ID and when, which a GPO backup blob won't.

## 6.8 AppLocker as Exported XML Policy

AppLocker isn't a simple registry key — use its native export/import cmdlets, with the XML committed to git:

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

`files/applocker-policy.xml` encodes exactly the rules from Module 5.2/5.2.1 (Program Files/Windows allow for everyone, wider allow scoped to the `Lab-Faculty` SID). Regenerate it whenever rules change via `Export-AppLockerPolicy -Xml` from a reference machine, then commit the diff for review before rolling out.

## 6.9 Role: `software_push` (Faculty/Admin-Requested Course Software)

```yaml
# roles/software_push/tasks/main.yml
- name: Install course software via Chocolatey
  chocolatey.chocolatey.win_chocolatey:
    name: "{{ item }}"
    state: present
  loop: "{{ room_software_list }}"
```

`room_software_list` is defined per room in `inventory/group_vars/`, so Room 3 (CAD lab) and Room 1 (intro programming) can carry different package lists while sharing every other role.

## 6.10 Running It & Keeping Drift in Check

```bash
ansible-playbook -i inventory/hosts.yml site.yml --vault-password-file .vault-pass
```

- Run the full playbook after **any** change to `data/*.yml` — new student, new faculty member, room reassignment.
- Consider a nightly `cron` run in **check mode** (`--check --diff`) that emails you a report of drift (e.g., someone manually added a local admin through the GUI) without applying changes automatically — a lightweight, free substitute for a full compliance/CI product.
- If this grows beyond what cron + a shared terminal can manage, free self-hosted options for scheduled/triggered runs include **Semaphore UI** or **AWX** (upstream of Red Hat Ansible Automation Platform) — both installable on the same bare-metal box or a small companion server.

---

# ADMINISTRATOR'S CHEAT SHEET

## Samba/Zentyal `samba-tool` Essentials (run on the Zentyal server, as root/sudo)

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

## Troubleshooting Matrix — Common Domain-Join / AD Errors

| Symptom / Error | Likely Cause | Fix |
|---|---|---|
| "An Active Directory Domain Controller could not be contacted" | Client DNS not pointing to Zentyal server | Set client DNS to `10.0.0.2`; run `ipconfig /flushdns` |
| `KRB_AP_ERR_SKEW` / clock skew errors | Client/DC time difference >5 min | `w32tm /resync /force`; verify DC is an NTP source (`w32tm /config /manualpeerlist:"dc1..." /syncfromflags:manual /update`) |
| "The trust relationship between this workstation and the primary domain failed" | Machine account password out of sync (common after re-imaging or long offline period) | Rejoin domain, or on client: `Reset-ComputerMachinePassword -Server dc1.lab.school.local` |
| Domain join dialog greyed out / not available | Windows 11 **Home** edition in use | Upgrade to Pro/Education — Home cannot join AD domains |
| Users can't log in with `DOMAIN\user` after join | NetBIOS name mismatch or DNS suffix not applied | Confirm NetBIOS name in Zentyal matches login prompt; check **System → About → Rename this PC (domain)** shows correct domain |
| GPOs not applying to student machines | Computer object in wrong OU, or `gpupdate` not run/rebooted | `samba-tool computer move` to correct OU; run `gpupdate /force` + reboot |
| AppLocker not blocking anything despite "Enforce" mode | `Application Identity` service (`AppIDSvc`) not running | `sc config AppIDSvc start=auto && sc start AppIDSvc` |
| OneDrive KFM not redirecting folders | Tenant ID missing/incorrect in GPO, or account not signed into OneDrive | Re-verify Tenant ID string; confirm user signed in (`OneDrive.exe /background`) |
| Students still show as local admin post-join | Account manually added to local `Administrators` group at some point, or built-in `Domain Users` mistakenly nested into local admins | Check `net localgroup administrators` on the client; remove the entry |
| Faculty can't install software despite local admin | AppLocker still blocking — local admin doesn't bypass AppLocker | Add/verify the `Lab-Faculty`-scoped Allow rule (Module 5.2.1); confirm `AppIDSvc` is running |
| Faculty local-admin GPO not applying | GPO linked to wrong OU (e.g., linked to `Faculty` user OU instead of the lab **computer** OUs) | Re-link `Faculty-LocalAdmin-LabPCs` GPO to the Room1–4 computer OUs, not the user OU; `gpupdate /force` + reboot |
| SYSVOL/GPO not replicating between DCs (if you add a 2nd DC later) | Samba `sysvol` replication is manual/scripted, not FRS/DFSR by default | Use `samba-tool ntacl sysvolreset` and verify `rsync`-based SYSVOL sync scripts are running |
| Slow logons across the 4 rooms | DNS forwarder misconfigured, causing internal lookups to leak externally | Ensure Zentyal DNS module has `lab.school.local` as an authoritative zone, forwarders only for external names |

---

**Deployment Order Recap:** Module 1 (server) → Module 2 (accounts/OUs, incl. `Faculty` OU + `Lab-Faculty` group) → Module 3 (join all client PCs across 4 rooms) → Module 4 (OneDrive/backup policy) → Module 5 (lockdown/AppLocker, incl. faculty local-admin + AppLocker exception) → Module 6 (wrap it all in Ansible so re-running `site.yml` is how you onboard, patch, and audit going forward) → validate with the Cheat Sheet before handing the lab over to students and faculty.
