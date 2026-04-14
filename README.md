# IT Support Home Lab - Enterprise Environment from Scratch

A fully functional enterprise IT environment built on VirtualBox, simulating a 25-person company's infrastructure. Every component was built with intentional design decisions, tested through deliberate breakage, and documented with the troubleshooting process that matters more than the final config.

## Why This Exists

I hold the CompTIA trifecta (A+, Network+, Security+), Cloud+, AWS CCP, AZ-900, LPI Linux Essentials, ITIL 4 Foundation, and I'm completing a B.S. in Cloud Computing with the AZ-104 in progress. The certifications prove I can pass exams. This lab proves I can build, break, diagnose, and fix real infrastructure.

Before touching a single VM, I spent time building a deep understanding of the networking fundamentals underneath everything - how DHCP bootstraps a device onto a network, how DNS resolution chains from local cache through recursive servers to authoritative nameservers, how Layer 2 and Layer 3 work together at every hop, and why a gateway is a separate device from a domain controller. I questioned every default setting until I understood the reason behind it. That foundation is what made the troubleshooting in this lab possible.

## Lab Architecture

| VM | OS | Role | IP |
|---|---|---|---|
| DC01 | Windows Server 2022 | Primary DC, DNS, DHCP | 10.0.0.10 |
| DC02 | Windows Server 2022 | Secondary DC, DNS replication | 10.0.0.11 |
| SRV-FILE | Windows Server 2022 | File server, print server | 10.0.0.20 |
| WS-PC01 | Windows 10 Pro | Domain-joined workstation | DHCP (10.0.0.100) |
| pfSense | pfSense CE | Firewall, router, VPN, VLANs | 10.0.0.1 |

**Network:** 10.0.0.0/24 - DHCP range 10.0.0.100-200, static assignments below .100. Gateway at 10.0.0.1 (pfSense). DNS forwarders to 1.1.1.1 and 8.8.8.8.

## What I Built and What Broke

### Phase 1: Active Directory & Identity Management

Built a two-DC domain (`lab.local`) with full replication. Designed an enterprise OU structure separating departments, workstations, servers, security groups, service accounts, disabled accounts, and a staging OU for new hires. Deployed 25 users across five departments via PowerShell bulk import. Created eight security groups implementing role-based access control - department groups separate from role-based groups because not everyone in a department needs the same access.

**Design decisions that matter:**
- VPN access is a role-based group, not department-based, because only hybrid/remote/traveling employees need it regardless of department. That's least privilege in practice.
- USB restriction GPO applied to Engineering, Finance, HR, and IT but not Sales — those departments handle PII, financial records, proprietary code, and admin credentials. Sales data lives in cloud CRMs, making the risk profile different.
- Drive mapping uses a single GPO at the Departments OU with item-level targeting by security group, rather than separate GPOs per department.

**Proactively identified a failover gap:** After setting up DC02, I realized DHCP was only handing clients a single DNS server (10.0.0.10). If DC01 went down, every client would lose name resolution even though DC02 had a complete copy of all DNS zones. Added 10.0.0.11 as secondary DNS in DHCP scope options before any failure occurred.

**Discovered that password/lockout policies only work from the Default Domain Policy.** Created a custom GPO that appeared to apply correctly in `gpresult /r` but didn't enforce account lockout during testing. Moved settings to Default Domain Policy and confirmed enforcement.

→ [Full Phase 1 Build Log](active-directory/BUILD-LOG.md)

### The Replication Failure - My Best Troubleshooting Story

During breakage exercises, I changed DC01's IP without updating downstream dependencies. Observed cascading failures, reverted the change, verified replication showed zero failures. Looked fixed.

**It wasn't.** The next day, during the failover exercise, DC02 couldn't authenticate users — passwords and computer accounts from the previous 24 hours had never replicated. `repadmin /replsummary` revealed 100% failure rate since the IP change. The `nslookup` showing "Server: Unknown" that I'd dismissed as cosmetic was actually a symptom of incomplete DNS recovery.

