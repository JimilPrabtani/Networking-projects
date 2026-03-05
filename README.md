# Site-to-Site IPSec VPN Configuration Lab with SHA-512

## Overview

This lab demonstrates a practical Site-to-Site IPSec VPN setup with **AES-256 encryption and SHA-512 hashing**. It simulates connecting two branch offices securely over the internet using modern cryptographic standards.

**Use Case:** Secure communication between Site A (Head Office) and Site B (Branch Office) over untrusted internet.

---

## Network Topology

```
┌─────────────────┐         ┌──────────────┐         ┌─────────────────┐
│   Site A (R1)   │         │ ISP Router   │         │   Site B (R3)   │
│  10.1.1.0/24    ├────────┤  (R2)        ├────────┤  10.2.2.0/24    │
│ LAN: 10.1.1.1   │ WAN:    │              │ WAN:    │ LAN: 10.2.2.1   │
└─────────────────┘ .168.12 └──────────────┘ .168.23 └─────────────────┘
        │            .1-/24    Plain Routing  .0-/24        │
        │                       No Encryption               │
        └─────────────IPSec Tunnel (AES-256+SHA512)─────────┘
```

---

## IP Address Configuration

| Device | Interface | IP Address | Purpose |
|--------|-----------|------------|----------|
| R1 | G0/0 | 192.168.12.1 | WAN (to ISP) |
| R1 | G0/1 | 10.1.1.1 | LAN (Head Office) |
| R2 | G0/0 | 192.168.12.2 | WAN (from R1) |
| R2 | G0/1 | 192.168.23.1 | WAN (to R3) |
| R3 | G0/0 | 192.168.23.2 | WAN (to ISP) |
| R3 | G0/1 | 10.2.2.1 | LAN (Branch Office) |

---

## Configuration

### Step 1: Basic IP Setup

**R1 (Head Office):**
```cisco
enable
configure terminal
hostname R1

interface GigabitEthernet0/0
 ip address 192.168.12.1 255.255.255.0
 no shutdown

interface GigabitEthernet0/1
 ip address 10.1.1.1 255.255.255.0
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.12.2
exit
```

**R2 (ISP - No VPN Config):**
```cisco
enable
configure terminal
hostname R2

interface GigabitEthernet0/0
 ip address 192.168.12.2 255.255.255.0
 no shutdown

interface GigabitEthernet0/1
 ip address 192.168.23.1 255.255.255.0
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.12.1
exit
```

**R3 (Branch Office):**
```cisco
enable
configure terminal
hostname R3

interface GigabitEthernet0/0
 ip address 192.168.23.2 255.255.255.0
 no shutdown

interface GigabitEthernet0/1
 ip address 10.2.2.1 255.255.255.0
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.23.1
exit
```

---

### Step 2: Phase 1 - ISAKMP (Authentication & Key Exchange)

**Apply on BOTH R1 and R3:**

```cisco
crypto isakmp policy 10
 encryption aes 256
 hash sha512
 authentication pre-share
 group 2
 lifetime 86400

crypto isakmp key SECRETKEY123 address 192.168.23.2    ! R1 points to R3
crypto isakmp key SECRETKEY123 address 192.168.12.1    ! R3 points to R1
```

**What Each Line Does:**
- `encryption aes 256` — 256-bit AES encryption (military-grade)
- `hash sha512` — SHA-512 for data integrity (stronger than SHA-1)
- `authentication pre-share` — Both sides verify identity using shared secret
- `group 2` — Diffie-Hellman Group 2 for key exchange
- `lifetime 86400` — Key valid for 24 hours, then re-negotiated
- `crypto isakmp key` — The pre-shared secret (must match on both sides)

---

### Step 3: Phase 2 - IPSec Transform Set (Data Encryption)

**Apply on BOTH R1 and R3:**

```cisco
crypto ipsec transform-set MYSET esp-aes 256 esp-sha512-hmac
 mode tunnel

crypto ipsec security-association lifetime seconds 3600
```

**What This Does:**
- `esp-aes 256` — Encrypt actual customer data with 256-bit AES
- `esp-sha512-hmac` — HMAC-SHA512 verifies data wasn't tampered with
- `mode tunnel` — Encapsulate entire original packet (not transport mode)
- `lifetime 3600` — IPSec tunnel re-negotiated every 1 hour

