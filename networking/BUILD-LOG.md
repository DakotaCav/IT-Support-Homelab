# Networking — Build Log

Phase 3: pfSense deployment as the network gateway/firewall, VLAN segmentation for guest network isolation, OpenVPN server configuration, and firewall rule design. This phase connected the networking fundamentals studied before the lab to hands-on implementation.

---

## pfSense Deployment

### Setup

Deployed pfSense CE on a VirtualBox VM with two network adapters:
- **Adapter 1 (NAT):** WAN interface — internet-facing, received 10.0.2.15 from VirtualBox NAT DHCP
- **Adapter 2 (Internal Network):** LAN interface — lab network at 10.0.0.1/24

pfSense sits between the lab network and the internet. Every device on 10.0.0.0/24 has its default gateway set to 10.0.0.1 (configured in DC01's DHCP scope back in Phase 1). All internet-bound traffic flows through pfSense, which handles routing, NAT translation, and firewall enforcement.

**DHCP disabled on pfSense LAN** — DC01 handles DHCP for the corporate network. Running two DHCP servers on the same subnet causes IP assignment conflicts, the same problem identified during Phase 1 setup.

### Installation Notes

During install, disabled VLAN tagging at the console prompt — VLANs are configured later through the web interface where there's more control. Set LAN IP to 10.0.0.1/24 as a static assignment.

pfSense warned against using `.local` as the domain suffix due to mDNS conflicts. The `.local` TLD is reserved for Multicast DNS (automatic device discovery), and using it for Active Directory can cause intermittent name resolution failures. In production, Microsoft recommends a domain you actually own. For the lab, `.local` works fine since it was already established in Phase 1.

### Troubleshooting: Couldn't Reach Web Interface

After installation, attempted to access the pfSense web interface at https://10.0.0.1 from DC01. Page wouldn't load.

**Diagnostic process:**
1. `ping 10.0.0.1` — **Success.** Layer 3 connectivity confirmed, pfSense is reachable.
2. `Test-NetConnection 10.0.0.1 -Port 443` — **Failed.** Port 443 not responding.
3. `Test-NetConnection 10.0.0.1 -Port 80` — **Failed.** Port 80 not responding.
4. Researched `pfctl -d` to temporarily disable pfSense's firewall to test if it was blocking web access.
5. **Caught the real issue:** DC01 was still on VirtualBox's NAT Network adapter, not the Internal Network where pfSense's LAN interface lived. They were on completely different virtual networks — the ping succeeded because VirtualBox NAT can route to Internal Network addresses, but the web service ports weren't accessible across that boundary.

**Fix:** Switched all workstations and servers to the Internal Network adapter matching pfSense's LAN. Web interface loaded immediately.

**Lesson:** The symptom looked like a port/firewall issue, but the root cause was a network topology mismatch. Followed the troubleshooting methodology (connectivity → ports → service) but the answer was at a layer below all of that — the devices weren't on the same network segment.

### Connectivity Verification

After switching all VMs to Internal Network, verified the full network path from WS-PC01:

- `ping 10.0.0.1` — pfSense reachable ✓
- `ping 8.8.8.8` — Internet reachable by IP ✓
- `nslookup google.com` — External DNS resolution working ✓
- `nslookup dc01.lab.local` — Internal DNS resolution working ✓

Full chain operational: workstation → pfSense (gateway/NAT) → internet, with DNS flowing through DC01 → pfSense → external forwarders.

---

## VLAN 20 — Guest Network Segmentation

### Concept

Created VLAN 20 (10.0.20.0/24) as a guest network, simulating corporate vs. guest Wi-Fi isolation. The corporate network (10.0.0.0/24) contains domain controllers, file shares, and workstations. The guest network provides internet access only — no visibility into corporate resources.

pfSense acts as the gateway for both networks: 10.0.0.1 for corporate, 10.0.20.1 for guests.

### Key Insight: VLANs Don't Enforce Isolation — Firewalls Do

VLANs create separate broadcast domains (separate "streets"), but a router sitting on both networks can freely pass traffic between them. pfSense has a foot on both 10.0.0.0/24 and 10.0.20.0/24 — without explicit firewall rules, it would happily route guest traffic to the corporate network.

The firewall rules are what actually enforce the isolation. This is the same principle that applies to VPN: the tunnel network (10.0.100.0/24) is just another subnet pfSense can route to — firewall policy determines what crosses between networks.

**Separate networks provide isolation at the traffic level. Firewall rules provide isolation at the policy level. One without the other is incomplete.**

### DHCP for Guest Network

Enabled DHCP on pfSense for the VLAN 20 interface only — range 10.0.20.100 through 10.0.20.200. DNS set to 8.8.8.8 (external), not 10.0.0.10 (DC01). Guests should never interact with domain infrastructure.

Two DHCP servers in the environment, each responsible for their own network:
- DC01 handles 10.0.0.0/24 (corporate)
- pfSense handles 10.0.20.0/24 (guest)

No overlap, no conflict.

### Firewall Rules — Order Matters

pfSense processes rules top to bottom and stops at the first match. Order determines behavior.

Rules on VLAN 20 interface, in order:

| # | Action | Protocol | Source | Destination | Port | Purpose |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | VLAN20 net | Any | 53 (DNS) | Allow guest DNS resolution |
| 2 | Block | Any | VLAN20 net | 10.0.0.0/24 | Any | Block all guest access to corporate |
| 3 | Pass | TCP | VLAN20 net | Any | 80, 443 | Allow guest web browsing |

**Why this order matters:**
- Rule 1 (DNS) must be above Rule 2 (block corporate). If the block rule was first, it would catch DNS queries headed to any destination and kill them before the DNS allow rule was ever evaluated. Guests would have "internet access" that doesn't work because nothing resolves — the exact symptom pattern of "can ping 8.8.8.8 but can't browse."
- Rule 2 (block) must be above Rule 3 (allow internet). If the allow rule was first, guest traffic to corporate ports 80/443 would match the allow rule and pass through — the block would never fire.

---

## OpenVPN Server Configuration

### Architecture

Built a full OpenVPN remote access server enabling authenticated users to securely access the corporate network from any location.

**Components:**
- **Certificate Authority (MeridianLabs-CA):** RSA 2048-bit, SHA-256, 10-year validity. Trust anchor for the VPN — server and client certificates must be signed by this CA.
- **Server Certificate (MeridianLabs-VPN-Server):** Proves pfSense's identity to connecting clients. Prevents man-in-the-middle attacks where a fake VPN endpoint could capture credentials.
- **Tunnel Network:** 10.0.100.0/24 — VPN clients receive addresses from this subnet.
- **Local Network:** 10.0.0.0/24 — tells VPN clients "you can reach this network through the tunnel."
- **DNS Server:** 10.0.0.10 (DC01) — pushed to VPN clients so they can resolve internal names like dc01.lab.local.

**Server mode:** Remote Access (SSL/TLS + User Auth) — requires both a valid certificate AND username/password for connection.

### Firewall Rules for VPN

**WAN:** Allow UDP 1194 inbound (OpenVPN default port).
**OpenVPN interface:** Allow all traffic from VPN clients to LAN (permissive for lab; production would restrict to specific destinations).

### How VPN Works at the Network Level

When a remote user connects: their device establishes an encrypted TLS tunnel to pfSense over the internet. pfSense assigns a second IP from 10.0.100.0/24. The user's device now has two IPs — their home network address and the VPN tunnel address. Traffic destined for 10.0.0.0/24 routes through the encrypted tunnel to pfSense, which forwards it to the corporate network. The corporate servers see the VPN client as a local device.

The VPN tunnel network is just another subnet pfSense has a foot on — same routing concept as the guest VLAN, just with the opposite intent. Guest VLAN: separate network, blocked from corporate. VPN: separate network, allowed into corporate. The firewall policy is what distinguishes them.

### Testing Limitation

OpenVPN server was running and verified with `sockstat -4 -l | grep 1194` showing the process listening on UDP 1194. Client configuration exported successfully.

Connection testing from the host machine failed due to VirtualBox networking limitations:
- **NAT port forwarding:** Configured UDP 1194 forwarding from host to pfSense WAN. Ran `tcpdump -i em0 udp port 1194` on pfSense — no packets arrived. Port forwarding was not delivering UDP traffic despite correct configuration.
- **Bridged mode:** Switched pfSense WAN to bridged adapter on host Wi-Fi (Intel Wi-Fi 6E AX211). pfSense received an IP from the lab network (10.0.0.31) instead of the home router — the Wi-Fi driver doesn't support bridged mode. Tested with promiscuous mode set to "Allow All" — same result.

**Conclusion:** VPN server configuration is correct. Testing was constrained by VirtualBox's NAT UDP forwarding limitations and Wi-Fi bridging incompatibility. In a Proxmox or bare-metal environment with proper network interfaces, this would test successfully.

---

## Networking Breakage Exercises

### NET-001: Wrong DNS on Client

**Ticket:** User reports internet is completely down. Can't reach any websites.

**Diagnosis:** `ipconfig /all` on the workstation showed DNS server set to a static 10.0.0.30 instead of the DHCP-assigned 10.0.0.10. All other settings (IP, gateway, DHCP lease) were correct.

**Why this breaks browsing but not connectivity:** DNS translates domain names to IP addresses. With the wrong DNS server, names can't resolve — but the network path to the internet is still functional. `ping 8.8.8.8` would succeed (raw IP connectivity works), but `ping google.com` would fail (name resolution is broken). This is the classic "can ping by IP but can't browse" pattern.

**Fix:** Switched adapter back to DHCP, ran `ipconfig /flushdns` to clear the stale cache of failed lookups, then `ipconfig /release` and `/renew` to pull fresh DHCP configuration. Connection restored.

**Root cause investigation:** DNS was set to a static IP rather than DHCP-assigned. Most likely causes in production: a previous technician set it during troubleshooting and forgot to revert, or the user followed an internet guide to "speed up DNS" by hardcoding an external server. When one client has wrong DNS but everyone else is fine, static override is the first suspect.

**Ticket closure:**
> User reported workstation internet is down. Ran `ipconfig /all` — workstation pointing to static DNS server 10.0.0.30 instead of DHCP-assigned 10.0.0.10. Switched adapter back to DHCP, flushed DNS cache, ran release/renew. Connection restored. Root cause of static DNS entry unknown — flagged for follow-up to prevent recurrence.