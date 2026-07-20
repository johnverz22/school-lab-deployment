# School Computer Laboratory Network Deployment Guide
### Zentyal Server (Samba4 AD DC) + Windows 11 Clients + Google Workspace Integration — 4-Room Lab Rollout

**Prepared for:** A developer/programmer moving into a systems administration role
**Scope:** Stand up a free, Linux-based Active Directory domain on dedicated physical hardware, create personalized student/faculty accounts, replace local persistent profiles with transient/auto-wiped ones so students are pushed toward cloud storage, sync that storage to **Google Drive** (not OneDrive), and lock the desktop down with a full set of lab-appropriate Group Policies — all within a lab network segment you administer, with no dependency on the larger CCSE network you don't have access to.

> **How to read this guide if you're coming from software, not sysadmin work:** A **Domain Controller** is basically a centralized auth server (like an identity provider/OAuth server, but for Windows logins). A **GPO (Group Policy Object)** is Windows' built-in configuration-management system — think Ansible/Puppet, except it's pushed automatically to any machine that's a member of the right "group" (OU) every time it boots or roughly every 90 minutes. An **OU (Organizational Unit)** is a namespace/folder you place accounts into so policy can target "everything in here." **Kerberos** is the token-based auth protocol AD runs on — same ideas as OAuth2 tokens with expiry, just older and not HTTP-native. Keep these mappings in mind; they make the GUI screens much less mysterious. Boxes like this one call out concepts worth pausing on throughout the guide.

---

## Architecture Overview

| Component | Choice | Reason |
|---|---|---|
| Domain Controller | **Zentyal Server, Community/Development Edition only** (Ubuntu Server 22.04/24.04 LTS base), installed directly on a dedicated physical box | Free forever, no subscription required, Samba4 AD-compatible, has a web GUI for the AD/DNS/DHCP pieces you need |
| Directory Service | Samba4 AD DC (behind Zentyal's "Domain Controller" module) | Fully interoperable with Windows 11 domain-join, GPOs, Kerberos |
| Internal DNS | Zentyal-managed BIND9, integrated with Samba AD | Required for Kerberos/AD to function — non-negotiable |
| DHCP | Zentyal's own DHCP module (since there's no separate router/firewall device in this design) | One box, one job list — no pfSense in this build, so Zentyal picks up DHCP alongside AD/DNS |
| Clients | Windows 11 **Pro** or **Education** (Home edition **cannot** domain-join) | Domain join requires Pro/Education |
| GPO Management | RSAT on an admin workstation | Zentyal's web GUI edits Samba/AD *objects* (users, groups, DNS, DHCP) but does **not** edit GPOs — those are Windows-native objects stored in SYSVOL |
| Cloud Storage | **Google Drive for Desktop**, with **Google Credential Provider for Windows (GCPW)** as an additional Windows sign-in option | Your call — replaces OneDrive entirely; keeps student data out of local, soon-to-be-wiped profiles |
| Local Profile Policy | **Transient/auto-wiped local profiles** (deleted on restart if unused for 1–3 days) | Nothing meaningful should live only on a shared lab PC — this is what makes Drive sync mandatory rather than optional |
| Network Filtering | **None at the network layer in this build** | You dropped pfSense, so there's no lab-level web filter or inter-room firewall. Content/behavior restriction now lives entirely in local Windows Group Policy (Module 5) — see the callout in Module 0 below. |

> **What changed from a pfSense-based design, and why it's fine here:** Without pfSense, you lose network-layer web filtering, an owned inter-room firewall, and lab-specific VLANs. Given your actual goals — a working domain, personalized accounts, controlled desktops, no local software installs, and CCSE-only network access — none of that network-layer tooling is required to hit those goals; it was solving a different problem (content filtering, room segmentation) that isn't the priority right now. The tradeoff to be honest about: without pfSense, you have **no network-level enforcement of what a student's traffic reaches** — everything protective in this build lives in Windows Group Policy, which is enforceable but not identical to a proper firewall/filter (a sufficiently technical student who somehow got code execution could route around device-level rules in a way they can't route around a network-level block). Module 5's lockdown (blocking script/interpreter access, cmd, PowerShell, Registry Editor) exists specifically to close that gap as much as is practical on the endpoint alone.

### Network Plan

Single flat network, one box, no VLANs, no router of your own:

```
Existing CCSE-managed uplink/gateway (you don't administer this device)
        │
        ▼
┌─────────────────────────────┐
│  Lab switch (flat L2)        │
│  10.0.0.0/24                 │
└───────────┬──────────┬───────┘
            │          │
   ┌────────┴───┐  ┌───┴─────────────────────────┐
   │ Room 1-4    │  │ Zentyal box (bare metal)     │
   │ lab PCs     │  │ 10.0.0.2 (static)             │
   │ DHCP-leased │  │ Samba4 + BIND9 + DHCP + GUI   │
   │ from Zentyal│  │ :8443                          │
   └─────────────┘  └───────────────────────────────┘
```

- All four rooms share one broadcast domain/subnet, `10.0.0.0/24`. There's no per-room VLAN in this design since there's no router of yours to enforce it — if you later want room-level network separation, that would require reintroducing a routing device (pfSense or otherwise); it's out of scope here.
- **Zentyal (`10.0.0.2`)** is the DHCP server, DNS server, and domain controller, all in one box. DHCP scope: something like `10.0.0.50`–`10.0.0.250`, gateway pushed to clients = whatever the existing uplink device's LAN-facing address is (get this one value from whoever manages the CCSE network — it's the only thing you need from them).
- **Wi-Fi is handled separately, at the endpoint, not the network** — see Module 5.5. Since you don't own a network device to enforce "CCSE SSID only," this has to be a Windows-side Group Policy restriction instead.

