# IT Support Engineer — Home Lab

A hands-on lab environment mirroring real enterprise IT infrastructure, built from scratch on VirtualBox.

## Lab Architecture

| VM | OS | Role | IP |
|---|---|---|---|
| DC01 | Windows Server 2022 | Primary DC, DNS, DHCP | 10.0.0.10 |
| DC02 | Windows Server 2022 | Secondary DC, DNS Replication | 10.0.0.11 |
| SRV-FILE | Windows Server 2022 | File Server, Print Server | 10.0.0.20 |
| pfSense | pfSense CE | Firewall, Router, VPN, VLANs | 10.0.0.1 |
| SRV-TICKET | Ubuntu Server | osTicket Ticketing System | 10.0.0.30 |
| WS-PC01 | Windows 10/11 Pro | Domain-Joined Workstation | DHCP |
| WS-PC02 | Windows 10/11 Pro | Domain-Joined Workstation | DHCP |
| SRV-LINUX | Ubuntu Desktop | Linux Troubleshooting, SSH | DHCP |

**Network:** 10.0.0.0/24 · NAT Network (LabNet) · Domain: lab.local

## What's Been Built So Far

### Phase 1: Active Directory & Identity Management

**Infrastructure (DC01 & DC02)**
- Deployed primary domain controller (DC01) with static IP 10.0.0.10
- Configured DNS with forward/reverse lookup zones, scavenging (7-day), and forwarders (8.8.8.8, 1.1.1.1)
- Built DHCP scope (10.0.0.100–200) with 8-hour lease, gateway 10.0.0.1, DNS 10.0.0.10
- Deployed secondary domain controller (DC02) at 10.0.0.11 for redundancy
- Verified AD replication across both DCs with repadmin — zero failures

**DNS Failover Fix (Self-Identified)**
- Discovered that all clients only had DC01 (10.0.0.10) as their DNS server
- If DC01 went down, every client would lose DNS resolution despite DC02 being available
- Added DC02 (10.0.0.11) as secondary DNS server in DHCP scope options
- Clients now fail over to DC02 automatically on lease renewal

**OU Structure**
```
lab.local
├── Corp
│   ├── Departments (IT, HR, Finance, Sales, Engineering)
│   ├── Workstations (Desktops, Laptops)
│   ├── Servers
│   ├── Security Groups
│   └── Service Accounts
├── Disabled Accounts
└── Staging (for new hires)
```

**User Management**
- Created 25+ domain users across 5 departments via PowerShell bulk import from CSV
- Each user placed in correct department OU with ChangePasswordAtLogon enabled

**Security Groups (RBAC)**
- Department groups: SG-IT, SG-HR, SG-Finance, SG-Sales, SG-Engineering
- Role-based groups: SG-VPN-Users, SG-Remote-Desktop-Users, SG-Finance-Share-ReadWrite
- All permissions assigned through groups, never individual users
- Department group membership populated via PowerShell based on OU location

**Group Policy Objects**
| GPO | Linked To | Key Settings |
|---|---|---|
| Password Policy | Domain (lab.local) | 12-char min, complexity on, 90-day expiry, lockout after 5 attempts (30 min) |
| Workstation Security | Workstations OU | Firewall on, RDP enabled, auto updates (scheduled 3AM), sleep timeout 600s |
| Drive Mapping | Departments OU | Maps department shares with item-level targeting by security group |
| USB Restriction | Engineering, Finance, HR, IT OUs | Deny all removable storage access — Sales excluded (lower data sensitivity) |

## Certifications
- CompTIA A+, Network+, Security+, Cloud+
- AWS Certified Cloud Practitioner (CLF-C02)
- AZ-900 Azure Fundamentals
- LPI Linux Essentials
- ITIL 4 Foundation

## Tools & Technologies
Windows Server 2022 · Active Directory · DNS · DHCP · Group Policy · PowerShell · VirtualBox · pfSense · Ubuntu · Microsoft 365 · osTicket