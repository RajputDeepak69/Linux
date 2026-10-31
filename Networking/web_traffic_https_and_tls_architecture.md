# How Web Traffic Reaches a Domain, Why HTTPS Exists, and How TLS Secures Data

A complete technical breakdown of what happens under the hood when a client navigates to a domain, why standard HTTP was replaced by HTTPS, and how TLS protects data across modern networks.

---

## Part 1: How Traffic Reaches a Domain

When you enter a URL like `https://example.com` into a web browser, the system executes an automated multi-step sequence spanning DNS resolution, IP routing, TCP transport establishment, TLS negotiation, and HTTP application delivery.

```
┌───────────┐      1. DNS Resolution      ┌─────────────┐
│  Browser  │ ──────────────────────────▶ │  DNS Server │
│ (Client)  │ ◀────────────────────────── │  (Resolver) │
└─────┬─────┘     Returns IP Address      └─────────────┘
      │
      │ 2. TCP 3-Way Handshake (Port 443)
      │ ────────────────────────────────────────────────────────┐
      │                                                         │
      │ 3. TLS Cryptographic Handshake                          │
      │ ──────────────────────────────────────────────────┐     │
      │                                                   │     │
      │ 4. Encrypted HTTP Request (GET / HTTP/2 or 3)    │     │
      ▼                                                   ▼     ▼
┌───────────────────────────────────────────────────────────────┐
│               Edge / Gateway / Server (example.com)           │
│   Load Balancer ──▶ Reverse Proxy (Nginx) ──▶ Web Backend     │
└───────────────────────────────────────────────────────────────┘
```

---

### Step 1: DNS Resolution (Finding the IP Address)

Computers communicate using IP addresses (e.g., `93.184.216.34`), but humans remember domain names. The first step is resolving `example.com` into an IP:

1. **Local Cache Check:**
   - The browser inspects its internal DNS cache.
   - If not found, the Operating System checks its local DNS cache and hosts file (`/etc/hosts`).
2. **Recursive DNS Resolver:**
   - The OS queries the configured DNS resolver (e.g., local ISP router, `8.8.8.8`, or `1.1.1.1`).
3. **Iterative Hierarchy Walk:**
   If the resolver doesn't have the entry cached:
   - Queries a **Root DNS Server** (`.`): Points to Top-Level Domain (TLD) servers.
   - Queries the **TLD DNS Server** (`.com`): Points to the authoritative name servers for `example.com`.
   - Queries the **Authoritative DNS Server**: Returns the actual IP address (via `A` record for IPv4 or `AAAA` for IPv6).

---

### Step 2: Routing Across the Internet (BGP & IP Routing)

Once the client has the destination IP:

- The operating system encapsulates packets with source and destination IP headers.
- Packets leave the local network interface, passing through default gateways and ISP border routers.
- Routers use **BGP (Border Gateway Protocol)** and autonomous system (AS) path algorithms to route packets across fiber backbones and internet exchange points (IXPs) to the data center or Content Delivery Network (CDN) hosting the site.

---

### Step 3: Establishing the Transport Layer (TCP Handshake)

Before application data can flow, TCP establishes a reliable, bidirectional connection:

- **Client $\to$ Server:** `SYN` (Initiates connection with sequence number $X$)
- **Server $\to$ Client:** `SYN-ACK` (Acknowledges $X+1$, sends sequence number $Y$)
- **Client $\to$ Server:** `ACK` (Acknowledges $Y+1$)

Traffic is now connected to port `443` (standard HTTPS).

---

### Step 4: Establishing the TLS Secure Tunnel

Once TCP is established, the client and server execute the TLS handshake to verify identity and generate session keys (covered in detail in Part 3).

---

### Step 5: Application Delivery (HTTP Request & Response)

Inside the encrypted TLS tunnel:

1. The client sends an HTTP request:
   ```http
   GET /index.html HTTP/2
   Host: example.com
   User-Agent: Mozilla/5.0 ...
   ```
2. The edge infrastructure routes the request:
   - **Load Balancers (ALB, HAProxy):** Distribute incoming traffic across instances.
   - **Reverse Proxies (Nginx, Envoy):** Terminate TLS and route to application workers.
   - **Backend Application (Node.js, Go, Python, Java):** Processes business logic and database queries.
3. The server generates an HTTP response (e.g., `200 OK`) and returns assets (HTML, CSS, JS, images) back through the encrypted session.

---

## Part 2: Why HTTPS Exists

Standard HTTP transmits data across the internet in plain, unencrypted text.

```
Plain HTTP:
Client ───────[ GET /account (Password: Secret123) ]───────▶ Server
                         ▲
                         │ Unencrypted packet visible to:
                         ├── Public Wi-Fi snooper
                         ├── Malicious ISP / Router
                         └── Network wiretaps
```

---

### Vulnerabilities of Plain HTTP

| Threat | Plain HTTP Reality | Attack Type |
|---|---|---|
| **Eavesdropping** | Anyone on the network path can read session tokens, passwords, cookies, and personal data. | Packet sniffing, Wi-Fi snooping |
| **Tampering** | Intermediaries can alter responses in transit, injecting malicious JavaScript, advertisements, or fake downloads. | Man-in-the-Middle (MITM) |
| **Impersonation** | No identity verification. A malicious router can redirect you to a fake banking IP without your knowledge. | DNS spoofing / ARP poisoning |

