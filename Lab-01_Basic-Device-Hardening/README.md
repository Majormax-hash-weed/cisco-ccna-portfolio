# LAB 01: Control Plane Hardening & Secure Remote Management Architecture

## 1. Technical Executive Summary & Domain Overview

The control plane of a network device — the logical subsystem responsible for processing routing decisions, terminal access, and administrative session negotiation — represents the single highest-value attack surface in enterprise infrastructure. Unlike the data plane, which merely forwards traffic between endpoints, the control plane holds the keys to the kingdom: it is where `enable` passwords are validated, where VTY sessions are authenticated, and where an attacker who gains a foothold can pivot into full administrative control of every downstream device.

Cisco IOS devices ship from the factory in a state that is architecturally hostile to security. Telnet (`transport input all`) is enabled by default on VTY lines, meaning credentials and command syntax traverse the wire as cleartext ASCII. Passwords, when configured, are stored in plaintext inside the running-configuration unless `service password-encryption` is explicitly invoked. There is no mandated timeout on idle sessions, no enforced identity separation between administrators, and no legal notification banner establishing that access is monitored. In this state, a single instance of packet capture (ARP spoofing plus Wireshark, or a compromised intermediate switch) is sufficient to fully compromise the device.

This lab documents the transformation of a two-node topology — a Cisco 2911 Integrated Services Router (**R1**) acting as the edge gateway and a Cisco 2960 Catalyst switch (**SW1**) acting as the access-layer core — from that factory-default posture into a hardened enterprise management baseline for the fictional organization **NovaTech Solutions**.

**Threat models explicitly mitigated by this hardening pass:**

| Threat Vector | Factory-Default Exposure | Hardened Mitigation Applied |
|---|---|---|
| Passive credential interception (packet sniffing) | Telnet transmits usernames/passwords in cleartext | `transport input ssh` — Telnet fully decommissioned on all VTY lines |
| Session hijacking via idle terminals | No timeout; console/VTY stay authenticated indefinitely | `exec-timeout 5 0` on console; `ip ssh time-out 60` on SSH sessions |
| Credential exposure from stolen config backups | Cleartext passwords stored in running-config/TFTP backups | `service password-encryption` (Type 7) + `enable secret` (Type 5 MD5 hash) |
| Lack of accountability / shared credentials | Single global `line password` used by all administrators | `login local` bound to individually created `username admin` account |
| Absence of legal deterrent / evidentiary basis | No warning banner presented pre-authentication | `banner motd` enforcing an "Authorized Access Only" legal notice on both devices |
| Brute-force / dictionary login attempts | No protocol-level authentication ceiling | SSHv2 enforcement with IOS default authentication-retry ceiling (3 attempts) |

---

## 2. Infrastructure Topology & Subnet Matrix

The entire administrative management plane for this lab is confined to a single flat Class C-sized segment. This is intentional: in a production branch deployment, out-of-band or restricted in-band management traffic should never share broadcast domains with general user data — but for this foundational hardening lab, the management and data segment are collapsed into one subnet to isolate the *device-hardening* variables from *routing/VLAN segmentation* variables (covered in a later lab).

**Subnet Parameters:**

| Parameter | Value |
|---|---|
| Network Address | `192.168.1.0` |
| Subnet Mask | `255.255.255.0` (`/24`) |
| CIDR Notation | `192.168.1.0/24` |
| Usable Host Range | `192.168.1.1` – `192.168.1.254` |
| Broadcast Address | `192.168.1.255` |
| Total Usable Hosts | 254 |

**Device Address Assignments:**

| Device | Role | Interface | IPv4 Address |
|---|---|---|---|
| R1 | Edge Gateway Router (Cisco 2911 ISR) | GigabitEthernet0/0 | `192.168.1.1` |
| SW1 | Enterprise Core Switch (Cisco 2960) | VLAN 1 (SVI) | `192.168.1.2` |
| PC1 | Administrative Host Terminal | NIC (DHCP/Static) | `192.168.1.10` |