---

# MODULE 1 — Zentyal Server Installation & Web GUI Setup

## 1.1 Download and Install Zentyal Server

1. Download the **Zentyal Server Development/Community Edition ISO** (free) from the official Zentyal site — confirm you're on the free download page, not a "buy subscription" page. It ships as a customized Ubuntu Server LTS installer.

> **A note on editions, read before downloading:** Zentyal's free tier has shown up under a couple of names ("Development Edition," "Community Edition") depending on when you look. The paid **Commercial/Server Edition with a subscription** adds a vendor security-patch feed, remote monitoring, and a content-filter/UTM module — none of which this guide assumes. Use the free edition only. The tradeoff: you patch the underlying Ubuntu packages yourself via plain `apt`, on your own schedule.

2. Write the ISO to a USB stick and boot the Zentyal machine from it.
3. Proceed through the standard Ubuntu-based installer:
   - Networking can stay on DHCP for the install itself (if something upstream is already handing out addresses on this segment); you'll set the permanent static IP from the web GUI in step 1.3 below. Otherwise, set `10.0.0.2/24` manually at this stage.
   - Set hostname, e.g. `dc1`.
   - Create the initial local admin (Linux) user — this account logs into the web interface.
   - Let the installer finish disk partitioning and base package installation.
4. Reboot when prompted. Zentyal boots into a **console notice** showing the URL for the web GUI:
   ```
   https://<server-ip>:8443/
   ```

## 1.2 First Web GUI Login & Module Selection

