# TCP 3-Way Handshake & Connection Lifecycle — Deep Dive

The **TCP 3-Way Handshake** (also known as `SYN-SYN-ACK`) is the mechanism used by the Transmission Control Protocol (TCP) to establish a reliable, full-duplex connection over an unreliable IP network before any application data is exchanged.

---

## 1. Why a 3-Way Handshake?

IP is a connectionless, best-effort protocol that does not guarantee delivery, ordering, or packet integrity. TCP solves this by establishing state before communication begins.

The handshake accomplishes three core goals:
1. **Synchronize Initial Sequence Numbers (ISNs):** Both client and server must agree on starting byte numbers so data can be reordered and duplicates eliminated.
2. **Verify Bidirectional Reachability:** It proves that Client $\to$ Server and Server $\to$ Client network paths are both functioning.
3. **Negotiate Options & Parameters:** Maximum Segment Size (MSS), Selective Acknowledgments (SACK), and Window Scaling are established during this exchange.

---

## 2. Step-by-Step Handshake Diagram

```text
       Client                                                    Server
  (State: CLOSED)                                           (State: LISTEN)
         │                                                         │
         │  1. SYN [Seq = x, Flags = SYN]                          │
         ├────────────────────────────────────────────────────────▶│ (State: SYN-RECEIVED)
         │                                                         │
  (State: SYN-SENT)                                                │
         │  2. SYN-ACK [Seq = y, Ack = x + 1, Flags = SYN, ACK]     │
         │◀────────────────────────────────────────────────────────┤
         │                                                         │
(State: ESTABLISHED)                                               │
         │  3. ACK [Seq = x + 1, Ack = y + 1, Flags = ACK]         │
         ├────────────────────────────────────────────────────────▶│
         │                                                  (State: ESTABLISHED)
         │                                                         │
         │ ═══════════════ DATA TRANSFER PHASE ═══════════════════ │
         │   Client & Server exchange payload data via TCP packets │
```

---

## 3. Detailed Step Breakdown

### Step 1: SYN (Synchronize)
- **Sender:** Client $\to$ Server
- **State Change:** Client transitions from `CLOSED` to `SYN-SENT`.
- **Key Fields in Header:**
  - `Flags`: `SYN = 1`, `ACK = 0`
  - `Sequence Number (Seq)`: Set to random Initial Sequence Number $x$ (Client ISN).
  - `Options`: Advertises Client MSS, Window Scale factor, and SACK support.
- **Meaning:** *"I want to open a connection. My bytes will start counting from $x$."*

---

