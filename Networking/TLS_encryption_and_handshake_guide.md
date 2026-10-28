# TLS Encryption & Handshake (Concept)

A comprehensive guide explaining the TLS handshake, cryptographic mechanics, and practical verification using OpenSSL.

---

## TLS Encryption & Handshake (Concepts)

### 1. What TLS Protects: The CIA Triad

Transport Layer Security (TLS) operates above TCP to secure application-layer protocols (such as HTTPS, FTPS, and SMTPS). It guarantees security by addressing all three pillars of the **CIA Triad**:

| CIA Pillar | What It Protects Against | How TLS Enforces It |
|---|---|---|
| **Confidentiality** | Eavesdropping, packet sniffing, man-in-the-middle data inspection | **Symmetric Encryption (AES-GCM, ChaCha20-Poly1305):** Encrypts all transmitted application payload. Anyone intercepting the traffic sees only ciphertext. |
| **Integrity** | Packet tampering, payload injection, bit-flipping | **Message Authentication Codes (MAC / AEAD):** Uses cryptographic hashes (such as HMAC-SHA256 or AEAD auth tags) to verify that data has not been modified or corrupted in transit. |
| **Availability / Authenticity** | Impersonation, spoofing, untrusted server routing | **Digital Certificates & PKI (X.509):** The server proves its identity via a certificate signed by a trusted Certificate Authority (CA), ensuring clients only connect to the authentic endpoint. |

```text
               TLS Security Guarantees
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
  Confidentiality      Integrity        Authenticity
(Ciphertext Only)   (Tamper-Proof)   (Verified Identity)
```

---

### 2. Why Symmetric Encryption is Used After the Handshake

During the initial handshake, TLS uses **asymmetric encryption** (public/private key pairs or Diffie-Hellman ephemeral exchanges). Once the handshake finishes, all subsequent application traffic is encrypted using **symmetric encryption** (a shared secret key).

#### Why not use Asymmetric Encryption for everything?
1. **Computational Overhead & Performance:**
   - Asymmetric cryptographic algorithms (like RSA or ECC) require heavy mathematical computations involving large prime numbers and modular exponentiation.
   - Symmetric algorithms (like AES-NI hardware-accelerated instructions) are **100x to 1,000x faster** than asymmetric operations.
2. **CPU & Battery Consumption:**
   - Encrypting gigabytes of streaming video, database queries, or file downloads with asymmetric ciphers would exhaust server CPU capacity and drain client batteries.
3. **Data Size Limitations:**
   - Asymmetric ciphers can only encrypt blocks smaller than their key length (e.g., a 2048-bit RSA key can encrypt at most ~245 bytes of raw payload).

```text
                   Hybrid Encryption Model
┌─────────────────────────────────┐     ┌──────────────────────────────────┐
│        TLS Handshake Phase      │     │      Data Transmission Phase     │
│       Asymmetric Encryption     │ ──▶ │       Symmetric Encryption       │
│  - Authenticate server identity │     │  - Fast, bulk data streaming     │
│  - Securely exchange shared key │     │  - Low CPU usage (AES / ChaCha)  │
└─────────────────────────────────┘     └──────────────────────────────────┘
```

---

### 3. Role of Public & Private Keys

TLS leverages public-key cryptography (asymmetric keys) primarily during the handshake phase:

```text
       Client                                                    Server
         │                                                         │
         │ ──────────── Client Hello (Supported Ciphers) ─────────▶│
         │                                                         │
         │ ◀─────────── Server Hello + Public Certificate ─────────┤
         │              (Contains Server's Public Key)             │
         │                                                         │
   [Verify Cert with CA]                                           │
   [Extract Public Key]                                            │
         │                                                         │
         │ ──────────── Key Exchange / Pre-Master Secret ─────────▶│
         │              (Encrypted with Public Key OR              │
         │               Derived via Ephemeral ECDHE)              │
         │                                                         │
         │                                             [Decrypts/Derives using]
         │                                             [Private Key in RAM]
         ▼                                                         ▼
   Both compute the identical Symmetric Session Key independently!
```

#### The Public Key
- **Distributed freely** inside the server's digital X.509 certificate.
- Used by the client to:
  1. Verify the digital signature of the Certificate Authority (authenticating the certificate).
  2. Encrypt parameters sent to the server (or verify ephemeral signatures in TLS 1.3 ECDHE).
- Anyone can read and hold this key without compromising security.

#### The Private Key
- **Kept strictly confidential** on the server (never leaves the server's filesystem or HSM).
- Used by the server to:
  1. Decrypt data encrypted with its public key.
  2. Sign ephemeral handshake parameters to prove it is the legitimate owner of the certificate.
- If compromised, attackers can impersonate the server and potentially decrypt intercepted sessions.

---
