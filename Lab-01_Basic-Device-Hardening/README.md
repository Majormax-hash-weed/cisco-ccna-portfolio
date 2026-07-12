# Control Plane Hardening & Secure Remote Management Architecture

## 1. Technical Executive Summary
The primary objective of this infrastructure deployment is to systematically isolate, harden, and defend the management and control planes of an enterprise branch topology. Utilizing a Cisco 2911 Integrated Services Router (ISR) and a Cisco 2960 Catalyst Switch, this architecture implements a highly secure administrative baseline. The design mitigates unauthorized boundary access, replaces plaintext application streams with cryptographic transport, secures data-at-rest system credentials, and introduces structural protection against dictionary attacks to ensure alignment with standard enterprise compliance frameworks.

---

## 2. Infrastructure Topology & Subnet Matrix
The architecture restricts administrative access to an isolated, software-defined management segment. This logical segregation guarantees that administrative traffic loops are structurally separated from standard user payload paths.

* **Management Subnet Boundary:** `192.168.1.0/24`
* **Router R1 Gateway Interface (Gi0/0):** `192.168.1.1`
* **Switch SW1 Management SVI (Vlan1):** `192.168.1.2`
* **Administrative Host Terminal (PC1):** `192.168.1.10`
* **Segment Subnet Mask:** `255.255.255.0`

![Network Diagram](Lab-01_Topology.png)

---

## 3. Logical Traffic Flow & Structural Architecture Map

The reference diagram below delineates the routing vectors, interface boundaries, and socket enforcements utilized during an active administrative management session:

```text
    ┌────────────────────────────────────────────────────────┐
    │              ADMINISTRATIVE OPERATION HOST (PC1)       │
    │              IP Address Assignment: 192.168.1.10      │
    └───────────────────────────┬────────────────────────────┘
                                │
                   [Traffic Parameters Injected]
                   ► Layer 3 Protocol: ICMP Echo Request / Reply
                   ► Layer 4 Encapsulation: TCP Port 22 (SSHv2)
                                │
                                ▼ [Medium: Category 5e UTP Straight-Through]
    ┌────────────────────────────────────────────────────────┐
    │               ENTERPRISE SWITCH CORE (SW1)             │
    │  • Hardened Access Port: FastEthernet 0/24             │
    │  • Port Configuration Mode: 'switchport mode access'   │
    │  • Native Operational Domain: VLAN 1 Binding Only      │
    │  • Management Interception: Interface Vlan1 SVI         │
    │    Target Operational IP: 192.168.1.2                  │
    └───────────────────────────┬────────────────────────────┘
                                │
                   [Subnet Gateways Forwarding]
                   ► Untagged Native Intra-VLAN Traffic Flows
                                │
                                ▼ [Medium: High-Speed Copper Uplink]
    ┌────────────────────────────────────────────────────────┐
    │               EDGE SEGMENT GATEWAY (R1)                │
    │  • Gateway Interface Path: GigabitEthernet 0/0         │
    │  • Ingress Node Hardware Routing IP: 192.168.1.1        │
    └───────────────────────────┴────────────────────────────┘

```

### Operational Phase Analysis:

1. **Host Outbound Ingress:** The administrative terminal initializes an access attempt directed to the switch management interface (`192.168.1.2`). The local communication stack generates an IP packet encapsulated with a **TCP Destination Port 22** signature.
2. **Switchplane Filtering & SVI Interception:** Frames are received at physical interface `Fa0/24` on `SW1`. Layer-2 enforcement rules restrict the ingress boundary to an access port tied exclusively to VLAN 1. The internal switching logic channels the frame up to the Layer 3 Switched Virtual Interface (SVI). The management plane reads the TCP port 22 header, drops legacy port 23 traffic, and initializes asymmetric session verification.
3. **Gateway Routing Vector:** When communications extend beyond the local broadcast domain, frames route through the physical uplink (`Fa0/1` on the switch to `Gi0/0` on the router) where `R1` processes the traffic path according to configured gateway policies.

---

## 4. Engineering Implementation Analysis & Threat Mitigation

### Cryptographic Transport Enforcement (SSHv2)

