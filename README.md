# Northbridge Savings & Finance – Network Segmentation (Zone-Based Firewall / ACL) in Cisco Packet Tracer

**Module:** IE3122 – Network Security (Year 3, Semester 1, 2026) – SLIIT
**Assignment:** Northbridge Savings & Finance Network Security Upgrade (Group Y3.S1.WE.CS.02)
**Author of this part:** Aaqib H M – IT24102514

This repository contains my individual contribution to the group assignment: two threat models, four technology proposals with justification, and two network hardening configurations built and verified in Cisco Packet Tracer.

> ⚠️ Lab environment only. Passwords and IP addresses in these files are for the simulation and must never be reused on real equipment.

---

## 📁 Files in this repository

| File | Description |
|---|---|
| [`Zone-Based Firewall.pkt`](Zone-Based%20Firewall.pkt) | **Config 1 (plan-linked)** – STAFF ↔ GUEST/POS segmentation with ACL 110 |
| [`Zone-Based Firewall configure 2.pkt`](Zone-Based%20Firewall%20configure%202.pkt) | **Config 2 (free choice)** – Telnet/SSH block from GUEST/POS with ACL 120 |
| [`Y3S1.WE.CS02.NS_Assignment.pdf`](Y3S1.WE.CS02.NS_Assignment.pdf) | Full group report (my sections: Threats 3 & 4, Appendix B, Figures 10–27) |
| `README.md` | This file |

**How to open:** download a `.pkt` file (click it → **Download raw file**) and open it with **Cisco Packet Tracer 8.x or later**.

---

## 🌐 Scenario (short)

Northbridge is a mid-sized regional bank (head office + 6 branches, ~450 staff). Head-office systems, branch systems, guest Wi-Fi and POS equipment all share **one flat network**, there is no dedicated firewall, no IDS/IPS, no central authentication and no MFA. Regional ransomware incidents started from a single branch workstation and spread across unsegmented networks. This work shows how segmentation can contain that kind of spread.

---

## 🧩 Topology

```
            [Router0 – Cisco 1941]
             Gi0/0.10  192.168.10.1  (VLAN 10 STAFF)
             Gi0/0.20  192.168.20.1  (VLAN 20 GUEST_POS)
                      |  802.1Q trunk
            [Switch0 – Cisco 2960-24TT]
              Fa0/1 (VLAN 10)     Fa0/2 (VLAN 20)
                 |                    |
          [PC1 – STAFF]         [PC2 – GUEST_POS]
          192.168.10.10         192.168.20.10
```

| Device | Interface | VLAN / Zone | IP address |
|---|---|---|---|
| Router0 (1941) | Gi0/0.10 | VLAN 10 – STAFF | 192.168.10.1/24 |
| Router0 (1941) | Gi0/0.20 | VLAN 20 – GUEST_POS | 192.168.20.1/24 |
| Switch0 (2960) | Fa0/1 → PC1 | VLAN 10 | – |
| Switch0 (2960) | Fa0/2 → PC2 | VLAN 20 | – |
| Switch0 (2960) | Gi0/1 → Router0 | 802.1Q trunk | – |
| PC1 | Fa0 | STAFF | 192.168.10.10 (GW .10.1) |
| PC2 | Fa0 | GUEST_POS | 192.168.20.10 (GW .20.1) |

---

## ⚙️ Configuration 1 (Plan-linked) – Zone segmentation STAFF ↔ GUEST/POS

**Linked control:** C1 – Network segmentation (group Layered Security Plan)
**Threats addressed:** T3 (rogue device via unlocked branch cabinet – a device on the guest/POS segment cannot reach staff hosts) and T5 (ransomware lateral movement across the flat network).

**What it does:** Traffic from the GUEST/POS VLAN into the STAFF VLAN is denied. STAFF hosts can still reach GUEST/POS, and ICMP echo-replies are allowed back so staff-initiated pings work.

> Note: The group plan specifies a zone-based firewall. In this lab the same directional policy was enforced with an extended ACL on the GUEST sub-interface.

```
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
!
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 ip access-group 110 in
!
access-list 110 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
access-list 110 permit icmp 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 echo-reply
access-list 110 deny   ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
access-list 110 permit ip any any
```

**Verification** (screenshots: report Figures 10–23)

| Test | Expected | Result |
|---|---|---|
| PC1 → 192.168.10.1 (gateway) | Reply | ✅ 4/4 |
| PC1 (STAFF) → PC2 192.168.20.10 | Reply | ✅ 4/4 (after echo-reply rule) |
| PC2 (GUEST) → PC1 192.168.10.10 | Blocked | ✅ "Destination host unreachable" |
| `show vlan brief` | Fa0/1 in VLAN 10, Fa0/2 in VLAN 20 | ✅ |

---

## ⚙️ Configuration 2 (Free choice) – Block Telnet/SSH from GUEST/POS to STAFF

**What it does:** A protocol-specific extended ACL blocks management protocols (Telnet TCP/23, SSH TCP/22) from the GUEST/POS VLAN toward the STAFF VLAN, while STAFF hosts keep remote management access.

**Real-world problem solved:** A compromised POS terminal or a guest device should never be able to try to log in to staff-side devices. This rule stays in place even if the broader segmentation policy (ACL 110) is later relaxed for other traffic — a second, independent layer for management access.

