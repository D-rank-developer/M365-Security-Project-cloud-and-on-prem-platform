# Securing Hybrid Identity: An End-to-End Microsoft 365 and Entra ID Security Implementation

A hands-on security engineering project that builds, hardens, manages, and monitors a complete hybrid identity environment, from an on-premises Active Directory domain controller all the way to SIEM detection rules and incident response.

## Architecture Overview

![Hybrid identity architecture: on-premises Active Directory synced to Microsoft Entra ID, secured by Conditional Access, PIM, Identity Protection and lifecycle management, with logs exported to SIEM detection and an incident response runbook](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/704d15f46d5964ef8f7a8f2554a5cdcb9e5aa33c/M365%20resources/Gemini_Generated_Image_%20%281%29.png)

*On-premises Active Directory acts as the authoritative identity source. Entra Connect syncs scoped OUs to Microsoft Entra ID using password hash sync. Entra ID enforces Conditional Access, MFA, PIM, Identity Protection, and the joiner-mover-leaver lifecycle, securing sign-in to Microsoft 365 while exporting sign-in and audit logs to the SIEM for detection and incident response.*

![Layered flow: Windows Server domain controller and user accounts sync via Entra Connect to Microsoft Entra ID controls, which secure Microsoft 365 services and feed SIEM detection leading to incident response](https://raw.githubusercontent.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/704d15f46d5964ef8f7a8f2554a5cdcb9e5aa33c/M365%20resources/Gemini_Generated_Image_%20%282%29.png)

*The same environment viewed as a layered data flow: identities originate on-premises, security controls are applied in the cloud, and everything downstream is monitored.*

## What Was Built

| Phase | Deliverable |
|-------|-------------|
| 1. Foundation | Windows Server 2022 domain controller on Azure, `corp.freedomlab.local` forest, DNS, OU structure, 20+ user accounts created via PowerShell |
| 2. Synchronisation | Microsoft Entra Connect with password hash sync, OU-scoped filtering, and Seamless SSO |
| 3. Hardening | Four Conditional Access policies (MFA, legacy auth block, compliant devices for admins, location-based), Privileged Identity Management, risk-based sign-in and user policies |
| 4. Lifecycle | Full joiner-mover-leaver lifecycle implemented through Microsoft Graph (user provisioning, group moves, access reviews, secure offboarding with session revocation) |
| 5. Detection | Microsoft Sentinel with Entra ID connectors and four custom KQL analytics rules: MFA fatigue, impossible travel, backdoor account creation, and privilege escalation, plus an incident response runbook |

## Skills Demonstrated

Active Directory Domain Services, Microsoft Entra ID, Entra Connect, Conditional Access, PIM, Identity Protection, Microsoft Graph API, PowerShell, KQL, Microsoft Sentinel, Zero Trust architecture, identity lifecycle management, and detection engineering.

## Full Report

The complete step-by-step report, with every command, Graph request, KQL query, and 60 screenshots of evidence, is available in this repository:

- [Hybrid_Identity_Security_Lab_Report.md]([./Hybrid_Identity_Security_Lab_Report.md](https://github.com/D-rank-developer/M365-Security-Project-cloud-and-on-prem-platform/blob/main/M365%20security%20posture.md))

## Author

**Freedom (Dumanyie Dornubari Chamberlain)**
MSc Cyber Security, University of Roehampton, London