---

### Enter HTTPS (`HTTP Secure`)

**HTTPS is not a separate protocol.** It is simply standard HTTP traffic wrapped inside an encrypted cryptographic tunnel provided by **TLS (Transport Layer Security)**:

```
┌───────────────────────────────────────┐
│        Application: HTTP / HTTP/2     │
├───────────────────────────────────────┤
│        Security: TLS Layer            │  <-- Provides Encryption, Integrity & Auth
├───────────────────────────────────────┤
│        Transport: TCP (Port 443)      │
├───────────────────────────────────────┤
│        Network: IP                    │
└───────────────────────────────────────┘
```

By encrypting the channel:
- **Passersby see only random noise:** Eavesdroppers can see that you are connected to an IP address, but cannot see URLs, parameters, cookies, or body content.
- **Payloads cannot be modified:** Any alteration of data en route invalidates cryptographic integrity checks.
- **Server identities are verified:** Web browsers confirm that the domain belongs to the entity holding the signed certificate.

---

## Part 3: How TLS Secures Data

TLS protects web communication by combining asymmetric cryptography, symmetric cryptography, and digital identity verification.

---

### 1. The Core Guarantees: The CIA Triad

```
                        TLS Security Pillars
            ┌─────────────────────┼─────────────────────┐
            ▼                     ▼                     ▼
     Confidentiality          Integrity            Authenticity
  (Symmetric Encryption)   (Cryptographic MAC)   (Digital Certificates)
```

1. **Confidentiality:** Scrambles data into unreadable ciphertext using AES-GCM or ChaCha20-Poly1305.
2. **Integrity:** Appends an authentication tag (AEAD) to every packet. If an attacker changes a single bit, the receiver rejects the packet.
3. **Authenticity:** Uses Public Key Infrastructure (PKI) and Certificate Authorities (CAs) so clients know they are communicating with the actual domain owner.

---

### 2. How the TLS 1.3 Handshake Works

TLS 1.3 optimizes security and speed, completing key exchange and certificate verification in just **one round trip (1-RTT)**:

```text
       Client                                                    Server
  (TCP Established)                                         (TCP Established)
         │                                                         │
         │  1. Client Hello                                        │
         │     - Supported Cipher Suites (e.g., TLS_AES_256_GCM)   │
         │     - Ephemeral Diffie-Hellman Key Share ($Client_{pub}$)│
         │     - Server Name Indication (SNI: example.com)         │
         ├────────────────────────────────────────────────────────▶│
         │                                                         │
         │  2. Server Hello                                        │
         │     - Chosen Cipher Suite                               │
         │     - Ephemeral Diffie-Hellman Key Share ($Server_{pub}$)│
         │     - Server Digital Certificate (Public Identity)      │
         │     - Encrypted Handshake Signature                     │
         │◀────────────────────────────────────────────────────────┤
         │                                                         │
   [Verify Certificate]                                            │
   [Derive Symmetric Key]                                    [Derive Symmetric Key]
         │                                                         │
         │  3. Finished + Encrypted HTTP Request                   │
         ├────────────────────────────────────────────────────────▶│
         │                                                         │
         │ ══════════════ SECURE SYMMETRIC SESSION ═══════════════ │
         │   Client and Server exchange data using AES-GCM key     │
```

---

### 3. Step-by-Step Breakdown

#### A. Cryptographic Key Derivation (Diffie-Hellman)
- Rather than transmitting passwords or symmetric keys across the network, the client and server exchange public parameters ($Client_{pub}$ and $Server_{pub}$).
- Using the **ECDHE (Elliptic Curve Diffie-Hellman Ephemeral)** mathematical principle, both sides independently calculate the exact same **shared symmetric session key**.
- An attacker recording all network traffic cannot compute this key without solving computationally infeasible mathematical problems.

#### B. Certificate & Identity Verification (PKI)
- The server presents an **X.509 Digital Certificate** containing:
  - Subject domain name (`example.com`)
  - Server's long-term public key
  - Expiration date
  - Digital signature from an accredited **Certificate Authority (CA)** (e.g., Let's Encrypt, DigiCert)
- The client verifies this signature using the CA public keys already built into the operating system or browser trust store.

#### C. Bulk Data Encryption (Symmetric Session)
- Once the handshake completes, asymmetric math is set aside.
- All HTTP requests, cookies, and responses are encrypted and decrypted using the lightweight, hardware-accelerated symmetric session key.

---

## Summary Comparison

| Concept | Plain HTTP | HTTPS / TLS |
|---|---|---|
| **Default Port** | `80` | `443` |
| **Wire Payload** | Plain text | Ciphertext |
| **Authentication** | None | Verified via X.509 Certificates |
| **Tamper Detection** | None (easily spoofed) | Cryptographic MAC / AEAD validation |
| **Protection** | Insecure | Confidentiality, Integrity, Authenticity |

```text
Traffic Flow:
User ──▶ DNS (IP Lookup) ──▶ TCP (Port 443) ──▶ TLS (Tunnel) ──▶ HTTPS (Data Delivery)
```