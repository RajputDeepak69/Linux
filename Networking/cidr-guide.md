# CIDR Subnetting & AWS VPC Design

> A practical cheat sheet covering CIDR math, subnet sizing, VLSM, and AWS VPC terminology.

---

## 1. What is CIDR?

**CIDR (Classless Inter-Domain Routing)** defines an IP range using a prefix length:

```
192.168.1.0/24
```

- `/24` = first 24 bits are **network**, remaining 8 bits are **host**
- Replaces rigid classful boundaries (Class A/B/C) with flexible subnet sizes
- The `/X` tells you exactly how many leading bits identify the network

---

## 2. Core Formulas

| What | Formula |
|:---|:---|
| Total IPs in a subnet | `2^(32 − prefix)` |
| Usable hosts (traditional) | `2^(32 − prefix) − 2` |
| Usable hosts (AWS) | `2^(32 − prefix) − 5` |
| Number of subnets from borrowing `n` bits | `2^n` |
| Block size (interesting octet) | `256 − partial_octet_value` |

---

## 3. The "Hard" Prefixes (Partial Octets)

The tricky part is the octet that isn't all 1s or all 0s.

### How to find it:

```
prefix ÷ 8 → full octets
remainder  → bits in the partial octet
```

### Partial Octet Reference (memorize this):

| Bits set | Binary | Decimal |
|:---:|:---:|:---:|
| 1 | `10000000` | 128 |
| 2 | `11000000` | 192 |
| 3 | `11100000` | 224 |
| 4 | `11110000` | 240 |
| 5 | `11111000` | 248 |
| 6 | `11111100` | 252 |
| 7 | `11111110` | 254 |

### Quick Table:

| CIDR | Subnet Mask | Block Size | Total IPs |
|:---:|:---:|:---:|:---:|
| /12 | 255.255.240.0 | 16 (3rd octet) | 1,048,576 |
| /19 | 255.255.224.0 | 32 (3rd octet) | 8,192 |
| /20 | 255.255.240.0 | 16 (3rd octet) | 4,096 |
| /21 | 255.255.248.0 | 8 (3rd octet) | 2,048 |
| /22 | 255.255.252.0 | 4 (3rd octet) | 1,024 |
| /23 | 255.255.254.0 | 2 (3rd octet) | 512 |
| /24 | 255.255.255.0 | 1 (4th octet) | 256 |
| /25 | 255.255.255.128 | 128 (4th octet) | 128 |
| /26 | 255.255.255.192 | 64 (4th octet) | 64 |
| /27 | 255.255.255.224 | 32 (4th octet) | 32 |
| /28 | 255.255.255.240 | 16 (4th octet) | 16 |
| /29 | 255.255.255.248 | 8 (4th octet) | 8 |
| /30 | 255.255.255.252 | 4 (4th octet) | 4 |

---

## 4. Worked Example: Finding a Subnet Range

**Given:** `192.168.37.0/20`

1. **Mask:** 20 → 2 full octets + 4 bits in 3rd → `255.255.240.0`
2. **Block size:** 256 − 240 = **16** (in 3rd octet)
3. **Network:** Round 37 down to nearest multiple of 16 → **32**
   - Network = `192.168.32.0`
4. **Broadcast:** Next block − 1 → `192.168.47.255`
5. **Usable:** `192.168.32.1` – `192.168.47.254`
6. **Total IPs:** 2^(32−20) = **4,096**

---

## 5. Subnetting for AWS VPC

### Example: 10 subnets from `10.0.0.0/24`

**Step 1 — How many bits to borrow?**

| n | 2^n | Enough for 10? |
|:---:|:---:|:---:|
| 3 | 8 | ❌ |
| **4** | **16** | ✅ |

New prefix = `/24 + 4` = **`/28`**

**Step 2 — Block size:**

`/28` → 4 bits in 4th octet → `11110000` = 240
Block size = 256 − 240 = **16**

**Step 3 — List subnets (count by 16):**

| # | Subnet CIDR | Range | AWS Usable |
|:---:|:---|:---|:---:|
| 1 | 10.0.0.0/28 | .0 – .15 | 11 |
| 2 | 10.0.0.16/28 | .16 – .31 | 11 |
| 3 | 10.0.0.32/28 | .32 – .47 | 11 |
| 4 | 10.0.0.48/28 | .48 – .63 | 11 |
| 5 | 10.0.0.64/28 | .64 – .79 | 11 |
| 6 | 10.0.0.80/28 | .80 – .95 | 11 |
| 7 | 10.0.0.96/28 | .96 – .111 | 11 |
| 8 | 10.0.0.112/28 | .112 – .127 | 11 |
| 9 | 10.0.0.128/28 | .128 – .143 | 11 |
| 10 | 10.0.0.144/28 | .144 – .159 | 11 |

> You actually have 16 available (up to `10.0.0.240/28`) — room to grow.

---

## 6. VLSM: Splitting `/16` into 3 AZs (Different Sizes)

**Parent:** `10.0.0.0/16` (65,536 IPs)