1. From an admin workstation on the same network, browse to `https://<server-ip>:8443/`.
2. Accept the self-signed certificate warning (a real cert isn't required for AD to function).
3. Log in with the Linux admin credentials from the install.
4. On first login, Zentyal presents a **package/module selection wizard**. Select:
   - **Domain Controller and File Sharing**
   - **DNS Server**
   - **DHCP Server** — check this one. Without pfSense in this build, Zentyal is the only thing that can hand out addresses to the four rooms.
5. Click **Install** — Zentyal pulls `samba`, `bind9`, `heimdal`/MIT Kerberos, `isc-dhcp-server`, etc. automatically via apt.

## 1.3 Configure Static IP and Network Interface

1. Left sidebar → **Network → Interfaces**. Select the NIC, set method to **Static**:
   - IP address: `10.0.0.2`
   - Netmask: `255.255.255.0`
   - Gateway: the existing uplink's address on this segment (the one value you need from whoever manages the CCSE network)
2. **Network → DNS** (or **DNS Resolver**): add upstream forwarders for anything Zentyal's own BIND can't answer internally (`8.8.8.8`/`1.1.1.1`, or whatever the CCSE network prefers). Internal AD queries never leave Zentyal's own BIND regardless of this setting.
3. Click **Save Changes** (top-right green button) — Zentyal batches config changes and only applies them on this click.

## 1.4 Configure DHCP

1. **Network → DHCP** (module name varies slightly by Zentyal version).
2. Create a scope for `10.0.0.0/24`:
   - Range: `10.0.0.50` – `10.0.0.250` (leaves room below for static reservations if you want predictable IPs for specific lab PCs)
   - **Gateway:** the CCSE uplink address
   - **DNS servers pushed to clients:** `10.0.0.2` (Zentyal itself) — mandatory, this is what lets Kerberos/SRV lookups work; never push a public DNS server here.
3. Save Changes.

## 1.5 Configure the Domain Controller Module

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

## 1.6 Best Practices for Naming & DNS

- **Never** use a public TLD you don't own for your internal domain (avoid `.com`/`.net`); `.local` or a subdomain like `lab.school.local` is Microsoft's recommended safe pattern for internal AD.
- Keep the NetBIOS name ≤15 characters, uppercase, no special characters.
- Set Zentyal's own DNS resolution to point at **itself** (`127.0.0.1` or its static IP) as primary, with your forwarders configured only as *forwarders* for names outside the domain — this is critical, otherwise Kerberos/SRV record lookups fail intermittently.
- Verify DNS is serving AD SRV records:
  ```bash
  dig _ldap._tcp.lab.school.local SRV
  dig _kerberos._udp.lab.school.local SRV
  ```
  Both should return the DC's hostname and port. If they don't, stop here and fix DNS before touching anything else.

---

# MODULE 2 — User & Group Architecture

## 2.1 Create Organizational Units (OUs) via Web GUI

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

## 2.2 Create Student Accounts as Standard (Non-Admin) Users

1. Right-click the `Students` OU (or the relevant room sub-OU) → **New → User**.
2. Fill in: first name, surname, **username** (becomes the `sAMAccountName`), and a temporary password.
3. **Critical setting:** leave the account as a plain **Domain User**. Do **not** add it to `Domain Admins`, `Administrators`, or any local Administrators group.
4. Check **"User must change password at next logon"** for first-time onboarding.
5. Repeat per student, or use **bulk import**: Zentyal supports **CSV/LDIF import** under **Users and Computers → Import/Export** — much faster for 100+ students across four rooms.
6. **Why this enforces restriction automatically:** on Windows 11, a domain user who is *not* a member of the local `Administrators` group (or `Domain Admins`) is automatically a **Standard User** — no local admin rights, no installing software, no changing system settings, no bypassing UAC prompts. This is the foundation Module 5's lockdown builds on top of.

## 2.3 Restrict Student Logon Hours (Optional, Recommended)

Since this is a shared lab, you can restrict when student accounts are even allowed to authenticate — e.g., only during school hours, so a stolen or reused credential can't be used to log in at 2 a.m. from off-site.

From an RSAT-equipped admin workstation:

```powershell
# Restrict a single student account to Mon-Fri, 7am-5pm
Set-ADUser jdoe -LogonHours (New-Object byte[] 21) # placeholder array; use a GUI tool for the actual bit-mask
```

In practice, it's far less error-prone to set this visually: open **Active Directory Users and Computers** (part of RSAT) → right-click the student → **Properties → Account → Logon Hours...** and paint the allowed hours on the grid. Apply this via the `_StudentTemplate` account (Module 2.4) so every cloned student inherits the same schedule.

## 2.4 Create the Lab-Admins Group and an Onboarding Template

1. Under **Users and Computers → Groups**, create a security group `Lab-Admin-Staff`. Add it (or individual IT staff accounts) to the built-in `Domain Admins` group *only* if they need full domain rights — otherwise use **Delegation** (2.6) for least-privilege.
2. For fast student onboarding, create a **template user** (e.g., `_StudentTemplate`, disabled) with correct OU membership, group memberships, and logon-hours restriction pre-set.
3. When creating new students, use Zentyal's **"Copy user"** function (right-click the template → Copy) to clone settings — fill in name/username/password and you're done.

## 2.5 Faculty OU & Elevated Software-Install Rights

Faculty need to install course-specific software on lab PCs without being full Domain Admins. The model: **faculty are local admins on lab PCs only, never on the domain itself.**

### 2.5.1 Create the Faculty OU and Security Group

1. Create a top-level OU `Faculty` (sibling to `Students` and `Lab-Admins`).
2. Create faculty accounts inside it the same way as students (2.2) — plain **Domain Users**, no domain-level elevation.
3. Create a security group `Lab-Faculty`.
4. Add every faculty account to `Lab-Faculty`. This one group is what GPOs will target.

```bash
# On the Zentyal server
samba-tool group add Lab-Faculty
samba-tool group addmembers Lab-Faculty jsmith,mgarcia,rwong
```

### 2.5.2 Grant Local Admin on Lab PCs Only (Group Policy Preferences)

1. In `gpmc.msc`, create a GPO `Faculty-LocalAdmin-LabPCs`, linked **only** to the OU containing the lab computer objects.
2. Edit the GPO → **Computer Configuration → Preferences → Control Panel Settings → Local Users and Groups**.
3. Right-click → **New → Local Group**:
   - Group name: **Administrators (built-in)**
   - Action: **Update**
   - Members → Add → `LABSCHOOL\Lab-Faculty` → check **Add to this group**.
4. Verify on a test client:
   ```cmd
   net localgroup administrators
   ```
   should list `LABSCHOOL\Lab-Faculty` alongside the built-in `Administrator`.

**Why not the older "Restricted Groups" node instead?** Restricted Groups *overwrites* the entire local Administrators membership on every refresh, which can silently strip the built-in local `Administrator` account. Group Policy Preferences with **"Update"** is additive and safer here.

### 2.5.3 Scope AppLocker So It Doesn't Block Faculty (see Module 5.2.1)

Local admin rights alone don't bypass AppLocker — handled in **Module 5.2.1**.

## 2.6 Delegate Limited Admin Rights (Optional, Recommended)

```bash
samba-tool ou list
samba-tool delegation add-service-account <lab-admin-user> "Reset Password" --ou="OU=Students,DC=lab,DC=school,DC=local"
```

---

# MODULE 3 — Windows 11 Client Provisioning

## 3.1 Network Adapter / DNS Configuration

1. On each Windows 11 machine: **Settings → Network & Internet → Ethernet → Edit DNS settings**.
2. Set:
   - **Preferred DNS:** `10.0.0.2` (Zentyal) — mandatory.
   - **Alternate DNS:** leave blank, or a secondary DC's IP if you build one later. Never put a public DNS server here.
   - Since Zentyal is now also the DHCP server for this flat network, gateway and DNS should already be correct automatically — you shouldn't need to touch this manually if DHCP is working.
3. Confirm resolution:
   ```cmd
   nslookup lab.school.local
   ping dc1.lab.school.local
   ```

## 3.2 Join the Domain

1. **Settings → Accounts → Access work or school → Connect** → **"Join this device to a local Active Directory domain"** (Windows 11 **Pro/Education** only).
2. Enter the domain name: `lab.school.local`.
3. Authenticate with a domain account that has rights to join computers (a Lab-Admin or Domain Admin account — **not** a student account).
4. Choose which OU the computer object lands in (or accept the default `Computers` OU and move it later via the Zentyal GUI or `samba-tool computer move`).
5. Restart when prompted.
6. At the new logon screen, click **"Other user"** and log in as `LABSCHOOL\studentusername`.

## 3.3 Verification & Troubleshooting Commands (Client-Side)

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

**Important:** Kerberos requires client and DC clocks to be within **5 minutes** of each other. If joins fail with time-sync errors, fix `w32tm` **before** retrying the domain join.

## 3.4 Install RSAT to Manage GPOs

Zentyal's web GUI has no native GPO editor — manage Group Policy from a Windows machine using RSAT, exactly as with a traditional Windows Server AD DC (Samba4 stores GPOs in SYSVOL in Windows-compatible format).

1. On an admin Windows 11 Pro/Education workstation (already domain-joined): **Settings → Optional Features → Add a feature**.
2. Install:
   - **RSAT: Group Policy Management Tools**
   - **RSAT: Active Directory Domain Services and Lightweight Directory Tools**
3. Launch **Group Policy Management** (`gpmc.msc`) — it connects to `lab.school.local` automatically since the machine is domain-joined.

---

# MODULE 4 — Google Workspace Integration (Google Drive instead of OneDrive)

> **Why this replaces the OneDrive module entirely:** GCPW and Google Drive for Desktop are Google Workspace products, unrelated to your AD domain — they're layered on top of, not integrated with, Samba4/Kerberos. You still need your Google Workspace admin (or your own super-admin access to the Workspace Admin Console) to configure the Google side; this module assumes you have that.

## 4.1 What GCPW Actually Does (read this before deploying)

**Google Credential Provider for Windows (GCPW)** adds Google Workspace accounts as an **additional sign-in option** at the Windows logon screen — it does not replace or remove AD domain authentication, and it works alongside an already domain-joined machine. When a student first signs in through GCPW, Windows creates a **local user account** on that machine mapped to their Google identity (separate from their `LABSCHOOL\username` AD account) with a locally-cached credential.

Two things worth knowing going in, given this is a shared-lab scenario and not one-device-per-person:

- **Multi-user login must be explicitly enabled.** By default GCPW settings assume mostly one primary Google user per device. For a shared lab PC where many different students log in over the course of a day, you need the **multi-user login** setting turned on in GCPW's configuration (Admin Console → Devices → Mobile & endpoints → Settings → Windows → GCPW Settings), or only the first student to ever sign in on that machine gets a usable profile going forward.
- **This is a hybrid setup (AD + GCPW), which is a less common combination than either alone.** Pilot it on one or two lab PCs before rolling out to all four rooms — specifically test: a student signing in fresh, signing out, a second student signing in on the same machine, and what happens to each one's local profile after the transient-wipe policy (Module 5.4) runs.

## 4.2 Prepare the Google Workspace Admin Console Side

1. In the Google Admin Console: **Devices → Mobile & endpoints → Settings → Windows settings → Google Credential Provider for Windows (GCPW) setup**.
2. Set **Permitted domains** — this is the only required setting; without it, no one can sign in via GCPW at all. Enter your school's Workspace domain(s).
3. While here, also grab the **enrollment token** and enable **multi-user login** (per 4.1) for the OU that contains your lab devices, if you're managing settings from the Admin Console rather than per-device registry keys.
4. Download the GCPW installer **from the Admin Console** (not the generic public download page) — this build has your organization's enrollment token baked in automatically, which saves you a manual registry step per machine.

## 4.3 Install GCPW on Lab PCs

Manually, per machine (or scripted — see Module 6.8):

```cmd
:: 64-bit installer, silent install
gcpwstandaloneenterprise64.exe /silent /install
```

If you didn't download the installer from the Admin Console (so it has no baked-in token), set the enrollment token manually instead:

1. `regedit` → `HKEY_LOCAL_MACHINE\Software\Policies`
2. Create key `Google`, then inside it create key `CloudManagement`
3. Inside `CloudManagement`, create a **String Value** named `EnrollmentToken`, value = the token you copied in 4.2
4. Restart the device

## 4.4 Deploy Google Drive for Desktop, Forced Workspace Sign-In

1. Download the **Google Drive for Desktop** enterprise MSI and the corresponding **ADMX templates** from Google Workspace's admin resources.
2. Add the ADMX/ADML files to your Central Store (same SYSVOL location as any other admin template):
   ```
   \\lab.school.local\SYSVOL\lab.school.local\Policies\PolicyDefinitions\
   \\lab.school.local\SYSVOL\lab.school.local\Policies\PolicyDefinitions\en-US\
   ```
3. Create a GPO `GoogleDrive-Config`, linked to `Students` (and `Faculty` if desired), under **Computer Configuration → Administrative Templates → Google → Google Drive**:
   - **"Only allow signed-in Google accounts from these domains"** → set to your Workspace domain — this stops a student from syncing a personal @gmail.com account instead of their school one.
   - **"Default web browser"** or auto-launch setting → enable Drive for Desktop to auto-start at login.
   - **Sync mode: use "Stream" (the default), not "Mirror."** This matters specifically because of Module 5.4's transient-profile wipe: in Stream mode, files primarily live in the cloud and are fetched/cached on demand rather than fully duplicated to local disk, so a profile wipe doesn't risk losing unsynced local copies the way Mirror mode's always-present local folder would. Recommend students avoid leaving unsaved work open right before a scheduled profile wipe regardless.
4. Run `gpupdate /force` on a test client, sign in with GCPW (4.3) or a normal AD login plus a manual Google sign-in inside Drive for Desktop, and confirm the Drive icon appears in the system tray with the school-domain account connected.

## 4.5 Block Competing Cloud-Sync Clients

Since you're standardizing on Google Drive, make sure nothing else quietly becomes the "real" backup path:

1. Create/extend a GPO blocking installation and execution of `OneDriveSetup.exe`, `Dropbox.exe`, etc. — this is enforced for free anyway once Module 5's AppLocker whitelist is in place (anything not in `Program Files`/`Windows`, or not explicitly allowed, won't run), but it's worth calling out explicitly here rather than assuming.
2. If any lab image previously had OneDrive pre-installed (common on OEM Windows images), uninstall it during imaging rather than leaving it dormant.

