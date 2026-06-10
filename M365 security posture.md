# Securing Hybrid Identity: An End-to-End Microsoft 365 and Entra ID Security Implementation

## Design and Implementation of a Hybrid Identity Security Architecture across On-Premise Active Directory, Microsoft Entra ID, and Microsoft 365

*A practical engineering Project demonstrating enterprise identity protection, access hardening, secure lifecycle management, and identity threat detection in a hybrid estate.*

| Field | Detail |
|---|---|
| Author | Dumanyie Dornubari Chamberlain  |
| Role  | M365 Security Engineer |
| Project discipline | Hybrid Identity and Access Management Security |
| Environment | Representative experience (non-production) |
| Version | 2.0 (implemented build with evidence) |
| Date | June 2026 |

> **About this document.** This is the as-built record of a Project I designed, broke, fixed, and validated end to end. Every screenshot is from my own tenant and server. Where something failed the first time, I have kept the failure and the fix in the write-up, because the troubleshooting is the part that proves the work is real.

---

## Contents

1. Introduction and Aim
2. Prerequisites, Tools, and Safety
3. Phase 1 — Build the On-Premise Foundation
4. Phase 2 — Establish Hybrid Identity
5. Phase 3 — Harden Identity and Access
6. Phase 4 — Secure the Identity Lifecycle
7. Phase 5 — Detection and Response
8. Risk Posture, Before and After
9. Limitations and Honest Assessment
10. Mapping to Recognised Frameworks
11. Conclusion
- Appendix A — Evidence Checklist
- Appendix B — Command and Query Reference

---

## 1. Introduction and Aim

This document records a complete, hands-on Projectoratory build. The aim was to construct a hybrid identity environment — where an on-premise Active Directory is synchronised into the Microsoft cloud — and then to secure that environment to a professional standard using Microsoft 365 and Azure security capabilities.

The Project deliberately mirrors a real enterprise. Large organisations rarely run purely in the cloud. They keep on-premise infrastructure and extend it into Microsoft Entra ID (formerly Azure Active Directory) so that staff carry a single identity across both worlds. That hybrid model is convenient, but it inherits risk from both sides. The work below shows how to reduce that risk in a structured, measurable way.

Every phase follows the same rhythm: each task is described in plain language, the exact command or setting is shown, and each security control is explained as *what I configured, why it matters, the threat it mitigates, and how I validated it*.

### 1.1 Learning Outcomes

- Build a hybrid identity foundation by synchronising on-premise Active Directory to Microsoft Entra ID.
- Apply identity and access controls including Conditional Access, multi-factor authentication, and just-in-time privileged access.
- Implement a secure joiner, mover, and leaver lifecycle, treating deprovisioning as a security control rather than housekeeping.
- Ingest identity logs into a SIEM and detect a realistic identity-based attack chain.
- Document the work as a professional deliverable framed in business and regulatory risk terms.

### 1.2 Architecture Overview

On-premise Active Directory holds the authoritative staff accounts. Microsoft Entra Connect synchronises those accounts into Entra ID. Entra ID then becomes the identity provider for Microsoft 365 services. Security controls and logging sit on top of that flow.

```
On-Premise Active Directory  (corp.freedomProject.local)
        |   Entra Connect: password hash synchronisation
        v
Microsoft Entra ID   (Conditional Access | MFA | PIM | Identity Protection)
        |   sign-in and audit logs
        v
Microsoft 365 Services   ---->   SIEM (Microsoft Sentinel)
```

---

## 2. Prerequisites, Tools, and Safety

### 2.1 Accounts and Platforms

| Resource | Purpose | Cost |
|---|---|---|
| Microsoft 365 Developer Program (E5) | Free tenant providing Entra ID P2 (Conditional Access, PIM, Identity Protection) | Free |
| Azure subscription | Hosts the Windows Server VM, networking, and the Sentinel workspace | Free credit / pay-as-you-go |
| Windows Server 2022 VM | Acts as the on-premise domain controller | Within free credit |
| Microsoft Entra Connect | Synchronisation engine between AD and Entra ID | Free |
| Microsoft Sentinel | SIEM for log ingestion and detection | Pay-as-you-go (minimal at Project scale) |

> **A note from the build.** My tenant is `universityofroehampton747.onmicrosoft.com`, created through the Microsoft 365 Developer Program. Because my school Azure account was restricted, I kept the cloud identity work on this separate developer tenant and the on-premise VM on the Azure side — which is realistic, since infrastructure and identity are often managed separately in real organisations.

### 2.2 Software and Modules

- Active Directory Domain Services role on Windows Server 2022.
- Microsoft Entra Connect (now downloaded from the Entra admin centre, not the old Download Centre).
- PowerShell module: `Microsoft.Graph` for scripting lifecycle tasks; Microsoft Graph Explorer as a browser-based alternative.
- Access to the Microsoft Entra admin centre, the Azure portal, and Microsoft Sentinel.

### 2.3 Safety and Ethics Notice

**Important.** Every attack simulation in Phase 5 was performed only inside my own developer tenant and Project network. These techniques must never be tested against a production tenant, a real employer estate, or any account you do not own. The simulation exists to prove the detections fire, not to attack anyone.

**Honesty guardrail.** I claim only what I built and validated myself. Where a specialist team would normally own a component in production, I say so.

---

## 3. Phase 1 — Build the On-Premise Foundation

**Goal.** Stand up a Windows domain controller that represents the legacy on-premise estate. This is the authoritative source of staff identities before any cloud synchronisation.

### 3.1 Provision the Server

In the Azure portal I created a Windows Server 2022 Datacenter virtual machine (size B2s is plenty for a Project), set a strong local administrator password, and — importantly — allowed inbound RDP (port 3389) only from my own IP address in the Network Security Group. Leaving RDP open to the whole internet is one of the most common and most exploited mistakes, so locking it down is the very first security control in the Project.