```
access-list 120 deny   tcp 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 eq 23
access-list 120 deny   tcp 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 eq 22
access-list 120 permit ip any any
!
interface GigabitEthernet0/0.10
 ip access-group 120 out
!
line vty 0 4
 password cisco        ! lab only – used for the Telnet verification test
 login
```

**Verification** (screenshots: report Figures 24–27)

| Test | Expected | Result |
|---|---|---|
| `show access-lists` | ACL 110 + ACL 120 present | ✅ |
| PC2 (GUEST) → `telnet 192.168.10.1` | Blocked | ✅ Connection timed out |
| PC1 (STAFF) → `telnet 192.168.10.1` | Allowed | ✅ Password prompt |

---

## 🛡️ Threat models (asset → vulnerability → threat → exploit → attack → impact → risk)

### T3 – Rogue device / network tap via unlocked branch networking cabinets

| Step | Detail |
|---|---|
| Asset | Branch switches, LAN traffic, workstations and POS devices |
| Vulnerability | Cabinets found unlocked in at least two branches; no port-level restrictions on switches |
| Threat | Malicious insider, opportunistic intruder or visiting contractor with physical access |
| Exploit | Plugging an unauthorised laptop, tap or rogue switch into an open port |
| Attack | Passive sniffing, DHCP spoofing or ARP poisoning on the flat branch network |
| Impact | Interception of credentials and banking data; foothold toward head office; undetected because there are no central logs |
| Risk | **L3 × I4 = 12 (Medium-High)** – needs physical presence, but there is no port security, DHCP snooping, DAI, CCTV or logging |

### T4 – Credential stuffing / account takeover (password reuse & sharing)

| Step | Detail |
|---|---|
| Asset | User accounts on branch workstations and applications (local accounts per device) |
| Vulnerability | Common password reuse and sharing; no central AAA, no password policy, no MFA |
| Threat | External attacker with leaked credentials, or a malicious insider |
| Exploit | Reused passwords from public breaches; one shared password opens several accounts |
| Attack | Automated login attempts with leaked username/password pairs |
| Impact | Unauthorised access and lateral movement toward core banking; likely undetected |
| Risk | **L4 × I4 = 16 (High)** – reuse and sharing are common and there is no MFA anywhere |

---

## 🧠 Technology proposals & justification

| Area | My proposal | Alternative considered & rejected |
|---|---|---|
| VPN | FortiClient IPsec VPN on FortiGate | Direct RDP to jump hosts |
| AAA | FortiAuthenticator + FortiToken MFA | Microsoft AD + NPS |
| Network architecture | Three-tier zones at HQ (perimeter / internal / DMZ) + zone-based firewall on branch routers | Dedicated firewall appliance at every branch |
| IDS/IPS | Suricata (open source) | Cisco Secure IPS |

**VPN – FortiClient IPsec.** Replaces the single shared VPN account with individual accounts plus MFA through FortiAuthenticator, and can check endpoint compliance (AV, patch level) before a tunnel is built — important because Northbridge allows personal devices with no BYOD policy. *Why not direct RDP:* internet-exposed RDP is a common ransomware entry point and, with no MFA and widespread password reuse, it would just replace one unaccountable access model with another.

**AAA – FortiAuthenticator.** One central identity source for VPN, Wi-Fi and device logins, with built-in FortiToken MFA, closing the "no MFA anywhere" and "every device has its own local accounts" gaps. Accounts can be migrated gradually. *Why not AD + NPS:* Northbridge has no directory today; building a full AD forest is a large, slow project for a newly formed team, NPS needs extra licensing for MFA, and a single DC would be a new single point of failure.

**Network architecture – Three-tier HQ + branch router ZBFW.** Strongest protection is concentrated around core banking and the DMZ at head office, while zone rules on the existing branch routers still separate staff, POS and guest Wi-Fi locally, containing a compromised branch workstation. *Why not a firewall at every branch:* six extra appliances is a big capital and licensing cost for a bank whose security spend went almost entirely into physical measures, and managing seven firewalls without central management risks configuration drift.

**IDS/IPS – Suricata.** Free, supports inline prevention, runs on commodity hardware, uses open rule sets (e.g. Emerging Threats) and feeds logs into a central logging/SIEM pipeline — also helping close the "no centralised logging" gap. *Why not Cisco Secure IPS:* recurring licensing at seven sites while the bank is funding segmentation, AAA, patching and training at the same time from a near-zero IT security base.

**Group final selections:** FortiClient IPsec VPN · FortiAuthenticator · Hub-and-spoke with FortiGate branch firewalls (with my three-tier HQ design adopted for the head office) · Suricata at head office.

---

## 👥 Group members

| IT Number | Name |
|---|---|
| IT24102747 | Kosgollage S R |
| IT24201195 | Palugaswewa K I K I B |
| IT24102899 | Shaheen M F M |
| IT24102514 | Aaqib H M |

## 🛠️ Tools

- Cisco Packet Tracer 8.x
- Cisco 1941 router, Cisco 2960-24TT switch, generic PCs

## 📄 Academic use

Submitted as coursework for IE3122 at SLIIT. Shared for learning and portfolio purposes; please do not submit it as your own work.