**Fix:** `ipconfig /registerdns` on DC01 → `repadmin /syncall /AeD` → `ipconfig /flushdns` on DC02 → verified zero failures → successful failover to DC02 confirmed with `nltest`.

A "fixed" problem left hidden damage that surfaced 24 hours later in an unrelated test. The root cause of the failover failure wasn't the failover configuration — it was residual DNS damage from a change made the previous day.

→ [Full Breakage Exercise Log](breakage-exercises-on-prem/BREAKAGE-LOG.md)

### Phase 2: Microsoft 365 Administration

Built a full M365 tenant (MeridianLabSolutions.onmicrosoft.com) with department Teams, security groups, Exchange Online shared mailbox, mail flow rules, and four Conditional Access policies. Created a break-glass emergency admin account excluded from all CA policies.

**Investigated a blocked sign-in through audit logs before re-enabling it** — checked sign-in logs (2 failed attempts, not enough for auto-lockout), then audit logs (found manual "UpdateUser" action confirming it was an admin-initiated block, not a brute force attempt). The distinction determines whether re-enabling the account is the right response or a security risk.

Understood hybrid identity architecture - Entra Connect's one-directional sync from on-prem to cloud, the 30-minute sync gap that makes manual intervention necessary for terminations, and the risk when the sync server goes down.

→ [Full Phase 2 Build Log](microsoft-365/BUILD-LOG.md)

### Phase 3: Networking - pfSense, VLANs, VPN & Firewall Rules

Deployed pfSense as the network gateway/firewall with WAN (internet) and LAN (lab network) interfaces. Built VLAN 20 (10.0.20.0/24) as a guest network with firewall rules enforcing isolation from the corporate 10.0.0.0/24 network. Configured a full OpenVPN server with CA, server certificates, user certificates, tunnel network, and firewall rules.

**Key insight from VLAN work:** VLANs create separate broadcast domains but don't enforce isolation by themselves - the firewall rules are what actually block traffic between networks. Without the block rule, pfSense would happily route between VLAN 20 and the corporate network. Same principle applies to VPN: the tunnel network is just another subnet pfSense can route to, controlled by firewall policy.

**Firewall rule ordering:** DNS allow rule must be above the corporate block rule — otherwise DNS queries from the guest network get killed before they're evaluated, leaving guests with "internet access" they can't actually use because nothing resolves.

**VPN testing limitation:** OpenVPN server built and running correctly (verified with `sockstat` and pfSense status). Connection testing from host machine was blocked by VirtualBox's NAT adapter not supporting UDP port forwarding reliably, and the Wi-Fi driver not supporting bridged mode. Diagnosed through `tcpdump` packet capture showing no inbound traffic reaching pfSense's WAN despite port forwarding rules. Documented as a hypervisor limitation, not a configuration issue.

→ [Full Phase 3 Build Log](networking/BUILD-LOG.md)

### Phase 4: Hybrid Identity with Entra Connect

Connected on-prem AD to an M365 E5 tenant via Entra Connect with password hash synchronization and SSO. Users are created and managed entirely in on-prem AD, sync to Entra ID automatically every 30 minutes, and receive M365 licenses through group-based licensing on a synced security group. The complete onboarding pipeline - AD account creation → group membership → delta sync → Entra ID user → auto-licensed → mailbox provisioned → working email — runs without touching the cloud.

**The password reset spiral** was the most valuable troubleshooting experience in this phase. Reset a user's password from the M365 admin center, which wrote back to AD via password writeback. The cloud temp password got past the initial login screen but failed the "change password" prompt because it was validating against the synced AD hash. Multiple resets from both sides, forced syncs, blocked/unblocked sign-in - the password just wouldn't take. Turned out to be propagation timing: password hash sync needs several minutes beyond the delta sync completion to apply on the authentication side. The lesson for desktop support: always reset from on-prem AD in a hybrid environment, sync, and tell the user to wait 5 minutes.

**Breakage exercises discovered:** Sync scheduler can be disabled while manual syncs still work (silent failure), UPN changes to .local get silently ignored by Entra ID (no error but identity becomes mismatched), moving a user outside the synced OU scope instantly soft-deletes them from Entra ID (and restoring from the cloud side while still in the wrong OU creates a conflict - always fix from the source of truth).