---

# MODULE 5 — Desktop Lockdown & Group Policies

This is the module doing the most work in a pfSense-free design (see the callout in the Architecture Overview) — everything protective about this lab now lives here, at the endpoint.

## 5.1 Block Execution from Flash Drives / Downloads

1. Create a GPO `Student-ExecutionLockdown`, linked to the `Students` OU.
2. Prefer **AppLocker** (5.2) over the legacy **Software Restriction Policies** node.
3. To hard-block **removable media execution** regardless of AppLocker:
   - **Computer Configuration → Administrative Templates → System → Removable Storage Access** → Enable **"All Removable Storage classes: Deny all access,"** or more surgically, **"Removable Disks: Deny execute access."**
4. Combined with AppLocker, this ensures a student can't run a portable `.exe` from a USB stick or a browser-downloaded file even if they copy it locally.

## 5.2 AppLocker Path-Rule Whitelist

1. **Computer Configuration → Policies → Windows Settings → Security Settings → Application Control Policies → AppLocker**.
2. Right-click **Executable Rules** → **Create Default Rules**, then tighten to exactly:
   - **Allow:** Path = `%PROGRAMFILES%\*`
   - **Allow:** Path = `%WINDIR%\*`
   - **Delete/exclude** any default rule whitelisting `%OSDRIVE%\*` broadly, or "all files" for `BUILTIN\Users`.