### Step 2: SYN-ACK (Synchronize-Acknowledge)
- **Sender:** Server $\to$ Client
- **State Change:** Server transitions from `LISTEN` to `SYN-RECEIVED`.
- **Key Fields in Header:**
  - `Flags`: `SYN = 1`, `ACK = 1`
  - `Sequence Number (Seq)`: Set to random Initial Sequence Number $y$ (Server ISN).
  - `Acknowledgment Number (Ack)`: Set to $x + 1$ (acknowledging receipt of client's SYN).
- **Meaning:** *"I received your SYN. My bytes will start counting from $y$, and I expect your first data byte at $x + 1$."*

---

### Step 3: ACK (Acknowledge)
- **Sender:** Client $\to$ Server
- **State Change:** 
  - Client transitions from `SYN-SENT` to `ESTABLISHED`.
  - Server transitions from `SYN-RECEIVED` to `ESTABLISHED` upon receipt.
- **Key Fields in Header:**
  - `Flags`: `SYN = 0`, `ACK = 1`
  - `Sequence Number (Seq)`: $x + 1$
  - `Acknowledgment Number (Ack)`: $y + 1$
- **Payload:** This final ACK packet **can carry application data** (e.g., HTTP `GET` request or TLS Client Hello).

---

## 4. Key TCP Header Flags & Fields

| Flag / Field | Meaning | Handshake Role |
|---|---|---|
| `SYN` | Synchronize | Request connection; sync starting sequence numbers |
| `ACK` | Acknowledgment | Confirms receipt of bytes up to `Ack - 1` |
| `FIN` | Finish | Polite connection teardown initiator |
| `RST` | Reset | Immediate, forceful termination / port closed signal |
| `ISN` | Initial Sequence Number | Random 32-bit integer generated to prevent spoofing/collisions |
| `MSS` | Maximum Segment Size | Largest unfragmented TCP payload size either side accepts |

---

## 5. Security Threat: SYN Flood & Mitigations

### How a SYN Flood Works
An attacker sends a high volume of spoofed `SYN` packets without completing Step 3 (`ACK`). The server allocates memory in its **SYN backlog queue** for each half-open connection until the queue is exhausted, rejecting legitimate traffic.

```text
Attacker (Spoofed IPs)                    Target Server
     SYN  ──────────────────────────────────────▶ (Allocates Half-Open Entry)
     SYN  ──────────────────────────────────────▶ (Allocates Half-Open Entry)
     SYN  ──────────────────────────────────────▶ (Allocates Half-Open Entry)
    (Never sends final ACK)                       Queue Full: Drops new clients!
```

### Defenses & Mitigations

1. **SYN Cookies (`syncookies`):**
   Encodes connection state into the server's own `ISN` rather than saving state in RAM. State is only allocated if the client returns a valid final `ACK`.
   ```bash
   # Enable TCP SYN Cookies in Linux
   sudo sysctl -w net.ipv4.tcp_syncookies=1
   ```

2. **Tune SYN Backlog & Retries:**
   ```bash
   # Increase backlog queue size
   sudo sysctl -w net.ipv4.tcp_max_syn_backlog=4096

   # Reduce SYN-ACK retry attempts (default is usually 5)
   sudo sysctl -w net.ipv4.tcp_synack_retries=2
   ```

---

## 6. Connection Teardown: 4-Way FIN Handshake

Terminating a TCP connection is handled independently in each direction because TCP is full-duplex.

```text
       Client                                                    Server
(State: ESTABLISHED)                                       (State: ESTABLISHED)
         │                                                         │
         │  1. FIN [Seq = u, Flags = FIN, ACK]                     │
         ├────────────────────────────────────────────────────────▶│ (State: CLOSE-WAIT)
(State: FIN-WAIT-1)                                                │
         │  2. ACK [Ack = u + 1, Flags = ACK]                      │
         │◀────────────────────────────────────────────────────────┤
(State: FIN-WAIT-2)                                                │ (Flushes remaining data)
         │                                                         │
         │  3. FIN [Seq = v, Ack = u + 1, Flags = FIN, ACK]        │
         │◀────────────────────────────────────────────────────────┤ (State: LAST-ACK)
         │                                                         │
(State: TIME-WAIT)                                                 │
         │  4. ACK [Ack = v + 1, Flags = ACK]                      │
         ├────────────────────────────────────────────────────────▶│ (State: CLOSED)
         │                                                         │
   [Wait 2 × MSL]                                                  │
(State: CLOSED)                                                    │
```

### Why `TIME-WAIT` (2MSL)?
The initiating host stays in `TIME-WAIT` for **$2 \times \text{MSL}$** (Maximum Segment Lifetime, typically 60 seconds) to:
1. Ensure the final `ACK` arrived at the remote peer (re-transmitting if the peer resends its `FIN`).
2. Allow old duplicate packets lingering in transit across routers to expire before the same socket pair (`IP:Port`) is reused.

---

## 7. Inspecting Handshakes with Linux CLI Tools

### Capture Handshake Packets with `tcpdump`

```bash
# Capture full handshake on port 80/443
sudo tcpdump -nn -i any 'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0' and port 443
```

### Inspect Socket States with `ss`

```bash
# Count connections grouped by TCP state
ss -ant | awk '{print $1}' | sort | uniq -c

# Show sockets currently in SYN-RECV or TIME-WAIT state
ss -t state syn-recv
ss -t state time-wait
```

---

## 8. Quick Reference Summary

| Phase | Direction | Flags Set | Sequence / Acknowledgment Math |
|---|---|---|---|
| Step 1 | Client $\to$ Server | `SYN` | `Seq = x`, `Ack = 0` |
| Step 2 | Server $\to$ Client | `SYN, ACK` | `Seq = y`, `Ack = x + 1` |
| Step 3 | Client $\to$ Server | `ACK` | `Seq = x + 1`, `Ack = y + 1` |
| Teardown 1 | Initiator $\to$ Peer | `FIN, ACK` | `Seq = u`, `Ack = Ack` |
| Teardown 2 | Peer $\to$ Initiator | `ACK` | `Ack = u + 1` |
| Teardown 3 | Peer $\to$ Initiator | `FIN, ACK` | `Seq = v`, `Ack = u + 1` |
| Teardown 4 | Initiator $\to$ Peer | `ACK` | `Ack = v + 1` |