`SW1` resolves off-segment traffic and out-of-network administrative reachability through `ip default-gateway 192.168.1.1`, which points directly at R1's `Gi0/0` interface — establishing R1 as the Layer 3 boundary for this segment.

![Network Topology Layout](./Lab-01_Topology.png)

---

## 3. Logical Traffic Flow & Structural Architecture Map

```text
    ┌────────────────────────────────────────────────────────┐
    │              ADMINISTRATIVE OPERATION HOST (PC1)        │
    │              IP Address Assignment: 192.168.1.10        │
    └───────────────────────────┬────────────────────────────┘
                                 │
                    [Traffic Parameters Injected]
                    ► Layer 3 Protocol: ICMP Echo Request / Reply
                    ► Layer 4 Encapsulation: TCP Port 22 (SSHv2)
                    ► Layer 4 Encapsulation: TCP Port 23 (Telnet — REJECTED)
                                 │
                                 ▼ [Medium: Copper Ethernet, Fa0/24]
    ┌────────────────────────────────────────────────────────┐
    │               ENTERPRISE SWITCH CORE (SW1)              │
    │  • Physical Access Port: FastEthernet0/24                │
    │  • Port Mode: 'switchport mode access'                   │
    │  • VTY Line Filter: transport input ssh (0-4, 5-15)      │
    │  • Management Interception: Interface Vlan1 SVI          │
    │    Target Operational IP: 192.168.1.2                    │
    │  • Default Route Egress: ip default-gateway 192.168.1.1  │
    └───────────────────────────┬────────────────────────────┘
                                 │
                    [Layer 3 Forwarding Decision]
                    ► Traffic destined off-segment routes
                      via default-gateway statement
                                 │
                                 ▼ [Medium: Copper Uplink]
    ┌────────────────────────────────────────────────────────┐
    │                EDGE SEGMENT GATEWAY (R1)                 │
    │  • Gateway Interface: GigabitEthernet0/0                 │
    │  • Description: "*** LINK TO SW1 ***"                    │
    │  • Interface IP: 192.168.1.1/24                          │
    │  • VTY Line Filter: transport input ssh (0-4)             │
    └────────────────────────────────────────────────────────┘
```

### Detailed Phase Analysis

**Phase 1 — Host Egress and Segment Formation:** When the administrator on `PC1` initiates a management session against `192.168.1.2`, the local IP stack constructs a packet with the destination socket set to **TCP/22**. The Ethernet frame is placed on the wire toward the switch's access port. If the client instead targets **TCP/23** (Telnet), the packet is still transmitted onto the wire by PC1 — the rejection does not happen at the host, it happens at the **VTY line boundary** on the receiving device, discussed in Phase 3.

**Phase 2 — Physical Ingress at SW1 (Fa0/24):** The frame arrives on `FastEthernet0/24`, which is explicitly hardcoded via `switchport mode access`. This forces the port into a single, non-negotiable access mode — it will never form a trunk and will never carry 802.1Q tags, closing off VLAN-hopping vectors (double-tagging, DTP spoofing) that exist on dynamic (`switchport mode dynamic auto/desirable`) ports.

**Phase 3 — Control-Plane Interception at the SVI:** Because the destination IP (`192.168.1.2`) belongs to the switch's own management interface (`interface Vlan1`), the frame is not forwarded through the MAC address table toward another host — it is punted up to the switch's local control-plane process. Here, the IOS TCP/IP stack inspects the destination port against the VTY line configuration:
- If the socket is **TCP/22** and `transport input ssh` is set on the matching VTY line (`line vty 0 4` or `line vty 5 15`), the three-way TCP handshake completes and the SSHv2 key exchange begins.
- If the socket is **TCP/23**, the VTY line's `transport input ssh` directive contains no allowance for Telnet transport — the connection is refused at the line level before any credential prompt is ever generated. This is not a firewall/ACL rejection; it is a protocol-transport-list rejection native to the line configuration itself.