3. Configure **Windows Installer Rules**, **Script Rules**, and **Packaged app Rules** the same way, so `.msi`, `.ps1`, `.bat`, and UWP sideloading are equally restricted.
4. Enable the AppLocker service: same GPO node → **AppLocker → Properties** → check **"Configured"** for each rule type.
5. Ensure the **Application Identity service** starts automatically:
   - **Computer Configuration → Preferences → Control Panel Settings → Services**, add `AppIDSvc` set to Automatic, or `sc config AppIDSvc start=auto`.
6. Set enforcement to **"Enforce rules"** once testing confirms the whitelist doesn't block legitimate lab software.
7. Test on one machine per room before wide rollout.

### 5.2.1 Faculty Exception

1. In the same GPO (or one linked only to the lab computer OUs), right-click **Executable Rules** → **Create New Rule**.
2. **Permissions:** Action = **Allow**, User or group = `LABSCHOOL\Lab-Faculty`.
3. **Conditions:** a **Path** rule → `%OSDRIVE%\*`, or scope to a dedicated `C:\FacultyInstalls\` staging folder.
4. Repeat for **Windows Installer Rules** and **Script Rules** if faculty need them.
5. Confirm on a test lab PC: student login → random `.exe` fails. Faculty login → same file runs.

## 5.3 Lock Down System Access (Control Panel, Task Manager, Registry, Command-Line)

These close the gap left by not having network-layer enforcement — a student who can't reach `cmd.exe`, `regedit.exe`, or PowerShell can't route around endpoint policy nearly as easily.

1. In a GPO linked to `Students` (e.g. reuse `Student-ExecutionLockdown`), go to **User Configuration → Administrative Templates**:
   - **System → "Prevent access to the command prompt"** → Enabled (also disable script processing if you enable this, per the setting's own sub-option).
   - **System → "Prevent access to registry editing tools"** → Enabled.
   - **Ctrl+Alt+Del Options → "Remove Task Manager"** → Enabled.
   - **Control Panel → "Prohibit access to Control Panel and PC settings"** → Enabled, or use **"Settings Page Visibility"** (below) for finer-grained control instead of an all-or-nothing block.
2. **Control Panel → "Settings Page Visibility"** (a single policy that takes a list) → Enabled, with a value like:
   ```
   hide:network-wifi;personalization-background;personalization-lockscreen;personalization-themes;bluetooth
   ```
   This is the modern replacement for blocking individual legacy Control Panel applets — it hides specific pages inside the Windows 11 Settings app rather than the whole thing, so you can be surgical (e.g., let students still see Ease of Access settings while hiding Wi-Fi and Personalization).
3. **Windows Components → Microsoft Store → "Disable the Store application"** → Enabled.

## 5.4 Lock the Desktop Appearance (Wallpaper, Lock Screen, Themes)

1. **User Configuration → Administrative Templates → Desktop → Desktop → "Desktop Wallpaper"** → Enabled, path set to a UNC location (e.g. `\\dc1\Branding$\lab-wallpaper.jpg`) readable by Domain Computers.
2. **User Configuration → Administrative Templates → Control Panel → Personalization → "Prevent changing desktop background"** → Enabled (belt-and-suspenders alongside the Settings Page Visibility hide above).
3. Same node → **"Prevent changing lock screen and logon image"** → Enabled.
4. **Windows Components → Desktop Window Manager**, or **Personalization** node → disable theme-switching similarly if present in your ADMX version.
5. Add an inactivity **screen saver + lock timeout**:
   - **User Configuration → Administrative Templates → Control Panel → Personalization → "Enable screen saver"** → Enabled.
   - **"Screen saver timeout"** → e.g. `600` seconds.
   - **"Password protect the screen saver"** → Enabled — this is what actually locks the session, not just blanks the screen, important on a shared lab PC.

## 5.5 Transient / Auto-Wiped Local Profiles

You chose to wipe local profiles that haven't been used within **1–3 days**, rather than every logoff. This gives short-term continuity within a class period or a couple of days of a project, while guaranteeing nothing survives more than a few days locally — which is exactly the pressure that makes Google Drive sync (Module 4) mandatory rather than optional.

1. Create a GPO `Student-ProfileWipe`, linked to the lab **computer** OUs (this is a computer-side policy, not user-side).
2. **Computer Configuration → Administrative Templates → System → User Profiles → "Delete user profiles older than a specified number of days on system restart"** → Enabled, set to **2 days** (a reasonable middle point in your chosen 1–3 day range; adjust per room if some classes run projects over a long weekend).
3. Know what this policy does and doesn't touch:
   - It deletes a profile only **on system restart**, and only if that profile hasn't been used (logged into) within the configured window — it does not touch the built-in local `Administrator`, the `Default` profile, or system/service profiles.
   - "Used" is tracked by a registry timestamp (Windows 10 and later), which is more reliable than the older NTFS-timestamp method — but background processes that touch a profile (antivirus scans, sync clients) can occasionally reset that timestamp and delay deletion. If you notice profiles not being cleaned up on schedule, that's the first thing to check.
4. Reboot lab PCs on a schedule (e.g., overnight via Module 6's software-push mechanism, or a simple scheduled task) so the deletion actually has a restart to trigger on — a PC that's never rebooted never runs this cleanup, no matter how the policy is configured.
5. Roll this out to one room first and confirm behavior over a few real school days before applying lab-wide — profile cleanup policies are exactly the kind of thing that looks fine in testing and then behaves unexpectedly once real student usage patterns (multiple logins per day, some machines never rebooted) show up.

## 5.6 Restrict Wi-Fi: No Ability to Add New Networks

You chose to leave any existing Wi-Fi profiles alone and just remove the ability for students to add new ones, rather than hard-blocking every SSID except CCSE's.

> **Honest caveat:** Windows 11 doesn't have one clean "block adding Wi-Fi networks only" toggle the way it does for, say, Task Manager. The practical approach is hiding the UI that lets someone add a network, combined with the command-line lockdown from 5.3 (since `netsh wlan add profile` is exactly how someone would add one from the command line instead).

1. You already hid the Wi-Fi settings page in 5.3's Settings Page Visibility value (`hide:network-wifi`) — that removes "Add a network" from the Settings app UI for standard users.
2. Also hide the legacy Control Panel route: **User Configuration → Administrative Templates → Network → Network Connections → "Prohibit access to properties of a LAN connection"** and the equivalent for wireless if present in your ADMX version.
3. Because 5.3 already blocks Command Prompt and PowerShell for students, `netsh wlan add profile` isn't reachable either — this is a good example of how the endpoint-side lockdowns in this module reinforce each other rather than each needing to be airtight alone.
4. Any Wi-Fi profile you *want* every lab PC to have (e.g., the CCSE SSID with its credentials pre-configured) should be provisioned once, centrally, before this policy locks the UI down — either manually per machine during imaging, or via a **Wireless Network (IEEE 802.11) Policy** in `gpmc.msc` (**Computer Configuration → Policies → Windows Settings → Security Settings → Wireless Network (IEEE 802.11) Policies**), which pushes a specific SSID/profile without needing to touch every device by hand. Leave "Only use Group Policy profiles for allowed networks" **unchecked** — checking it would strip out any other existing profiles, which you explicitly want to leave alone.

## 5.7 Pushing Software Updates While Staying Locked Down

Because students can't install anything, all software deployment must be **admin-pushed**. Free options:

**Option A — GPO Software Installation (native, simplest):**
1. Place `.msi` installers on a network share (e.g., `\\dc1\Software$`) readable by Domain Computers.
2. In a GPO linked to your **Computers** OU: **Computer Configuration → Policies → Software Settings → Software Installation → New → Package**, point to the `.msi`, choose **Assigned**.
3. Installs automatically at next machine boot.

**Option B — Free Deployment Tooling:**
- **Chocolatey**, pushed via a **Computer Startup Script** GPO running as `SYSTEM` (bypasses the AppLocker/standard-user restriction, since SYSTEM-run scripts aren't subject to the same lockdown as the student session):
  ```powershell
  Set-ExecutionPolicy Bypass -Scope Process -Force
  iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
  choco upgrade all -y
  ```
- **PDQ Deploy (Free tier)** — GUI-based, good across four rooms from one console.
- **WSUS** (free) for Windows Update management specifically.

---

# MODULE 6 — Infrastructure as Code (Ansible)

> **Context for this environment:** bare metal, single Zentyal box, no router/firewall of your own — there's no VM/network-device resource for Terraform to declare, so Ansible carries the whole stack: Samba/AD object management, GPO-equivalent registry pushes, and Windows client configuration over WinRM.

## 6.0 Honest Caveats Before You Commit to This

- **Zentyal's GUI is not the source of truth once you do this.** Treat **Samba4 (`samba-tool`) as the real source of truth** and use Zentyal's GUI purely as an optional read/monitoring dashboard.
- **`samba-tool` commands are not natively idempotent** — every task below wraps calls with existence checks.
- **GPOs are semi-opaque as code.** Two viable strategies: (1) **Backup/Restore roundtrip** via `Backup-GPO`/`Import-GPO`, simple but git-unreadable binary blobs; (2) **Granular registry-value management**, shown below for the settings that map cleanly to a registry key (OneDrive-equivalent policies, GCPW/Drive settings, profile-wipe policy). AppLocker specifically uses its own XML export/import cmdlets.
- **WinRM chicken-and-egg problem.** A freshly domain-joined PC doesn't have WinRM enabled by default — bootstrap it once via a GPO enabling the listener + firewall rule, or Microsoft's `ConfigureRemotingForAnsible.ps1` during imaging.
- **Single box, no redundancy.** With no hypervisor and no second server, there's no snapshot/rollback safety net if an in-place Ubuntu upgrade goes wrong. Back up Zentyal's Samba database on a schedule (`samba-tool domain backup offline`) and store it off-box.

## 6.1 Repository Layout

```
lab-iac/
├── ansible.cfg
├── inventory/
│   ├── hosts.yml                  # dc1, plus all lab PCs grouped by room
│   └── group_vars/
│       ├── all.yml                # domain name, GCPW settings, etc.
│       └── windows_clients.yml    # winrm connection vars
├── data/
│   ├── students.yml               # source of truth for student accounts
│   ├── faculty.yml
│   └── computers.yml              # hostname -> room/OU mapping
├── roles/
│   ├── zentyal_samba_dc/          # AD + DNS + DHCP bootstrap
│   ├── ad_objects/                # OUs, groups, users, logon hours
│   ├── windows_client_join/
│   ├── google_workspace_policy/   # GCPW + Drive for Desktop registry settings
│   ├── profile_wipe_policy/       # transient/auto-wipe profile registry settings
│   ├── applocker_policy/
│   └── software_push/
├── gpo-backups/                   # only if using the backup/restore GPO strategy
└── site.yml
```

## 6.2 Control Node Prerequisites

```bash
pip install --user ansible pywinrm requests-ntlm requests-kerberos
ansible-galaxy collection install ansible.windows community.windows microsoft.ad
```

`microsoft.ad` provides modules like `microsoft.ad.user`, `microsoft.ad.ou`, `microsoft.ad.group` that speak native AD/LDAP against any AD-compatible DC, including Samba4 — fall back to raw `samba-tool` for anything it doesn't support (e.g., initial domain provisioning).

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

## 6.4 Role: `zentyal_samba_dc` (Server Bootstrap, incl. DHCP)

```yaml
# roles/zentyal_samba_dc/tasks/main.yml
- name: Install Zentyal core + Samba AD + DHCP packages
  apt:
    name: [zentyal-samba, zentyal-dns, zentyal-dhcp, samba, krb5-user, isc-dhcp-server]
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