* **Design Execution:** Insecure Telnet options are decommissioned across all virtual terminal paths (`line vty 0 4` and `line vty 5 15`) and replaced with a strict **SSHv2** constraint (`transport input ssh`).
* **Key Derivation Architecture:** A standard namespace domain (`novatech.local`) is provisioned to seed an asymmetric **2048-bit RSA key pair** on all managed elements. Operational timers drop idle lines at 60 seconds (`ip ssh time-out 60`) and restrict credential attempts to a ceiling of 3 (`ip ssh authentication-retries 3`).
* **Architectural Justification:** Legacy plaintext protocols expose cleartext payloads to data harvesting via mid-stream packet capture tools. Forcing an upgrade to SSHv2 ensures all terminal transactions undergo asymmetric session negotiation, neutralizing credential interception threat vectors.

### Identity Management & Localized Authorization

* **Design Execution:** Shared infrastructure keys are replaced with discrete database entries (`login local`) mapped to distinct user profiles (`username admin`).
* **Execution Mapping:** Account privileges are bound directly to **Privilege Level 15** to allow seamless administrative progression upon verification. Legal notices are implemented globally via the Message of the Day engine (`banner motd`).
* **Architectural Justification:** Individual entry tracking introduces accountability and establishes compatibility paths for eventual integration with centralized AAA server pools (TACACS+/RADIUS). Concurrently, formal login banners satisfy organizational compliance requirements necessary for legal enforcement parameters.

### Data-at-Rest Security Frameworks

* **Design Execution:** One-way **Type-5 MD5 hashing parameters** protect the privileged EXEC mode path (`enable secret`). Legacy system passwords are encrypted globally using Cisco’s baseline obfuscation routine (`service password-encryption`).
* **Architectural Justification:** While standard Type-7 obfuscation deters accidental disclosure during concurrent peer reviews, it lacks long-term cryptographic strength. Applying a mathematical one-way Type-5 hash to administrative execution layers ensures master passwords remain secured against typical dictionary search attacks in the event of configuration repository leaks.

---
## 5. Configuration Scripts
The production scripts for both active infrastructure elements are decoupled from this document to facilitate independent auditing procedures:
* **Edge Gateway Parameters:** [View R1 Configurations](./R1_running-config.txt)
* **Switch Core Parameters:** [View SW1 Configurations](./SW1_running-config.txt)

---

## 6. Verification Protocols & Operational Diagnostics

The following testing phases confirm functional deployment parameters against baseline targets:

### Test Phase 1: Bidirectional Transport Validation

* **Method:** Execute an ICMP echo request from host terminal `PC1` to the core switch virtual interface.
* **Syntax:** `ping 192.168.1.2`
* **Target Result:** 100% success rate with 0% packet loss metrics.

```text
C:\> ping 192.168.1.2
Pinging 192.168.1.2 with 32 bytes of data:
Reply from 192.168.1.2: bytes=32 time<1ms TTL=255
Reply from 192.168.1.2: bytes=32 time<1ms TTL=255
Ping statistics for 192.168.1.2: Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)

```

### Test Phase 2: Insecure Terminal Block Testing

* **Method:** Attempt an unencrypted connection request to the switch plane via standard socket utilities.
* **Syntax:** `telnet 192.168.1.2`
* **Target Result:** Connection dropped or blocked immediately by the remote infrastructure node, validating successful telnet filtration.

### Test Phase 3: Cryptographic Handshake Verification

* **Method:** Initialize an SSH terminal connection from `PC1` targeting the management segment core.
* **Syntax:** Open SSH Client Utility ➔ Target: `192.168.1.2` ➔ Identity: `admin`
* **Target Result:** Device returns appropriate regulatory legal notice parameters and passes the session to executive control mode upon challenge verification.

```text
======================================================================
AUTHORIZED ACCESS ONLY --SW1 NovaTech Solutions
======================================================================
SW1>

```

### Test Phase 4: Operational State Verification

* **Method:** Inspect active cryptographic transport parameters from the local command execution prompt.
* **Syntax:** `show ip ssh`
* **Target Result:** Operating status confirms Version 2.0 implementation with matching timeout limits.

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

```

### What to do next:
1. Paste this completely into that new file window on GitHub.
2. Scroll to the bottom and hit **Commit changes**.
3. Once this saves, check the visual layout. It will look like a completely independent, peer-reviewed engineering documentation summary!

```