→ [Full Phase 4 Build Log](hybrid-identity/BUILD-LOG.md)

### Phase 5: Exchange Online Administration

Configured shared mailboxes with Full Access and Send As permissions, distribution lists (including a dynamic All-Staff list with restricted sending), and mail flow rules for external email tagging and confidentiality disclaimers. Practiced message trace to diagnose delivery issues and identify which transport rules fired during email routing.

**Discovered that traditional Outlook troubleshooting doesn't apply on modern managed devices.** Followed a standard runbook (clear Credential Manager, rebuild OST, repair profile) on an Intune-managed Entra-joined device running New Outlook - every step failed because the architecture is fundamentally different. Credential Manager was empty (auth uses WAM/PRT, not cached passwords). No OST file exists (New Outlook is a web app wrapper). No profile repair or creation options. The entire traditional runbook collapses into: check OWA for isolation → close/reopen app → Repair → Reset. 

**Password change token persistence:** Changed a user's password and revoked sessions. OWA kicked out immediately, but desktop Outlook kept working for 10+ minutes because the device-level PRT was still valid. Only broke when the app was closed and reopened, forcing a token re-evaluation.

→ [Full Phase 5 Build Log](exchange-online/BUILD-LOG.md)

### Phase 6: Teams & SharePoint Online Administration

Created department Teams with guest access, private channels, and custom meeting policies using group-based RBAC (global default restricts recording, SG-Managers override allows it). Configured SharePoint permission inheritance — broke inheritance on a confidential folder so only SG-Managers could access it, verified regular users couldn't even see the folder.

**Teams troubleshooting:** Private channel creation failed with a vague error. Investigated policies, ownership, and ultimately discovered channel names must be unique across the entire tenant, not just within the team. Renamed and resolved.

**SharePoint permissions are the #1 SharePoint support ticket:** "I can't access this folder" almost always means the user isn't in the right group or inheritance was broken. Understanding the inheritance model and how to check/fix permissions is core desktop support work.

→ [Full Phase 6 Build Log](teams-sharepoint/BUILD-LOG.md)

### Phase 7: Intune & Endpoint Management Deep Dive

Re-enrolled a device through Autopilot into the full hybrid environment. Created a dynamic device group, compliance policy (BitLocker, firewall, antivirus, OS version), BitLocker configuration profile with recovery key escrow to Entra ID, device restrictions, and deployed M365 Apps as a required package.

**The complete pipeline working end to end:** User created in on-prem AD → synced to Entra ID via Entra Connect → auto-licensed through group-based licensing → mailbox auto-provisioned in Exchange Online → device enrolled through Autopilot → compliance policies evaluated → BitLocker encrypting with key in Entra → M365 Apps installed → Outlook auto-configured with user's mailbox. Zero manual configuration on the device. User powers on, signs in, and is working within an hour.

→ [Full Phase 7 Build Log](intune-endpoint-management/BUILD-LOG.md)

## Breakage Exercises Summary