- name: Template DHCP scope config
  template:
    src: dhcpd.conf.j2
    dest: /etc/dhcp/dhcpd.conf
    mode: "0644"
  notify: restart isc-dhcp-server
```

`ad_realm`, `ad_netbios`, `ad_admin_password` live in `inventory/group_vars/all.yml` (vault-encrypt the password: `ansible-vault encrypt_string`).

## 6.5 Role: `ad_objects` (OUs, Groups, Users, Logon Hours)

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
  command: samba-tool group addmembers "Students" "{{ item.username }}"
  loop: "{{ students }}"
  changed_when: true

- name: Restrict student logon hours via microsoft.ad
  microsoft.ad.user:
    identity: "{{ item.username }}"
    logon_hours: "{{ default_student_logon_hours }}"  # e.g. Mon-Fri 07:00-17:00 bitmask
  loop: "{{ students }}"
  delegate_to: dc1
```

Repeat for `faculty.yml` targeting `Lab-Faculty` (no logon-hours restriction needed for staff, typically).

## 6.6 Role: `windows_client_join`

```yaml
# roles/windows_client_join/tasks/main.yml
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

- name: Install RSAT features on admin workstations only
  ansible.windows.win_optional_feature:
    name:
      - Rsat.GroupPolicy.Management.Tools~~~~0.0.1.0
      - Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0
    state: present
  when: "'admin_workstations' in group_names"
```