| AZ | Purpose | CIDR | Range | Total IPs |
|:---:|:---|:---|:---|:---:|
| A | Production | `10.0.0.0/17` | 10.0.0.0 – 10.0.127.255 | 32,768 |
| B | Staging | `10.0.128.0/18` | 10.0.128.0 – 10.0.191.255 | 16,384 |
| C | Dev | `10.0.192.0/19` | 10.0.192.0 – 10.0.223.255 | 8,192 |
| — | Spare | `10.0.224.0/19` | 10.0.224.0 – 10.0.255.255 | 8,192 |

### The 3-Step Rule:

1. **Size it:** `2^(32 − prefix)` = IPs in that subnet
2. **Place it:** next subnet starts at `previous_end + 1`
3. **No overlap:** each start ≥ previous end + 1

> You don't need equal splits. Just non-overlapping, CIDR-aligned chunks.

---

## 7. AWS Reserved IPs (Critical!)

AWS reserves **5 IPs per subnet**, regardless of size:

| IP | Purpose |
|:---|:---|
| `.0` (first) | Network address |
| `.1` | VPC router (default gateway) |
| `.2` | AWS DNS resolver |
| `.3` | Future use (reserved) |
| Last IP | Broadcast address |

**So:**

| CIDR | Total | AWS Reserved | **Usable** |
|:---:|:---:|:---:|:---:|
| /28 | 16 | 5 | **11** |
| /27 | 32 | 5 | **27** |
| /26 | 64 | 5 | **59** |
| /25 | 128 | 5 | **123** |
| /24 | 256 | 5 | **251** |

> ⚠️ **AWS minimum subnet size is `/28`** (11 usable IPs). You cannot create a `/29` or `/30` in a VPC.

---

## 8. Terminology Cheat Sheet

### Address-Level

| Term | Meaning |
|:---|:---|
| **Network Address** | First IP (all host bits = 0). Identifies the subnet. Never assigned to a device. |
| **Broadcast Address** | Last IP (all host bits = 1). Reaches all hosts on the subnet at once. |
| **Usable Host Range** | Everything between network and broadcast. |
| **Default Gateway** | Where traffic goes when destination is outside the local subnet. In AWS = `.1`. |

### Mask & Sizing

| Term | Meaning |
|:---|:---|
| **Subnet Mask / Netmask** | Dotted-decimal form of the prefix (e.g., `255.255.255.240`). |
| **Prefix Length** | The `/X` — number of leading network bits. |
| **VLSM** | Variable-Length Subnet Masking — different prefix sizes in the same parent. |
| **Block Size / Magic Number** | Increment between subnets. `256 − partial_octet`. |
| **Supernetting / Route Aggregation** | Combining smaller CIDRs into one larger route (reverse of subnetting). |

### Routing

| Term | Meaning |
|:---|:---|
| **Route Table** | Rules: "if destination is X, send via Y." Each subnet in AWS has an associated route table. |
| **Default Route** | `0.0.0.0/0` — catch-all for unknown destinations. Points to IGW (public) or NAT (private). |
| **Longest Prefix Match** | Most specific route wins. `/24` beats `/16` beats `/0`. |
| **Route Propagation** | Routes auto-added due to gateways/peering connections. |

### AWS VPC-Specific

| Term | Meaning |
|:---|:---|
| **Public Subnet** | Has route to **Internet Gateway (IGW)**. Resources get public IPs, directly reachable from internet. |
| **Private Subnet** | No IGW route. Outbound via **NAT Gateway** (you initiate, internet can't reach you). |
| **NAT Gateway** | Managed device in a *public* subnet; lets private subnets make outbound connections. |
| **VPC Peering** | Direct private connection between two VPCs. Add routes on both sides. |
| **Transit Gateway** | Central hub connecting many VPCs + on-prem via Direct Connect. Replaces peering mesh at scale. |
| **Elastic IP (EIP)** | Static public IP, attachable/detachable from resources. Survives restarts. |
| **Secondary CIDR** | Additional IP range added to a VPC when you run out of space. |

### Security

| Term | Meaning |
|:---|:---|
| **Security Group** | **Stateful** firewall at **instance** level. Default deny. Remembers return traffic. |
| **NACL** (Network ACL) | **Stateless** firewall at **subnet** level. Separate inbound + outbound rules. Must allow both directions explicitly. |

---

## 9. Mental Model

```
VPC (10.0.0.0/16)
 ├── Public Subnet A  (10.0.0.0/17)  ──→ IGW  ──→ Internet
 │     └── EC2 with EIP, ALB
 ├── Private Subnet B (10.0.128.0/18) ──→ NAT GW ──→ Internet (outbound only)
 │     └── RDS, internal services
 └── Private Subnet C (10.0.192.0/19) ──→ NAT GW
       └── Dev workloads

Route Table (public subnets):
  10.0.0.0/16  → local
  0.0.0.0/0    → igw-xxx

Route Table (private subnets):
  10.0.0.0/16  → local
  0.0.0.0/0    → nat-xxx
```

---

## 10. Quick-Reference Formulas

```
Total IPs        = 2^(32 − prefix)
Usable (AWS)     = 2^(32 − prefix) − 5
Subnets from n   = 2^n
Block size       = 256 − partial_octet_value
Partial octet    = 128, 192, 224, 240, 248, 252, 254  (for bits 1–7)
```

---

*Last updated: October 2026*   