# TLS Encryption & Handshake (Concept & Practical)

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

## TLS Handshake Version Check (Practical)

### Step 1: Execute the OpenSSL Inspection Command

Run `openssl s_client` against any HTTPS target:

```bash
openssl s_client -connect google.com:443
```

> **Tip for quick CLI output:** To exit automatically after the handshake rather than leaving an interactive session open, pipe an empty string:
> ```bash
> echo "" | openssl s_client -connect google.com:443
> ```

---

### Step 2: What to Observe in the Output

The output contains diagnostic information about the negotiated session. Pay specific attention to these three areas:

```text
CONNECTED(00000003)
---
Certificate chain
 0 s:CN = *.google.com
   i:C = US, O = Google Trust Services, CN = WR2
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: ecdsa-with-SHA256
   v:NotBefore: Feb 10 08:30:00 2026 GMT; NotAfter: May 05 08:30:00 2026 GMT
 1 s:C = US, O = Google Trust Services, CN = WR2
   i:C = US, O = Google Trust Services LLC, CN = GTS Root R4
 2 s:C = US, O = Google Trust Services LLC, CN = GTS Root R4
   i:C = US, O = Root CA, CN = GlobalSign
---
Server certificate
-----BEGIN CERTIFICATE-----
MIICljCCAX6gAwIBAgIRAM...
-----END CERTIFICATE-----
subject=CN = *.google.com
issuer=C = US, O = Google Trust Services, CN = WR2
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: ECDSA
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 3820 bytes and written 390 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 256 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
```

---

### Step 3: Breakdown of Inspected Fields

#### 1. Protocol Version
```text
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
```
- **Observed:** `TLSv1.3` (or `TLSv1.2` depending on client/server support).
- **Meaning:** Shows the highest mutually supported TLS version agreed upon during the handshake. TLS 1.3 reduces handshake latency from 2 Round-Trip Times (2-RTT) to 1-RTT and deprecates insecure legacy algorithms.

#### 2. Cipher Suite
```text
Cipher is TLS_AES_256_GCM_SHA384
```
This string specifies the exact algorithms chosen to secure the session:
- **`TLS`**: Protocol.
- **`AES_256_GCM`**: Symmetric bulk encryption algorithm (256-bit Advanced Encryption Standard in Galois/Counter Mode for authenticated encryption).
- **`SHA384`**: Secure Hash Algorithm used for key derivation and integrity checks.

*(In TLS 1.2, cipher suites also explicitly name key exchange and authentication, e.g., `ECDHE-RSA-AES256-GCM-SHA384`).*

#### 3. Certificate Chain
```text
Certificate chain
 0 s:CN = *.google.com                          <-- Leaf / Server Certificate
   i:C = US, O = Google Trust Services, CN = WR2
 1 s:C = US, O = Google Trust Services, CN = WR2 <-- Intermediate CA
   i:C = US, O = Google Trust Services LLC, CN = GTS Root R4
 2 s:C = US, O = Google Trust Services LLC, CN = GTS Root R4 <-- Root CA
```
- `s:` (Subject): The entity being identified by this certificate.
- `i:` (Issuer): The Certificate Authority that signed and verified this certificate.
- **Chain of Trust:** The Leaf certificate is signed by an Intermediate CA, which is signed by a Root CA pre-installed in your operating system's trust store.
- **Verification Return Code:** `Verify return code: 0 (ok)` confirms that the cryptographic signature chain is valid and untampered.

---

## Quick Reference CLI Filters

Extract specific details directly using grep and OpenSSL:

```bash
# Check negotiated protocol version only
echo "" | openssl s_client -connect google.com:443 2>/dev/null | grep -i "Protocol"

# Check negotiated cipher suite only
echo "" | openssl s_client -connect google.com:443 2>/dev/null | grep -i "Cipher"

# Inspect certificate expiration dates
echo "" | openssl s_client -connect google.com:443 2>/dev/null | openssl x509 -noout -dates

# Check certificate issuer and subject
echo "" | openssl s_client -connect google.com:443 2>/dev/null | openssl x509 -noout -issuer -subject
```