## 6.7 Role: `profile_wipe_policy` (Transient Profiles, Registry-Level)

```yaml
# roles/profile_wipe_policy/tasks/main.yml
- name: Enable "Delete user profiles older than N days on restart"
  ansible.windows.win_regedit:
    path: HKLM:\SOFTWARE\Policies\Microsoft\Windows\System
    name: RemoveProfileDays
    data: 2   # your chosen middle-ground within the 1-3 day range
    type: dword
```

## 6.8 Role: `google_workspace_policy` (GCPW + Drive for Desktop)

```yaml
# roles/google_workspace_policy/tasks/main.yml
- name: Ensure Google policy registry path exists and set CloudManagement enrollment token
  ansible.windows.win_regedit:
    path: HKLM:\SOFTWARE\Policies\Google\CloudManagement
    name: EnrollmentToken
    data: "{{ gcpw_enrollment_token }}"
    type: string

- name: Enable multi-user login for GCPW (critical for shared lab PCs)
  ansible.windows.win_regedit:
    path: HKLM:\SOFTWARE\Policies\Google\GCPW
    name: enable_multi_user_login
    data: 1
    type: dword

- name: Restrict Drive for Desktop to school Workspace domain only
  ansible.windows.win_regedit:
    path: HKLM:\SOFTWARE\Policies\Google\DriveFS
    name: domains_allowed_to_login
    data: "{{ workspace_domain }}"
    type: string

- name: Install GCPW silently
  ansible.windows.win_package:
    path: "\\\\dc1\\Software$\\gcpwstandaloneenterprise64.exe"
    arguments: "/silent /install"
    state: present

- name: Install Google Drive for Desktop silently
  ansible.windows.win_package:
    path: "\\\\dc1\\Software$\\GoogleDriveSetup.exe"
    arguments: "--silent"
    state: present
```

`gcpw_enrollment_token` and `workspace_domain` live in `inventory/group_vars/all.yml`, vault-encrypt the token the same way as the AD admin password.

## 6.9 AppLocker as Exported XML Policy

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

## 6.10 Role: `software_push`

```yaml
# roles/software_push/tasks/main.yml
- name: Install course software via Chocolatey
  chocolatey.chocolatey.win_chocolatey:
    name: "{{ item }}"
    state: present
  loop: "{{ room_software_list }}"
```

## 6.11 Running It & Keeping Drift in Check

```bash
ansible-playbook -i inventory/hosts.yml site.yml --vault-password-file .vault-pass
```