---

### Step 4: Define Interesting Traffic (ACL)

**On R1:**
```cisco
ip access-list extended VPN-TRAFFIC
 permit ip 10.1.1.0 0.0.0.255 10.2.2.0 0.0.0.255
```

**On R3:**
```cisco
ip access-list extended VPN-TRAFFIC
 permit ip 10.2.2.0 0.0.0.255 10.1.1.0 0.0.0.255
```

**Important:** ACLs are **mirror images** — source and destination are reversed. This tells each router which traffic triggers encryption.

---

### Step 5: Create Crypto Map & Apply to Interface

**On R1:**
```cisco
crypto map VPNMAP 10 ipsec-isakmp
 set peer 192.168.23.2
 set transform-set MYSET
 match address VPN-TRAFFIC
 exit

interface GigabitEthernet0/0
 crypto map VPNMAP
 exit
```

**On R3:**
```cisco
crypto map VPNMAP 10 ipsec-isakmp
 set peer 192.168.12.1
 set transform-set MYSET
 match address VPN-TRAFFIC
 exit

interface GigabitEthernet0/0
 crypto map VPNMAP
 exit
```

**What Crypto Map Does:**
- `set peer` — Remote VPN endpoint IP address
- `set transform-set` — Links to Phase 2 encryption settings
- `match address` — Links to ACL (defines interesting traffic)
- Applied to interface activates the entire VPN

---

## Verification

### Check Phase 1 Status
```cisco
show crypto isakmp sa
```

**Expected Output:**
```
dst              src              state          conn-id  status
192.168.23.2     192.168.12.1     QM_IDLE        1        ACTIVE
```

✅ `ACTIVE` = IKE session established
✅ `QM_IDLE` = Ready for Quick Mode (Phase 2)

### Check Phase 2 Status
```cisco
show crypto ipsec sa
```

**Expected Output:**
```
interface: GigabitEthernet0/0
 Crypto map tag: VPNMAP, local addr 192.168.12.1
 
 current_peer 192.168.23.2 port 500
 PERMIT ip 10.1.1.0/24 10.2.2.0/24
 
 #pkts encaps: 47    ← Packets encrypted going out
 #pkts decaps: 45    ← Packets decrypted coming in
 #pkts encrypt ok: 47
 #pkts decrypt ok: 45
```

✅ If `#pkts encaps` increases when you ping — VPN is working!

---

## Testing the VPN

### Ping Test

From Site A to Site B:
```cisco
R1# ping 10.2.2.10 repeat 5
```

**Success:**
```
Sending 5, 32-byte ICMP Echos to 10.2.2.10
Received 5 replies from 10.2.2.10 (via 192.168.23.2)
Success rate is 100 percent
```

### Verify Encryption

```cisco
show crypto ipsec sa | include "pkts encaps\|pkts decaps"
```

Run this while pinging — numbers should increase, proving traffic is encrypted.

---

## How IPSec VPN Works

### Phase 1: Authentication Handshake

```
1. R1 → R3: "Hi, I want to establish VPN"
2. R3 → R1: "I know you, using pre-shared key"
3. Both negotiate: AES-256, SHA512, DH Group 2
4. Both derive shared secret key via Diffie-Hellman
5. Secure management channel established ✅
```

**Time:** Happens once, valid for 24 hours

### Phase 2: Data Tunnel Setup

```
1. Using Phase 1 secure channel, negotiate IPSec parameters
2. Quick Mode (QM) establishes data tunnel
3. Each direction gets own Security Association (SA)
4. Tunnel ready to encrypt/decrypt ✅
```

**Time:** Happens once Phase 1 is up, valid for 1 hour

### Data Flow Through Tunnel