**Phase 4 — Onward Routing to R1:** For any administrative action requiring traffic to leave the `192.168.1.0/24` segment (e.g., a management workstation reaching R1's Gi0/0 directly for router-side administration), SW1 consults `ip default-gateway 192.168.1.1` and forwards the frame to R1's `Gi0/0`. R1 independently re-applies the identical control-plane logic on its own `line vty 0 4` — each device enforces its own, self-contained line-level SSH boundary rather than relying on a shared or centralized filtering point.

---

## 4. Engineering Implementation Analysis & Threat Mitigation

### a) Cryptographic Transport Enforcement

Both devices carry the identical baseline:

```text
ip ssh version 2
ip ssh time-out 60
no ip domain-lookup
ip domain-name novatech.local
```

`ip ssh version 2` is a hard version pin. IOS supports a legacy compatibility mode where both SSHv1 and SSHv2 clients are accepted; SSHv1 contains structurally broken CRC-32 integrity checking (CVE-2001-0572-class weaknesses) and is disabled outright here rather than left in dual-mode. The `ip domain-name novatech.local` statement is not cosmetic — it is a mandatory prerequisite for the `crypto key generate rsa` process, since IOS uses the hostname plus domain name as the identifier bound into the RSA key pair (`R1.novatech.local`, `SW1.novatech.local`). Without a domain name configured, `crypto key generate rsa` cannot execute and `ip ssh version 2` has no cryptographic material to bind the transport to.

`ip ssh time-out 60` caps the *authentication negotiation window* — a partially-authenticated SSH session (one that has opened a TCP socket but not completed the credential exchange) is force-closed after 60 seconds. This directly forecloses on slow-loris-style connection exhaustion attacks against the VTY pool, where an attacker opens sessions and stalls the login prompt indefinitely to consume all available VTY lines. The default IOS authentication-retry ceiling (3 attempts before the session is torn down) remains in effect and is confirmed operationally in Section 6, Test Phase 4.

On the VTY lines themselves, `transport input ssh` is applied with **no fallback list** — this is a deliberate architectural choice over `transport input ssh telnet`, which would silently re-open the plaintext vector this entire lab exists to close.

One structural asymmetry worth flagging: **R1 exposes only `line vty 0 4`** (5 concurrent sessions), whereas **SW1 exposes `line vty 0 4` and `line vty 5 15`** (16 concurrent sessions total). This reflects SW1's role as the more frequently-touched access-layer device in a larger fabric, while R1, as a single edge gateway, has a narrower expected concurrent-administrator count.

### b) Identity Management & Localized Authorization

Both devices reject the single-shared-secret line-password model in favor of a locally-authenticated identity database:

```text
username admin privilege 15 secret 5 $1$mERr$fuJbp3RdiIdObhS8awSQ..   (R1)
username admin secret 5 $1$mERr$fuJbp3RdiIdObhS8awSQ..                (SW1)
```

`login local` on every VTY line forces authentication against this local user database rather than a flat `line vty 0 4 / password X` statement. The architectural difference matters: a shared line password produces an AAA log entry that says only *"someone with the password logged in"* — it cannot distinguish between administrators. `login local` produces a distinct, attributable identity per session, which is the minimum prerequisite for any future audit trail or `logging` correlation. On R1, the `admin` account is additionally provisioned with `privilege 15` (full EXEC rights), collapsing authentication and authorization into a single explicit statement rather than leaving privilege level ambiguous.

Both devices present a `banner motd` **before** the login prompt is ever reached:

- **R1:** `AUTHORIZED ACCESS ONLY NovaTech Solutions` / `Unauthorized access is PROHIBITED and will be prosecuted.` / `All sessions are monitored and logged.`
- **SW1:** `AUTHORIZED ACCESS ONLY --SW1 NovaTech Solutions`

This is not a cosmetic welcome message — legally, an MOTD banner delivered *pre-authentication* is what establishes that an intruder had explicit notice they were accessing a restricted, monitored system, which is a foundational element in most computer-misuse prosecution frameworks (analogous to the U.S. CFAA "authorized access" standard). A device with no banner cannot demonstrate that an unauthorized user was ever warned.

### c) Data-at-Rest Security Frameworks

Two distinct obfuscation/hashing tiers are stacked on these devices:

```text
service password-encryption
enable secret 5 $1$mERr$c5rFZDLK1OFJTBW3GgY1R0
```

`enable secret` uses **Type 5**, an MD5-based one-way hash (`$1$<salt>$<hash>`) applied to the privileged-EXEC password. Because it is a true one-way cryptographic hash rather than a reversible cipher, it cannot be mathematically decrypted back to plaintext — the only attack surface against it is offline dictionary/brute-force hash cracking, which is computationally far more expensive than reversing an obfuscation scheme.

`service password-encryption` globally applies **Type 7** obfuscation to every other plaintext password stored in the configuration — in this deployment, that covers the `line con 0 password 7 080243401A160912322503122B1F` entries on both devices. Type 7 is a **reversible Vigenère-style XOR cipher** using a fixed, publicly known key schedule hardcoded into IOS. It stops shoulder-surfing a `show running-config` screen or a casual glance at a backup file, but it provides **no cryptographic protection** — any Type 7 string can be reversed instantly using widely available decoder tools (including built-in functions in Packet Tracer's own Cisco IOS emulation and dozens of public web-based decoders). This is precisely why the *privileged* `enable` credential is deliberately elevated to Type 5, while only the lower-consequence console-line password is left under the weaker Type 7 scheme.

An architectural observation worth documenting for review: R1 and SW1 currently share **identical** `enable secret` and `username admin secret` hash values (`$1$mERr$...` on both). This indicates the same enable and admin passwords were provisioned across both devices. Functionally this simplifies lab administration, but in a production hardening review it would be flagged — shared credentials across devices mean a single compromised hash defeats both nodes simultaneously, collapsing the intended per-device blast-radius isolation.

---

## 5. Deployment Configurations & Scripts

### 5.1 Edge Gateway Router (R1)

📂 **Local Repository Link:** [View Raw R1 Script File](./Lab-01-R1-running-config.txt)

```cisco
hostname R1
!
service password-encryption
!
enable secret 5 $1$mERr$c5rFZDLK1OFJTBW3GgY1R0
!
username admin privilege 15 secret 5 $1$mERr$fuJbp3RdiIdObhS8awSQ..
!
ip ssh version 2
ip ssh time-out 60
no ip domain-lookup
ip domain-name novatech.local
!
interface GigabitEthernet0/0
 description *** LINK TO SW1 ***
 ip address 192.168.1.1 255.255.255.0
 duplex auto
 speed auto
!
banner motd ^C
AUTHORIZED ACCESS ONLY NovaTech Solutions
Unauthorized access is PROHIBITED and will be prosecuted.
All sessions are monitored and logged.
^C
!
line con 0
 exec-timeout 5 0
 password 7 080243401A160912322503122B1F
 logging synchronous
 login
line vty 0 4
 login local
 transport input ssh
```

![Router Running Config Part 1](./R1%20Running-config.png)
![Router Running Config Part 2](./R1%20Running-config2.png)
![Router Running Config Part 3](./R1%20Running-config3.png)

### 5.2 Enterprise Core Switch (SW1)

📂 **Local Repository Link:** [View Raw SW1 Script File](./Lab-01-SW1_running-config.txt)

```cisco
hostname SW1
!
service password-encryption
!
enable secret 5 $1$mERr$c5rFZDLK1OFJTBW3GgY1R0
!
username admin secret 5 $1$mERr$fuJbp3RdiIdObhS8awSQ..
!
ip ssh version 2
ip ssh time-out 60
no ip domain-lookup
ip domain-name novatech.local
!
interface FastEthernet0/24
 switchport mode access
!
interface Vlan1
 ip address 192.168.1.2 255.255.255.0
!
ip default-gateway 192.168.1.1
!
banner motd ^C
======================================================================
AUTHORIZED ACCESS ONLY --SW1 NovaTech Solutions
======================================================================
^C
!
line con 0
 password 7 080243401A160912322503122B1F
 logging synchronous
 login
 exec-timeout 5 0
line vty 0 4
 login local
 transport input ssh
line vty 5 15
 login local
 transport input ssh
```

![Switch Running Config Part 1](./SW1%20Running-Config.png)
![Switch Running Config Part 2](./SW1%20Running-Config-.png)

---

## 6. Verification Protocols & Operational Diagnostics

### Test Phase 1: Bidirectional Transport Validation

**Objective:** Confirm baseline Layer 3 IP reachability across the management segment before layering on protocol-level security tests.

**Methodology:** From `PC1`, issue a standard ICMP echo sweep against SW1's management SVI.

```text
C:\> ping 192.168.1.2

Pinging 192.168.1.2 with 32 bytes of data:
Reply from 192.168.1.2: bytes=32 time<1ms TTL=255
Reply from 192.168.1.2: bytes=32 time<1ms TTL=255
Reply from 192.168.1.2: bytes=32 time<1ms TTL=255
Reply from 192.168.1.2: bytes=32 time<1ms TTL=255

Ping statistics for 192.168.1.2:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

![PC Ping Verification](./PC%20Ping%20test.png)

**Analysis:** 100% delivery, sub-millisecond round-trip time, and a TTL of 255 (indicating a directly-adjacent Layer 3 hop with no intermediate router decrement) confirms clean physical, data-link, and network-layer connectivity before any transport-layer security controls are exercised.

### Test Phase 2: Insecure Terminal Block Testing

**Objective:** Verify that plaintext Telnet is refused at the VTY transport boundary rather than merely being unadvertised.

**Methodology:** From `PC1`, attempt `telnet 192.168.1.2`.

**Analysis:** Because both `line vty 0 4` and `line vty 5 15` on SW1 are configured with `transport input ssh` and **no** `telnet` keyword in the allow-list, the switch's control plane refuses the TCP/23 socket at the line-configuration level — the session is rejected before any username or password prompt is generated. This is distinct from an ACL-based drop: the rejection is a direct consequence of the transport-input protocol list bound to the VTY line itself, meaning there is no code path by which a Telnet session could reach the authentication stage on this device.

### Test Phase 3: Cryptographic Handshake Verification

**Objective:** Confirm the SSH server correctly negotiates the SSHv2 session, enforces the local username database, and displays the pre-authentication legal banner.

**Methodology:** From `PC1`, launch the SSH client utility targeting `192.168.1.2` under identity `admin`.

![SSH Client Configuration Screen](./SSH%20Client%20.png)

```text
======================================================================
AUTHORIZED ACCESS ONLY --SW1 NovaTech Solutions
======================================================================

Password:
SW1>
```

![Active SSH Session Verification](./SW1%20-%20SSH.png)

**Analysis:** The banner is rendered in full **before** any credential prompt appears, satisfying the legal-notice requirement discussed in Section 4(b). Following banner display, the session drops into the `login local` credential challenge; upon a valid match against the `username admin secret 5 ...` hash, the client is placed directly into user EXEC mode (`SW1>`), confirming the SSHv2 key exchange, local-database authentication, and privilege assignment all executed successfully end-to-end.

### Test Phase 4: Operational State Verification

**Objective:** Directly audit the SSH server's runtime state on both devices to confirm the configured protocol version, timeout, and retry parameters are actively enforced (not merely present in the saved configuration).

**Methodology:** Execute `show ip ssh` from privileged EXEC mode on each device.

```text
SW1# show ip ssh
SSH Enabled - version 2.0
Authentication timeout: 60 secs; Authentication retries: 3
```

```text
R1# show ip ssh
SSH Enabled - version 2.0
Authentication timeout: 60 secs; Authentication retries: 3
```

**Analysis:** Both devices confirm `SSH Enabled - version 2.0`, matching the `ip ssh version 2` directive and eliminating any SSHv1 fallback. The `Authentication timeout: 60 secs` field directly mirrors the configured `ip ssh time-out 60`. The `Authentication retries: 3` value reflects the IOS default retry ceiling (unmodified from its factory value, as no `ip ssh authentication-retries` override is present in either running-configuration) — confirming the brute-force mitigation described in Section 4(a) is active on both the edge router and the access-layer switch.