- Run after **any** change to `data/*.yml` — new student, new faculty member, room reassignment.
- Consider a nightly `cron` run in **check mode** (`--check --diff`) that emails you a drift report without applying changes automatically.
- Back up Zentyal's Samba database on a schedule (`samba-tool domain backup offline`) and store it off-box, given there's no hypervisor snapshot safety net.

---

# ADMINISTRATOR'S CHEAT SHEET

## Samba/Zentyal `samba-tool` Essentials (run on the Zentyal box, as root/sudo)

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

# Backup (do this on a schedule)
samba-tool domain backup offline --targetdir=/backup/samba-$(date +%F)
```

## Network Verification Commands

| Purpose | Command (run on server or client as noted) |
|---|---|
| Verify AD DNS SRV records (server) | `dig _ldap._tcp.lab.school.local SRV` |
| Verify Kerberos SRV records (server) | `dig _kerberos._udp.lab.school.local SRV` |
| Test LDAP bind (server) | `ldapsearch -x -H ldap://localhost -b "dc=lab,dc=school,dc=local"` |
| Check Samba AD service status (server) | `systemctl status samba-ad-dc` |
| Check DHCP leases (server) | `cat /var/lib/dhcp/dhcpd.leases` |
| Client secure channel check (client) | `nltest /sc_query:lab.school.local` |
| Client domain join test (client) | `nltest /dsgetdc:lab.school.local` |
| Force time sync (client) | `w32tm /resync /force` |
| Flush DNS (client) | `ipconfig /flushdns && ipconfig /registerdns` |
| Kerberos ticket check (client) | `klist` |
| Force GPO refresh (client) | `gpupdate /force` |
| GPO application report (client) | `gpresult /h report.html` |
| Check GCPW enrollment token (client) | `reg query HKLM\SOFTWARE\Policies\Google\CloudManagement` |
| Check profile-wipe policy is applied (client) | `reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\System /v RemoveProfileDays` |

## Troubleshooting Matrix — Common Errors

| Symptom / Error | Likely Cause | Fix |
|---|---|---|
| "An Active Directory Domain Controller could not be contacted" | Client DNS not pointing to Zentyal | Set client DNS to `10.0.0.2`; `ipconfig /flushdns` |
| `KRB_AP_ERR_SKEW` / clock skew errors | Client/DC time difference >5 min | `w32tm /resync /force`; verify DC is an NTP source |
| "The trust relationship between this workstation and the primary domain failed" | Machine account password out of sync | Rejoin domain, or `Reset-ComputerMachinePassword -Server dc1.lab.school.local` |
| Domain join dialog greyed out / not available | Windows 11 **Home** edition in use | Upgrade to Pro/Education |
| Users can't log in with `DOMAIN\user` after join | NetBIOS name mismatch or DNS suffix not applied | Confirm NetBIOS name matches login prompt; check **System → About → Rename this PC (domain)** |
| GPOs not applying to student machines | Computer object in wrong OU, or `gpupdate` not run/rebooted | `samba-tool computer move`; `gpupdate /force` + reboot |
| AppLocker not blocking anything despite "Enforce" mode | `AppIDSvc` not running | `sc config AppIDSvc start=auto && sc start AppIDSvc` |
| Students still show as local admin post-join | Account manually added to local `Administrators`, or `Domain Users` mistakenly nested into local admins | Check `net localgroup administrators`; remove the entry |
| Faculty can't install software despite local admin | AppLocker still blocking | Verify the `Lab-Faculty`-scoped Allow rule (5.2.1); confirm `AppIDSvc` is running |
| GCPW sign-in fails for a student | Permitted domains not set, or enrollment token missing/wrong | Re-check Admin Console → GCPW Settings → Permitted domains; verify `HKLM\SOFTWARE\Policies\Google\CloudManagement\EnrollmentToken` |
| Second student on same PC can't sign in via GCPW | Multi-user login not enabled | Enable `enable_multi_user_login` in GCPW settings/registry |
| Google Drive syncing personal @gmail account instead of school account | "Only allow signed-in accounts from these domains" not configured | Set the domain restriction in the `GoogleDrive-Config` GPO (Module 4.4) |
| Profile-wipe policy not deleting old profiles | A background process (AV, sync client) keeps resetting the profile's "last used" timestamp; or the PC is never rebooted | Confirm the machine actually restarts on a schedule; check for services touching profile files |
| Student able to add a new Wi-Fi network anyway | Command line not actually locked down, or Settings Page Visibility value missing `network-wifi` | Re-verify 5.3's command-prompt/PowerShell block is applied; re-check the Settings Page Visibility string |
| Slow logons across the four rooms | DNS forwarder misconfigured, internal lookups leaking out unnecessarily | Ensure Zentyal's BIND has `lab.school.local` as authoritative, forwards only genuinely external names |

---

**Deployment Order Recap:** Module 1 (Zentyal install: AD + DNS + DHCP on one box) → Module 2 (accounts/OUs, logon hours, `Faculty` OU + `Lab-Faculty` group) → Module 3 (join all client PCs, install RSAT) → Module 4 (GCPW + Google Drive for Desktop, replacing OneDrive) → Module 5 (desktop lockdown: AppLocker, Control Panel/Task Manager/Registry/command-line blocks, wallpaper/lock-screen lock, transient profile wipe, Wi-Fi restriction) → Module 6 (wrap the ongoing config in Ansible, so re-running `site.yml` is how you onboard, patch, and audit going forward) → validate with the Cheat Sheet before handing the lab over to students and faculty.