```
Step 1: PC1 (10.1.1.10) sends ping to PC2 (10.2.2.10)
        ↓
Step 2: Packet arrives at R1
        ↓
Step 3: R1 checks ACL: "Is 10.1.1.0 → 10.2.2.0? YES"
        ↓
Step 4: R1 encrypts with AES-256 + SHA512
        ↓
Step 5: R1 wraps in new IP header (192.168.12.1 → 192.168.23.2)
        ↓
Step 6: ISP/R2 sees encrypted blob, has NO visibility
        ↓
Step 7: R3 receives encrypted packet
        ↓
Step 8: R3 decrypts using matching keys
        ↓
Step 9: R3 checks ACL: "Is 10.2.2.0 → 10.1.1.0? YES"
        ↓
Step 10: R3 forwards original packet to PC2
```

---

## Security Comparison

| Feature | Benefit |
|---------|----------|
| **AES-256** | Military-grade confidentiality |
| **SHA-512** | Cryptographically strong integrity checking |
| **Pre-Shared Key** | Both sides must know the secret |
| **DH Group 2** | Secure key exchange (Perfect Forward Secrecy) |
| **24-Hour IKE Lifetime** | Keys re-negotiated daily |
| **1-Hour IPSec Lifetime** | Tunnel re-negotiated hourly |

---

## Troubleshooting

### Phase 1 Won't Come Up
```cisco
! 1. Verify pre-shared keys match
show run | include "crypto isakmp key"

! 2. Verify peer IPs are correct
! R1 should point to R3's WAN IP (192.168.23.2)
! R3 should point to R1's WAN IP (192.168.12.1)

! 3. Verify encryption/hash match
show crypto isakmp policy
```

### Phase 1 Up But Phase 2 Won't Come Up
```cisco
! 1. Verify ACLs match exactly (mirrored)
show access-lists VPN-TRAFFIC

! 2. Trigger interesting traffic
ping 10.2.2.10

! 3. Verify crypto map applied to WAN interface
show crypto map
```

### Tunnel Works But No Throughput
```cisco
! Check MTU size
show interface g0/0 | include MTU

! Might need to lower MTU for IPSec overhead
interface g0/0
 ip mtu 1400
```

---

## Interview Answers

**Q: How does IPSec VPN work?**

A: "IPSec works in two phases. Phase 1 uses IKE (Internet Key Exchange) to authenticate both endpoints using a pre-shared key and establish a secure management channel with negotiated encryption parameters like AES-256 and SHA-512. Phase 2 uses that secure channel to establish the actual data tunnel. Traffic matching the interesting traffic ACL gets encrypted at the source router, encapsulated with a new IP header, and sent across the internet. The ISP only sees encrypted traffic between two public IPs with no visibility into the actual customer data. The destination router decrypts using matching Phase 1 and 2 parameters and forwards the original packet."

**Q: Why use SHA-512 over SHA-1?**

A: "SHA-512 provides stronger cryptographic integrity. It produces a 512-bit hash compared to SHA-1's 160-bit hash, making it virtually impossible to forge a valid message. SHA-1 is considered cryptographically broken and should not be used for new systems. In production networks, we use SHA-512 for HMAC to verify data wasn't tampered with in transit."

**Q: What's interesting traffic?**

A: "Interesting traffic is defined by an ACL and tells the router which packets should go through the VPN tunnel. For example, traffic from 10.1.1.0/24 to 10.2.2.0/24 is interesting and gets encrypted. Traffic from 10.1.1.0 to the internet is not interesting and goes unencrypted. The ACLs on both sides must be exact mirrors of each other."

---

## Key Takeaways

1. **Two Phases:** Phase 1 authenticates, Phase 2 encrypts
2. **Pre-Shared Key:** Must match exactly on both sides
3. **ACL:** Defines which traffic gets encrypted
4. **ISP Blind:** Internet provider has no visibility into customer data
5. **Crypto Map:** Glues Phase 1, Phase 2, and ACL together
6. **SHA-512:** Modern cryptographic hash (stronger than SHA-1)
7. **Verify Commands:** Always use `show crypto isakmp sa` and `show crypto ipsec sa`

---

**Lab Complete!** You now understand Site-to-Site IPSec VPN with modern encryption standards. This is production-grade security using industry-standard protocols.

---

*Security Note: In production, use strong random pre-shared keys (20+ characters). Consider upgrading to IKEv2 for better security. Regularly rotate keys and monitor tunnel status.*