![Figure 1: Azure portal: Windows Server VM and networking resources provisioned](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image1.png)

*Figure 1: Azure portal: Windows Server VM and networking resources provisioned.*

### 3.2 Connect Over RDP

Think of RDP (Remote Desktop Protocol) as a window onto the server's desktop, even though the machine lives in Azure's data centre. From the VM's **Connect** blade I downloaded the `.rdp` file, opened it, entered the server credentials, accepted the certificate prompt, and landed on the Windows Server desktop.

![Figure 2: VM Connect blade: downloading the RDP file for the domain controller](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image2.png)

*Figure 2: VM Connect blade: downloading the RDP file for the domain controller.*

![Figure 3: Downloaded .rdp file ready to launch](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image3.png)

*Figure 3: Downloaded .rdp file ready to launch.*

![Figure 4: Windows Security credential prompt when connecting over RDP](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image4.png)

*Figure 4: Windows Security credential prompt when connecting over RDP.*

![Figure 5: RDP certificate trust prompt for the Project server](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image5.png)

*Figure 5: RDP certificate trust prompt for the Project server.*

![Figure 6: Windows Server desktop after a successful RDP connection](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image6.png)

*Figure 6: Windows Server desktop after a successful RDP connection.*

### 3.3 Install Active Directory Domain Services and Promote the Domain Controller

Active Directory is like a company's master register of users and computers; the **Domain Controller** is the server that runs it. Installing the role just places the software; *promoting* the server is what actually switches it on as a domain controller and creates the domain.

I did this with PowerShell rather than the wizard because it is faster and repeatable:

```powershell
# Install the AD DS role together with its management tools
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# Create a brand-new forest and promote this server to be its first domain controller
Install-ADDSForest `
  -DomainName "corp.freedomProject.local" `
  -InstallDns `
  -Force
```

**What each part does.** `Install-WindowsFeature` adds the directory-service software. `Install-ADDSForest` builds a new forest and domain, installs DNS (which Active Directory depends on to locate its services), and prompts for a DSRM recovery password. The server reboots automatically when promotion completes.

![Figure 7: Installing the AD DS role (installation in progress)](https://github.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/blob/c012d66fdd6ddfa875ac62a91df7bc889619d309/M365%20resources/image7.png)

*Figure 7: Installing the AD DS role (installation in progress).*

![Figure 8: Install-WindowsFeature completes with Success: True](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image8.png)

*Figure 8: Install-WindowsFeature completes with Success: True.*

![Figure 9: Install-ADDSForest running with the DSRM password prompt](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image9.png)

*Figure 9: Install-ADDSForest running with the DSRM password prompt.*

![Figure 10: Forest promotion validating prerequisites before reboot](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image10.png)

*Figure 10: Forest promotion validating prerequisites before reboot.*

**Validation.** After the reboot I confirmed the domain was healthy. `Get-ADDomain` returns the domain details and the core services (`DNS`, `ADWS`, `NTDS`) all report *Running*.

![Figure 11: Get-ADDomain and service checks confirm the domain controller is live](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image11.png)

*Figure 11: Get-ADDomain and service checks confirm the domain controller is live.*

**A real fix worth recording.** A domain controller must use *itself* for DNS. My first attempt to set this with `Set-DnsClientServerAddress -InterfaceIndex $adapter.InterfaceIndex` failed with *No MSFT_DNSClientServerAddress objects found with InterfaceIndex 4* — the adapter object picked up the wrong index. `Get-NetIPConfiguration` showed the real adapter was **interface index 7**, with DNS already pointing to `::1` and `10.0.0.4` (the server itself). So the configuration was already correct and no change was needed. The lesson: verify the actual interface before forcing a change.

![Figure 12: Network adapter and DNS verification (interface index troubleshooting)](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image12.png)

*Figure 12: Network adapter and DNS verification (interface index troubleshooting).*

### 3.4 Create a Realistic Organisational Structure

To mirror a real business I built Organisational Units (think of them as folders) and populated them with users. I opened **Active Directory Users and Computers** to confirm the domain tree, then scripted the rest.

![Figure 13: Active Directory Users and Computers open on corp.freedomProject.local](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image13.png)

*Figure 13: Active Directory Users and Computers open on corp.freedomProject.local.*

**Create the OUs:**

```powershell
# Top-level OUs
New-ADOrganizationalUnit -Name "Subsidiaries"     -Path "DC=corp,DC=freedomProject,DC=local"
New-ADOrganizationalUnit -Name "Departments"      -Path "DC=corp,DC=freedomProject,DC=local"
New-ADOrganizationalUnit -Name "Service Accounts" -Path "DC=corp,DC=freedomProject,DC=local"