| # | What I Broke | Key Finding |
|---|---|---|
| 1 | Changed DC01's IP without updating DNS | Cascading failure across DHCP, replication, DNS - plus hidden damage that survived the revert |
| 2 | Disabled a user account | Cached credentials keep sessions alive; disable ≠ immediate logoff |
| 3 | Locked out an account (5 wrong passwords) | Lockout policies must be in Default Domain Policy; different error message from disabled account |
| 4 | Moved user to wrong OU | Lost USB restriction but retained SG-IT membership - security gap requiring both OU and group updates |
| 6 | Removed user from department group | All associated permissions vanished instantly; RBAC working as designed |
| 7 | Shut down DC01, tested failover | Exposed hidden replication failure from Exercise 1; after fix, successful DC02 failover |
| NET-001 | Set wrong DNS on client | Client couldn't browse but could ping by IP; required DNS fix + cache flush + lease renewal |
| HYB-001 | Disabled Entra Connect sync scheduler | Manual syncs still work - silent failure; automatic cycle stops without any visible error |
| HYB-002 | Changed UPN back to .local | Entra ID silently ignored the change; no error but identity mismatched between on-prem and cloud |
| HYB-003 | Moved user outside synced OU scope | User soft-deleted from Entra ID instantly; restoring from cloud side creates conflicts - must fix from AD |
| HYB-004 | Password reset from cloud side in hybrid | Writeback + forced password change flag created validation loop; propagation timing was the actual issue |
| EXO-001 | Removed user from license group | Immediate loss of all M365 services; mailbox recovered intact after re-adding (30-day soft delete) |
| EXO-002 | Mail flow rule blocking all external email | Message trace identified the exact rule; simulates misconfigured transport rule in production |
| EXO-003 | Send As without Full Access | User could send from shared mailbox but couldn't see its inbox - independent permissions |
| EXO-004 | Intune policy removal didn't undo restriction | Deleted a Control Panel block policy but restriction persisted until device synced |
| EXO-005 | Password change on managed device | OWA killed immediately but desktop Outlook rode the PRT for 10+ minutes; token-based auth ≠ credential-based |

## Network Troubleshooting Methodology

This framework was used instinctively throughout the lab before it was formally documented:

| Step | Check | Tools | If It Fails |
|---|---|---|---|
| 1. Physical | Cable/NIC enabled? | Device Manager, link lights | Reseat cable, enable adapter |
| 2. IP Config | Valid IP? (169.254 = DHCP failure) | `ipconfig /all` | `ipconfig /release`, `/renew` |
| 3. Gateway | Can you reach the default gateway? | `ping 10.0.0.1` | Problem between client and gateway |
| 4. Remote IP | Can you reach the internet by IP? | `ping 8.8.8.8` | Routing, firewall, or ISP issue |
| 5. DNS | Can you resolve names? | `nslookup google.com` | DNS server down or misconfigured |
| 6. Service | Can you reach the specific service? | `Test-NetConnection -Port` | Service down or port blocked |

## Repository Structure

```
IT-Support-HomeLab/
├── README.md
├── active-directory/          ← DC setup, OU structure, users, groups, GPOs
│   └── BUILD-LOG.md
├── breakage-exercises-on-prem/← Deliberate breakage + networking exercises
│   └── BREAKAGE-LOG.md
├── microsoft-365/             ← Initial tenant config, Exchange, Conditional Access
│   └── BUILD-LOG.md
├── networking/                ← pfSense, VLANs, VPN, firewall rules
│   └── BUILD-LOG.md
├── hybrid-identity/           ← Entra Connect, sync config, group-based licensing
│   └── BUILD-LOG.md
├── exchange-online/           ← Shared mailboxes, mail flow, message trace
│   └── BUILD-LOG.md
├── teams-sharepoint/          ← Teams admin, SharePoint permissions, OneDrive
│   └── BUILD-LOG.md
├── intune-endpoint-management/← Compliance, config profiles, app deployment, Autopilot
│   └── BUILD-LOG.md
├── user-lifecycle/            ← Onboarding, offboarding, department transfer procedures
├── m365-capstone/             ← Tuesday morning simulation

```

## Tools & Technologies

Windows Server 2022, Windows 10/11 Pro, Active Directory Domain Services, DNS, DHCP, Group Policy, PowerShell, VirtualBox, pfSense, OpenVPN, Microsoft 365 (E5), Exchange Online, Entra ID, Entra Connect, Conditional Access, Microsoft Intune, Windows Autopilot, Message Trace, WAM/PRT authentication

## Current Status

- Phase 1 (Active Directory & Identity Management) — **Complete**
- Phase 2 (Microsoft 365 Administration) — **Complete**
- Phase 3 (Networking — pfSense, VLANs, VPN) — **Complete**
- Phase 4 (Hybrid Identity with Entra Connect) — **Complete**
- Phase 5 (Exchange Online Administration) — **Complete**
- Phase 6 (Teams & SharePoint Online) — **Complete**
- Phase 7 (Intune & Endpoint Management) — **Complete**
- Phase 8+ (Conditional Access, User Lifecycle, Capstone) — In Progress