# Department sub-OUs
New-ADOrganizationalUnit -Name "IT"         -Path "OU=Departments,DC=corp,DC=freedomProject,DC=local"
New-ADOrganizationalUnit -Name "Finance"    -Path "OU=Departments,DC=corp,DC=freedomProject,DC=local"
New-ADOrganizationalUnit -Name "HR"         -Path "OU=Departments,DC=corp,DC=freedomProject,DC=local"
New-ADOrganizationalUnit -Name "Operations" -Path "OU=Departments,DC=corp,DC=freedomProject,DC=local"
```

Here `OU=Departments,DC=corp,DC=freedomProject,DC=local` is simply the AD way of writing *the Departments folder inside the corp.freedomProject.local domain*.

![Figure 14: Creating the Organisational Unit structure with PowerShell](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image14.png)

*Figure 14: Creating the Organisational Unit structure with PowerShell.*

**Bulk-create eighteen ordinary staff accounts.** The `1..18 | ForEach-Object` loop runs the block once per number, so `$_` becomes 1, 2, 3 … 18:

```powershell
1..18 | ForEach-Object {
  New-ADUser -Name "Test User $_" `
    -SamAccountName "tuser$_" `
    -UserPrincipalName "tuser$_@corp.freedomProject.local" `
    -Path "OU=Departments,DC=corp,DC=freedomProject,DC=local" `
    -AccountPassword (ConvertTo-SecureString "Project!Pass123" -AsPlainText -Force) `
    -Enabled $true
}
```

`ConvertTo-SecureString` is required because PowerShell will not accept a plain-text password directly; `-Enabled $true` makes each account active.

![Figure 15: Bulk-creating eighteen test user accounts](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image15.png)

*Figure 15: Bulk-creating eighteen test user accounts.*

**Two administrative accounts** (note the `adm.` prefix — a real-world convention that makes admin accounts easy to spot and monitor), added to **Domain Admins**:

```powershell
New-ADUser -Name "Admin Freedom" -SamAccountName "adm.freedom" `
  -UserPrincipalName "adm.freedom@corp.freedomProject.local" `
  -Path "OU=IT,OU=Departments,DC=corp,DC=freedomProject,DC=local" `
  -AccountPassword (ConvertTo-SecureString "Project!Admin456" -AsPlainText -Force) -Enabled $true

New-ADUser -Name "Admin Support" -SamAccountName "adm.support" `
  -UserPrincipalName "adm.support@corp.freedomProject.local" `
  -Path "OU=IT,OU=Departments,DC=corp,DC=freedomProject,DC=local" `
  -AccountPassword (ConvertTo-SecureString "Project!Admin456" -AsPlainText -Force) -Enabled $true

Add-ADGroupMember -Identity "Domain Admins" -Members "adm.freedom","adm.support"
```

![Figure 16: Creating two administrative accounts and adding them to Domain Admins](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image16.png)

*Figure 16: Creating two administrative accounts and adding them to Domain Admins.*

**Two deliberately stale accounts** (`jlegacy`, `boldstaff`). These represent the dormant accounts that pile up in every large organisation when people leave but their accounts are never disabled. Attackers love them because nobody is watching them. They become the teaching point for Phases 4 and 5.

```powershell
New-ADUser -Name "Jane Legacy" -SamAccountName "jlegacy" `
  -UserPrincipalName "jlegacy@corp.freedomProject.local" `
  -Path "OU=Departments,DC=corp,DC=freedomProject,DC=local" `
  -AccountPassword (ConvertTo-SecureString "Project!Pass123" -AsPlainText -Force) -Enabled $true

New-ADUser -Name "Bob Oldstaff" -SamAccountName "boldstaff" `
  -UserPrincipalName "boldstaff@corp.freedomProject.local" `
  -Path "OU=Departments,DC=corp,DC=freedomProject,DC=local" `
  -AccountPassword (ConvertTo-SecureString "Project!Pass123" -AsPlainText -Force) -Enabled $true
```

![Figure 17: Creating the two deliberately stale (dormant) accounts](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image17.png)

*Figure 17: Creating the two deliberately stale (dormant) accounts.*

**Validation.** I confirmed every account exists, the admin accounts are in Domain Admins, the total user count is correct, and all OUs were created.

```powershell
Get-ADUser -Filter * -SearchBase "OU=Departments,DC=corp,DC=freedomProject,DC=local" | Select Name,SamAccountName,Enabled
Get-ADGroupMember -Identity "Domain Admins" | Select Name,SamAccountName
(Get-ADUser -Filter *).Count
Get-ADOrganizationalUnit -Filter * | Select Name,DistinguishedName
```

![Figure 18: Verifying all created users appear in the Departments OU](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image18.png)

*Figure 18: Verifying all created users appear in the Departments OU.*

![Figure 19: Validating admin membership, total user count, and OU list](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image19.png)

*Figure 19: Validating admin membership, total user count, and OU list.*

The on-premise foundation is now complete.

---

## 4. Phase 2 — Establish Hybrid Identity

**Goal.** Synchronise the on-premise directory into Microsoft Entra ID, recreating the identity migration carried out in real enterprises — except here I own the whole process.

### 4.1 Prepare the Entra Tenant

I signed in to the Entra admin centre (`https://entra.microsoft.com`) with my developer-tenant admin account, confirmed the tenant has **Entra ID P2** features (provided by the E5 developer subscription, and required later for PIM and Identity Protection), and chose to keep the default `onmicrosoft.com` domain so there were no public DNS changes to make.

### 4.2 Plan the Synchronisation Scope (a security decision)

Before installing anything I decided which accounts should and should not reach the cloud:

- **Synchronise:** ordinary staff in the **Departments** and **Subsidiaries** OUs.
- **Exclude:** the stale accounts, on-premise-only service accounts, and built-in administrative accounts.

**Why this matters.** Every account I do not sync is an account that cannot be attacked in the cloud. Filtering the sync scope is the principle of *attack-surface reduction* applied to identity.

### 4.3 Install and Configure Entra Connect

Entra Connect is the bridge between on-premise AD and the cloud. It must be installed **on the domain controller (the VM)** because only the VM can read Active Directory. Note that new versions are now downloaded from the **Entra admin centre** (the old Microsoft Download Centre link is retired). In the wizard I chose **Customised** settings, selected **Password Hash Synchronisation**, filtered to only the Departments and Subsidiaries OUs, and enabled **Seamless Single Sign-On**.

**Why Password Hash Synchronisation?** It is the simplest secure option and keeps cloud sign-in working even if the on-premise server is offline. The alternatives — Pass-through Authentication and Federation (ADFS) — keep authentication on-premise but add infrastructure and failure points. I chose hash sync deliberately for resilience and simplicity, and I understand the trade-offs of the alternatives.

**A real fix worth recording.** The wizard refused my Domain Admin and Enterprise Admin accounts with *using an enterprise or domain administrator account is not allowed*. The current version expects you to let it **create its own dedicated AD DS service account**; you supply Enterprise Admin credentials only once, so it can create that lower-privilege account. This is actually better security — the day-to-day sync account is not a domain admin.

![Figure 20: Entra Connect: sign-in / UPN suffix configuration screen](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image20.png)

*Figure 20: Entra Connect: sign-in / UPN suffix configuration screen.*

![Figure 21: Entra Connect: configuration complete and first sync initiated](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image21.png)

*Figure 21: Entra Connect: configuration complete and first sync initiated.*

![Figure 22: Entra Connect: full wizard completed (Connect, Filtering, SSO, Configure)](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image22.png)

*Figure 22: Entra Connect: full wizard completed (Connect, Filtering, SSO, Configure).*

**Validation.** In the Entra admin centre the on-premise users now appear as *synced from on-premises* rather than *cloud-only*, and a synced user can sign in to the Microsoft 365 portal.

![Figure 23: Synced on-premise users now visible in the Entra admin centre](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image23.png)

*Figure 23: Synced on-premise users now visible in the Entra admin centre.*

![Figure 24: A synced user signed into Microsoft 365 (Welcome back, Test User 2)](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image24.png)

*Figure 24: A synced user signed into Microsoft 365 (Welcome back, Test User 2).*

---

## 5. Phase 3 — Harden Identity and Access

**Goal.** Apply the access controls that separate a security engineer from an administrator. Each control is described as *what / why / threat / validation*. Conditional Access policies were built in **report-only** mode first so I could observe their impact before enforcing — the discipline that stops you locking yourself out.

Before the policies, I registered strong MFA for a test user via the Microsoft Authenticator QR code.

![Figure 25: Microsoft Authenticator registration via QR code](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image25.png)

*Figure 25: Microsoft Authenticator registration via QR code.*

### 5.1 Conditional Access Policies

Conditional Access is the security guard at the door: it checks conditions on every sign-in and decides allow, block, or challenge.

**Policy 1 — Require MFA for all users.**
*What:* MFA required for all users across all cloud apps. *Why:* passwords are routinely stolen through phishing and reuse; MFA adds a factor the attacker is unlikely to hold. *Threat mitigated:* account takeover with stolen passwords. *Validation:* signing in as a test user triggers the MFA prompt.

![Figure 26: Conditional Access Policy 1: Require MFA for all users (report-only)](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image26.png)

*Figure 26: Conditional Access Policy 1: Require MFA for all users (report-only).*

![Figure 27: Conditional Access policy list after creating the MFA policy](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image27.png)

*Figure 27: Conditional Access policy list after creating the MFA policy.*

**Policy 2 — Block legacy authentication.**
*What:* block older protocols (e.g. basic auth) that cannot enforce MFA. *Why:* legacy protocols are a common bypass route precisely because they ignore modern controls. *Threat mitigated:* MFA bypass via legacy protocol abuse. *Validation:* a legacy-protocol sign-in is blocked.

![Figure 28: Conditional Access Policy 2: targeting legacy client apps](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image28.png)

*Figure 28: Conditional Access Policy 2: targeting legacy client apps.*

![Figure 29: Conditional Access Policy 2: Block access grant control](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image29.png)

*Figure 29: Conditional Access Policy 2: Block access grant control.*

**Policy 3 — Require a compliant device for administrators.**
*What:* admin accounts may only sign in from a compliant/managed device. *Why:* admin access is the highest-value target; tying it to a known device raises the bar even if credentials and MFA are phished. *Threat mitigated:* use of stolen admin credentials from an attacker-controlled device. *Validation:* an admin sign-in from an unmanaged device is challenged or blocked.

![Figure 30: Conditional Access Policy 3: selecting the administrative accounts](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image30.png)

*Figure 30: Conditional Access Policy 3: selecting the administrative accounts.*

![Figure 31: Conditional Access Policy 3: require a compliant device](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image31.png)

*Figure 31: Conditional Access Policy 3: require a compliant device.*

![Figure 32: Conditional Access policy list: three policies created](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image32.png)

*Figure 32: Conditional Access policy list: three policies created.*

**Policy 4 — Restrict sign-in by location.**
*What:* challenge or block sign-ins from countries the organisation does not expect, using a named location for the United Kingdom. *Why:* geographic anomalies are a strong early signal of compromise. *Threat mitigated:* sign-ins from unexpected regions after credential theft. *Validation:* a sign-in from outside the named location is challenged.

![Figure 33: Conditional Access named location: Allow - United Kingdom](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image33.png)

*Figure 33: Conditional Access named location: Allow - United Kingdom.*

![Figure 34: Conditional Access Policy 4: restrict sign-in by location](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image34.png)

*Figure 34: Conditional Access Policy 4: restrict sign-in by location.*

### 5.2 Privileged Identity Management (PIM)

*What:* I converted privileged roles, including Global Administrator, from **permanently assigned** to **eligible**. Activation now requires justification, MFA, and is time-limited. *Why:* standing admin rights are a constant risk — if an always-on admin account is compromised, the attacker inherits full power instantly. Just-in-time access means the privilege simply is not there most of the time. *Threat mitigated:* abuse of standing privileged accounts and escalation to full tenant control. *Validation:* an eligible admin must activate the role through PIM (with MFA) before gaining access.

![Figure 35: Privileged Identity Management quick-start overview](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image35.png)

*Figure 35: Privileged Identity Management quick-start overview.*

![Figure 36: PIM directory roles list including Global Administrator](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image36.png)

*Figure 36: PIM directory roles list including Global Administrator.*

![Figure 37: PIM: adding an eligible assignment (membership)](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image37.png)

*Figure 37: PIM: adding an eligible assignment (membership).*

![Figure 38: PIM: eligible assignment settings (just-in-time activation)](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image38.png)

*Figure 38: PIM: eligible assignment settings (just-in-time activation).*

### 5.3 Identity Protection

*What:* sign-in-risk and user-risk responses — high sign-in risk blocks or challenges, high user risk forces a secure password change. *Why:* Microsoft scores risk from billions of signals, automating a response that would otherwise depend on a human noticing.

**A real fix worth recording.** The legacy Identity Protection risk-policy blade is now **read-only** ahead of its retirement on 1 October 2026, so the enforcement toggle could not be saved. Following Microsoft's current guidance, I rebuilt both the sign-in-risk and user-risk policies inside **Conditional Access**, which provides the same protection with more control over conditions.

![Figure 39: Identity Protection sign-in risk policy now read-only (retiring Oct 2026)](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image39.png)

*Figure 39: Identity Protection sign-in risk policy now read-only (retiring Oct 2026).*

![Figure 40: Conditional Access replacement: Sign-in Risk - High Block](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image40.png)

*Figure 40: Conditional Access replacement: Sign-in Risk - High Block.*

![Figure 41: Conditional Access replacement: User Risk - High Password Change](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image41.png)

*Figure 41: Conditional Access replacement: User Risk - High Password Change.*

![Figure 42: Identity Protection dashboard](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image42.png)

*Figure 42: Identity Protection dashboard.*

### 5.4 Strengthen Authentication Methods

*What:* favoured Microsoft Authenticator and FIDO2 keys and reduced reliance on SMS. *Why:* SMS codes can be intercepted through SIM-swapping; app-based and hardware methods are markedly more resistant. *Threat mitigated:* interception of one-time codes via SIM-swap. *Validation:* a test user registers an authenticator app and signs in using the stronger method.

![Figure 43: Authentication methods policy: Authenticator and FIDO2 favoured over SMS](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image43.png)

*Figure 43: Authentication methods policy: Authenticator and FIDO2 favoured over SMS.*

---

## 6. Phase 4 — Secure the Identity Lifecycle

**Goal.** Demonstrate the joiner, mover, and leaver lifecycle and show that deprovisioning is a security control. I used a single test account, **New Joiner**, to act out all three events. The lifecycle commands target Microsoft Graph; where the PowerShell `Microsoft.Graph` module hit authentication issues on the VM, I completed the same operations through **Microsoft Graph Explorer** (browser-based, identical API underneath) and the Entra admin centre.

### 6.1 Joiner — Provision with Least Privilege

Access is granted through **group membership**, not direct assignment, so it is auditable and reversible. The intended automation (PowerShell) is:

```powershell
Connect-MgGraph -Scopes "User.ReadWrite.All","Group.ReadWrite.All"

$password = @{ ForceChangePasswordNextSignIn = $true; Password = "Project!Onboard123" }
New-MgUser -DisplayName "New Joiner" `
  -UserPrincipalName "new.joiner@universityofroehampton747.onmicrosoft.com" `
  -MailNickname "new.joiner" -AccountEnabled `
  -PasswordProfile $password
```

`Connect-MgGraph` authenticates to Graph with only the scopes needed (least privilege applied to the connection itself). `New-MgUser` creates the account and forces a password change at first sign-in, so the temporary password is never permanent.

![Figure 44: PowerShell New-MgUser attempt to create the joiner](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image44.png)

*Figure 44: PowerShell New-MgUser attempt to create the joiner.*

Getting the module onto the locked-down server took some work (`Install-Module Microsoft.Graph -Scope CurrentUser`, plus TLS 1.2 and an updated NuGet provider).

![Figure 45: Installing the Microsoft.Graph PowerShell module](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image45.png)

*Figure 45: Installing the Microsoft.Graph PowerShell module.*

**The equivalent in Graph Explorer.** The same create uses `POST https://graph.microsoft.com/v1.0/users` with this body:

```json
{
  "displayName": "New Joiner",
  "userPrincipalName": "new.joiner@universityofroehampton747.onmicrosoft.com",
  "mailNickname": "new.joiner",
  "accountEnabled": true,
  "passwordProfile": { "forceChangePasswordNextSignIn": true, "password": "Project!Onboard123" }
}
```

**A real fix worth recording.** The first POST returned **403 Authorization_RequestDenied** — *insufficient privileges*. The fix was to consent to `User.ReadWrite.All` and `Directory.ReadWrite.All` on the **Modify permissions** tab in Graph Explorer (which requires the signed-in account to hold Global Administrator). After consent, the account was created and a `GET` confirmed it.

![Figure 46: Graph Explorer: create-user returns 403 before consent is granted](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image46.png)

*Figure 46: Graph Explorer: create-user returns 403 before consent is granted.*

![Figure 47: Graph Explorer: GET confirms the New Joiner account exists (200 OK)](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image47.png)

*Figure 47: Graph Explorer: GET confirms the New Joiner account exists (200 OK).*

**Validation.** The New Joiner account signs in to Microsoft 365 successfully and is prompted to change the temporary password.

![Figure 48: New Joiner successfully signed in to Microsoft 365](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image48.png)

*Figure 48: New Joiner successfully signed in to Microsoft 365.*

### 6.2 Mover — Adjust Access on Role Change

I simulated a promotion by moving New Joiner out of one access group and into another, then ran an access review to confirm the old permissions were gone. In Graph Explorer the move is a `DELETE` of the old membership and a `POST` of the new one:

```http
DELETE https://graph.microsoft.com/v1.0/groups/{oldGroupId}/members/{userId}/$ref
POST   https://graph.microsoft.com/v1.0/groups/{newGroupId}/members/$ref
Body:  { "@odata.id": "https://graph.microsoft.com/v1.0/directoryObjects/{userId}" }
```

**Why this matters.** Access added on promotion but never removed leads to *privilege creep* — a long-serving employee accumulating rights far beyond their current role. Removing old access is as important as granting new access. The audit log below records exactly these add/remove/update operations against the account.

![Figure 49: Audit logs showing mover group add/remove and update operations](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image49.png)

*Figure 49: Audit logs showing mover group add/remove and update operations.*

### 6.3 Leaver — Secure Deprovisioning

The leaver process is treated as a security event: cut off access immediately, then clean up. The intended PowerShell is:

```powershell
$upn  = "new.joiner@universityofroehampton747.onmicrosoft.com"
$user = Get-MgUser -UserId $upn

# 1. Block sign-in immediately
Update-MgUser -UserId $user.Id -AccountEnabled:$false

# 2. Revoke all active sessions and tokens
Revoke-MgUserSignInSession -UserId $user.Id

# 3. Remove from all groups (iterate membership)
# 4. Later: convert mailbox, reassign data, then delete the account
```

**Why the order matters.** Disabling stops *new* sign-ins; revoking sessions kills *existing* tokens that could otherwise keep working for hours. Together they cut the leaver off instantly. Group removal strips entitlements; deletion comes last, after any data handover. The same early-retirement or planned-departure case is handled identically, with the disable step scheduled for the effective leaving date so access ends exactly when employment does.

In Graph Explorer the urgent steps are a `PATCH` then a `POST`:

```http
PATCH https://graph.microsoft.com/v1.0/users/new.joiner@universityofroehampton747.onmicrosoft.com
Body: { "accountEnabled": false }

POST  https://graph.microsoft.com/v1.0/users/new.joiner@universityofroehampton747.onmicrosoft.com/revokeSignInSessions
```

![Figure 50: Graph Explorer: disabling the leaver account (204 No Content)](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image50.png)

*Figure 50: Graph Explorer: disabling the leaver account (204 No Content).*

![Figure 51: Graph Explorer: revoking all sign-in sessions (200 OK)](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image51.png)

*Figure 51: Graph Explorer: revoking all sign-in sessions (200 OK).*

![Figure 52: Graph Explorer: session revocation confirmed](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image52.png)

*Figure 52: Graph Explorer: session revocation confirmed.*

**Validation.** After deletion, a `GET` on the account returns **404 Not Found** — the deprovisioning is complete.

![Figure 53: Graph Explorer: GET returns 404 after the leaver is deleted](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image53.png)

*Figure 53: Graph Explorer: GET returns 404 after the leaver is deleted.*

**Connecting it back to the stale accounts.** This is exactly the process that *should* have run for `jlegacy` and `boldstaff` the day those staff left: disable, revoke, remove from groups, then delete after retention. Because it did not, they became dormant-but-enabled accounts — the classic breach route Phase 5 is built to catch.

### 6.4 Ongoing Hygiene — Access Reviews

Finally I configured an access review on a privileged group so that membership is re-checked on a recurring schedule rather than once. Access reviews turn identity hygiene into a standing control and are strong evidence of governance for auditors and regulators.

---

## 7. Phase 5 — Detection and Response

**Goal.** Prove I can detect the attacks I understand. I routed identity logs into Microsoft Sentinel, simulated an identity attack chain in my own tenant, and built detections that fire on that simulation.

### 7.1 Connect the Log Sources

I created a Log Analytics workspace (`law-Project-siem`), enabled Microsoft Sentinel on it, and connected the Microsoft Entra ID data connector to bring in **SignInLogs** and **AuditLogs**.

![Figure 54: Adding Microsoft Sentinel to the law-Project-siem Log Analytics workspace](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image54.png)

*Figure 54: Adding Microsoft Sentinel to the law-Project-siem Log Analytics workspace.*

![Figure 55: Sentinel data connectors: Microsoft Entra ID connected](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image55.png)

*Figure 55: Sentinel data connectors: Microsoft Entra ID connected.*

**Confirming events arrive.** Recent sign-ins and audit entries appear in the workspace.

![Figure 56: Entra sign-in logs flowing into the SIEM](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image56.png)

*Figure 56: Entra sign-in logs flowing into the SIEM.*

![Figure 57: Entra audit logs flowing into the SIEM](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image57.png)

*Figure 57: Entra audit logs flowing into the SIEM.*

**Field awareness.** Different sources name the same thing differently. Entra sign-in logs identify the user and source with `userPrincipalName` and `ipAddress`, while Exchange and SharePoint activity logs use `UserId` and `ClientIP`. A correlation rule that joins them must map `userPrincipalName` ↔ `UserId` and `ipAddress` ↔ `ClientIP`, or it silently breaks. Knowing this is a detail that signals real log-analysis experience.

### 7.2 Simulate the Identity Attack Chain

> Performed only inside my own Project tenant.

I recreated a realistic chain from initial pressure to full compromise:

1. **Repeated MFA prompts** against a target test account (MFA fatigue).
2. **A successful sign-in from an unusual location** (the compromise).
3. **Creation of a new account** (a backdoor).
4. **Assignment of a privileged role** to that new account (escalation to control).

### 7.3 Build the Detections

Each detection is a scheduled Sentinel analytics rule written in KQL.

**Detection 1 — MFA fatigue / push bombing** (catches step 1):

```kql
SigninLogs
| where ResultType in ("50074", "500121")   // MFA required / MFA failed
| summarize MfaPrompts = count() by UserPrincipalName, bin(TimeGenerated, 5m)
| where MfaPrompts >= 5
```

![Figure 58: Sentinel analytics rule: MFA fatigue detection (KQL)](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image58.png)

*Figure 58: Sentinel analytics rule: MFA fatigue detection (KQL).*

**Detection 2 — Impossible travel** (catches step 2): compares each successful sign-in's country with the previous one for the same user, flagging two different countries less than an hour apart.

```kql
SigninLogs
| where ResultType == 0
| project TimeGenerated, UserPrincipalName, IPAddress, Country = tostring(LocationDetails.countryOrRegion)
| order by UserPrincipalName, TimeGenerated asc
| serialize
| extend PrevCountry = prev(Country), PrevUser = prev(UserPrincipalName), PrevTime = prev(TimeGenerated)
| where UserPrincipalName == PrevUser and Country != PrevCountry
| where datetime_diff('minute', TimeGenerated, PrevTime) < 60
```

![Figure 59: Sentinel analytics rule: impossible travel detection (KQL)](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/bff4ed3641371c5945beab34af8e9e4c9cd5d7cb/M365%20resources/image59.png)

*Figure 59: Sentinel analytics rule: impossible travel detection (KQL).*

**Detection 3 — Backdoor account creation** (catches step 3):

```kql
AuditLogs
| where OperationName == "Add user"
| project TimeGenerated, Actor = tostring(InitiatedBy.user.userPrincipalName),
          NewUser = tostring(TargetResources[0].userPrincipalName)
```

**Detection 4 — Unauthorised privileged role assignment** (catches step 4):

```kql
AuditLogs
| where OperationName == "Add member to role"
| project TimeGenerated, Actor = tostring(InitiatedBy.user.userPrincipalName),
          TargetUser = tostring(TargetResources[0].userPrincipalName),
          Role = tostring(TargetResources[0].displayName)
```

**Validation.** All four rules are active in Sentinel, and each simulated action produced its corresponding alert. A detection that does not fire on your own simulation is not yet a control.

![Figure 60: Sentinel analytics: all four identity detections active](https://github.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/blob/c012d66fdd6ddfa875ac62a91df7bc889619d309/M365%20resources/image60.png)

*Figure 60: Sentinel analytics: all four identity detections active.*

### 7.4 Incident Response Runbook

A short runbook people will actually follow beats a long one nobody reads. Each alert maps to a first action and a follow-up.

| Alert | Immediate action | Then |
|---|---|---|
| MFA fatigue | Block sign-in for the targeted user; revoke sessions | Confirm whether any prompt was approved; reset credentials if unsure |
| Impossible travel | Revoke sessions; require re-auth with MFA | Review sign-in logs for what the session accessed |
| Backdoor account created | Disable the new account | Identify who created it; if unapproved, treat as compromise |
| Privileged role assigned | Remove the role; revoke the account's sessions | Verify against change records; if unauthorised, full investigation |

Mapped to the classic lifecycle: **Detect** (alert triaged) -> **Contain** (disable, revoke) -> **Investigate** (sign-in and audit timeline) -> **Eradicate** (remove backdoors, reverse roles) -> **Recover** (reset credentials, re-register MFA) -> **Learn** (record and tighten the control).

---

## 8. Risk Posture, Before and After

Impact is framed in business and regulatory terms, since identity compromise in a real estate carries data-protection consequences under regimes such as the UK GDPR.

| Risk | Before | Residual risk after |
|---|---|---|
| Stale and orphaned accounts | Dormant accounts present and synced, unmonitored | Excluded from sync, disabled on departure, reviewed periodically |
| Standing administrative rights | Permanent Global Admin accounts | Just-in-time activation with MFA and justification |
| Password-only sign-in | Single factor, phishable | MFA enforced, legacy auth blocked |
| Undetected identity compromise | No alerting on identity events | Attack chain detected and runbook in place |
| Unmanaged device access to admin | Any device could authenticate | Compliant device required for privileged roles |

---

## 9. Limitations and Honest Assessment

This Project demonstrates the correct configuration and validation of enterprise identity controls at small scale. It does not, by itself, prove operation under production load or the organisational change management a real rollout demands.

Several controls are layers rather than complete defences. Location-based rules can be defeated by an attacker using infrastructure inside the expected region. Stating these limits plainly is part of sound security engineering — a control presented without its weaknesses is a control not properly understood.

---

## 10. Mapping to Recognised Frameworks

| Control implemented | Framework reference |
|---|---|
| MFA and Conditional Access | Identity and access management controls in CIS and NIST guidance |
| Just-in-time privileged access | Least-privilege and privileged access management principles |
| Secure deprovisioning and access reviews | Account management and access-control lifecycle controls |
| Identity attack detection | MITRE ATT&CK techniques for credential access and privilege escalation |

---

## 11. Conclusion

The completed Project provides a working hybrid identity environment secured to a professional standard, with identity controls, a secure lifecycle, and working detections that I built and validated myself. It shows I can protect identity and access across on-premise and cloud, design controls as deliberate risk decisions, and detect the threats I have studied. The build is intentionally measured: it shows capability through evidence rather than assertion, and it is honest about what it does and does not prove.

---

## Appendix A — Evidence Checklist

| # | Evidence | Figure |
|---|---|---|
| 1 | Azure portal: Windows Server VM and networking resources provisioned | image1.png |
| 2 | VM Connect blade: downloading the RDP file for the domain controller | image2.png |
| 3 | Downloaded .rdp file ready to launch | image3.png |
| 4 | Windows Security credential prompt when connecting over RDP | image4.png |
| 5 | RDP certificate trust prompt for the Project server | image5.png |
| 6 | Windows Server desktop after a successful RDP connection | image6.png |
| 7 | Installing the AD DS role (installation in progress) | image7.png |
| 8 | Install-WindowsFeature completes with Success: True | image8.png |
| 9 | Install-ADDSForest running with the DSRM password prompt | image9.png |
| 10 | Forest promotion validating prerequisites before reboot | image10.png |
| 11 | Get-ADDomain and service checks confirm the domain controller is live | image11.png |
| 12 | Network adapter and DNS verification (interface index troubleshooting) | image12.png |
| 13 | Active Directory Users and Computers open on corp.freedomProject.local | image13.png |
| 14 | Creating the Organisational Unit structure with PowerShell | image14.png |
| 15 | Bulk-creating eighteen test user accounts | image15.png |
| 16 | Creating two administrative accounts and adding them to Domain Admins | image16.png |
| 17 | Creating the two deliberately stale (dormant) accounts | image17.png |
| 18 | Verifying all created users appear in the Departments OU | image18.png |
| 19 | Validating admin membership, total user count, and OU list | image19.png |
| 20 | Entra Connect: sign-in / UPN suffix configuration screen | image20.png |
| 21 | Entra Connect: configuration complete and first sync initiated | image21.png |
| 22 | Entra Connect: full wizard completed (Connect, Filtering, SSO, Configure) | image22.png |
| 23 | Synced on-premise users now visible in the Entra admin centre | image23.png |
| 24 | A synced user signed into Microsoft 365 (Welcome back, Test User 2) | image24.png |
| 25 | Microsoft Authenticator registration via QR code | image25.png |
| 26 | Conditional Access Policy 1: Require MFA for all users (report-only) | image26.png |
| 27 | Conditional Access policy list after creating the MFA policy | image27.png |
| 28 | Conditional Access Policy 2: targeting legacy client apps | image28.png |
| 29 | Conditional Access Policy 2: Block access grant control | image29.png |
| 30 | Conditional Access Policy 3: selecting the administrative accounts | image30.png |
| 31 | Conditional Access Policy 3: require a compliant device | image31.png |
| 32 | Conditional Access policy list: three policies created | image32.png |
| 33 | Conditional Access named location: Allow - United Kingdom | image33.png |
| 34 | Conditional Access Policy 4: restrict sign-in by location | image34.png |
| 35 | Privileged Identity Management quick-start overview | image35.png |
| 36 | PIM directory roles list including Global Administrator | image36.png |
| 37 | PIM: adding an eligible assignment (membership) | image37.png |
| 38 | PIM: eligible assignment settings (just-in-time activation) | image38.png |
| 39 | Identity Protection sign-in risk policy now read-only (retiring Oct 2026) | image39.png |
| 40 | Conditional Access replacement: Sign-in Risk - High Block | image40.png |
| 41 | Conditional Access replacement: User Risk - High Password Change | image41.png |
| 42 | Identity Protection dashboard | image42.png |
| 43 | Authentication methods policy: Authenticator and FIDO2 favoured over SMS | image43.png |
| 44 | PowerShell New-MgUser attempt to create the joiner | image44.png |
| 45 | Installing the Microsoft.Graph PowerShell module | image45.png |
| 46 | Graph Explorer: create-user returns 403 before consent is granted | image46.png |
| 47 | Graph Explorer: GET confirms the New Joiner account exists (200 OK) | image47.png |
| 48 | New Joiner successfully signed in to Microsoft 365 | image48.png |
| 49 | Audit logs showing mover group add/remove and update operations | image49.png |
| 50 | Graph Explorer: disabling the leaver account (204 No Content) | image50.png |
| 51 | Graph Explorer: revoking all sign-in sessions (200 OK) | image51.png |
| 52 | Graph Explorer: session revocation confirmed | image52.png |
| 53 | Graph Explorer: GET returns 404 after the leaver is deleted | image53.png |
| 54 | Adding Microsoft Sentinel to the law-Project-siem Log Analytics workspace | image54.png |
| 55 | Sentinel data connectors: Microsoft Entra ID connected | image55.png |
| 56 | Entra sign-in logs flowing into the SIEM | image56.png |
| 57 | Entra audit logs flowing into the SIEM | image57.png |
| 58 | Sentinel analytics rule: MFA fatigue detection (KQL) | image58.png |
| 59 | Sentinel analytics rule: impossible travel detection (KQL) | image59.png |
| 60 | Sentinel analytics: all four identity detections active | image60.png |

---

## Appendix B — Command and Query Reference

| Task | Tool | Command / Endpoint |
|---|---|---|
| Install AD DS role | PowerShell (VM) | `Install-WindowsFeature AD-Domain-Services -IncludeManagementTools` |
| Promote domain controller | PowerShell (VM) | `Install-ADDSForest -DomainName "corp.freedomProject.local" -InstallDns -Force` |
| Verify domain | PowerShell (VM) | `Get-ADDomain` ; `Get-Service ADWS,NTDS,DNS` |
| Create OUs | PowerShell (VM) | `New-ADOrganizationalUnit` |
| Bulk-create users | PowerShell (VM) | `1..18 | ForEach-Object { New-ADUser ... }` |
| Add admins to group | PowerShell (VM) | `Add-ADGroupMember -Identity "Domain Admins"` |
| Connect to Graph | PowerShell (VM) | `Connect-MgGraph -Scopes "User.ReadWrite.All","Group.ReadWrite.All"` |
| Create joiner | Graph Explorer (PC) | `POST /v1.0/users` |
| Disable leaver | Graph Explorer (PC) | `PATCH /v1.0/users/{upn}` with `accountEnabled:false` |
| Revoke sessions | Graph Explorer (PC) | `POST /v1.0/users/{upn}/revokeSignInSessions` |
| Verify / confirm deletion | Graph Explorer (PC) | `GET /v1.0/users/{upn}` |
| MFA fatigue detection | Sentinel KQL | `SigninLogs | where ResultType in ("50074","500121")` |
| Impossible travel detection | Sentinel KQL | `SigninLogs | ... prev(Country) ... datetime_diff < 60` |
| Backdoor account detection | Sentinel KQL | `AuditLogs | where OperationName == "Add user"` |
| Privilege escalation detection | Sentinel KQL | `AuditLogs | where OperationName == "Add member to role"` |

---

*End of report. All screenshots are from the author's own Project tenant (`universityofroehampton747.onmicrosoft.com`) and domain controller (`corp.freedomProject.local`).*
