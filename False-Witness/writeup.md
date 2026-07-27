# False Witness — Crypto Challenge

**CTF:** Cyber Apocalypse 2026  
**Category:** Crypto  
**Challenge:** False Witness  
**Flag:** `HTB{___l34k1ng_b1ts_0n3_by_0n3___}`

---

## Scenario

Caldrin Vowmark knows that not every seal deserves belief. Some marks still carry the weight of a living vow; others only imitate one well enough to pass a glance. She can question the realm's witnesses as many times as she likes, but certainty, it turns out, is much harder to earn than a convincing lie.

**Target:** `154.57.164.72:31679`

---

## How We Solved It — Reasoning

### The Core Insight

This challenge is a **bit-by-bit oracle leak** attack. The server generates a 256-bit random `KEY`, encrypts the flag with AES-ECB using that key, then gives us an oracle that leaks each bit of `KEY` — but we need to exploit a clever mathematical trick to distinguish bit=0 from bit=1.

### Key Observations

1. **The hash function `H(msg) = pow(G, msg, P)` is modular exponentiation** — and crucially, **we choose G**. The server asks us to provide `G` before generating the keys.

2. **The oracle** returns:
   - A **random number** if `KEY_BITS[i] == 0`
   - One of **two public key values** `PK[i][0]` or `PK[i][1]` (randomly chosen) if `KEY_BITS[i] == 1`
   
   Where `PK[i][j] = pow(G, sk[i][j], P)`

3. **The exploit: choose G = P-1**. Since P is prime, `(P-1)² ≡ 1 (mod P)`, so the element `P-1` has **multiplicative order 2**. This means:
   - `pow(P-1, x, P) = 1` if `x` is even
   - `pow(P-1, x, P) = P-1` if `x` is odd
   
   Therefore, for any `x`, `H(x) = pow(P-1, x, P)` is **always 1 or P-1**.

4. **Distinguishing bits**: When `KEY_BITS[i] == 1`, the oracle returns either `1` or `P-1` (always one of these two values). When `KEY_BITS[i] == 0`, the oracle returns a uniformly random value from `[0, 2²⁵⁶)`. The probability that a random value happens to be exactly `1` or `P-1` is `2/2²⁵⁶ ≈ 0` — effectively impossible. So:
   - Result is `1` or `P-1` → bit is `1`
   - Result is anything else → bit is `0`

5. **Once we have all 256 KEY_BITS**, we reconstruct the full 32-byte `KEY` and decrypt the AES-ECB-encrypted flag.

---

## Exploitation

### Solution Script

```python
#!/usr/bin/env python3
from pwn import *
from Cryptodome.Util.Padding import unpad
from Cryptodome.Cipher import AES
from hashlib import sha256

P = 0xCD4A96D3B7FA7251A1BB765933FB676FCAE8C9026682E34F779122DFD66915BB
N = sha256().digest_size * 8  # 256

HOST = "154.57.164.72"
PORT = 31679

r = remote(HOST, PORT)

# Step 1: Get the encrypted flag
r.recvline()  # "Here is something for you:"
enc_flag_hex = r.recvline().decode().strip()
enc_flag = bytes.fromhex(enc_flag_hex)
print(f"Encrypted flag: {enc_flag_hex}")

# Step 2: Provide G = P-1 (generator with order 2)
G = P - 1
r.sendlineafter(b"Before we start, give me the hashing generator: ", str(G).encode())

# Step 3: Recover KEY_BITS via the oracle
KEY_BITS = []
for i in range(N):
    r.sendline(b"1")
    r.recvuntil(b"Enter offset: ")
    r.sendline(str(i).encode())
    resp = r.recvline().decode().strip()
    val = int(resp.split(":")[-1].strip())
    if val == 1 or val == P - 1:
        KEY_BITS.append(1)
    else:
        KEY_BITS.append(0)

# Step 4: Reconstruct KEY
KEY_bytes = int("".join(str(b) for b in KEY_BITS), 2).to_bytes(32, 'big')
print(f"Recovered KEY: {KEY_bytes.hex()}")

# Step 5: Decrypt the flag
flag = unpad(AES.new(KEY_bytes, AES.MODE_ECB).decrypt(enc_flag), 16)
print(f"FLAG: {flag.decode()}")

r.close()
```

### Execution

```
$ python3 solve.py
[+] Opening connection to 154.57.164.72 on port 31679: Done
[*] Server says: Here is something for you:
[*] Encrypted flag: f43f3bdb882e5c68eadb94e9411e115ffaf97deac9eaf4b4e20920b6a6c607f751f0f4dbf68ed1e8023027a55c393b41
[*] Recovered KEY: ce0d3f15b8343099d46fbc93d057e7f0a4d49210c8977edef00d7c4fddbf1a39
[+] FLAG: HTB{___l34k1ng_b1ts_0n3_by_0n3___}
```

---

## Flag

| Item | Value |
|------|-------|
| **Flag** | `HTB{___l34k1ng_b1ts_0n3_by_0n3___}` |
| **Category** | Crypto |
| **Method** | Bit-by-bit oracle extraction via small-order generator (G = P-1, order 2) |

---

## Security Implications

The vulnerability here is that **the user controls the hash function generator G**. The challenge intends `H(msg) = pow(G, msg, P)` to act as a one-way function (discrete log), but by choosing `G = P-1` (an element of order 2), we completely break the "hash" — any input maps to one of only 2 possible outputs. This causes the oracle's "bit is 1" responses to be trivially distinguishable from random.

In a real system:
- **Never let the user control cryptographic parameters** that affect security properties
- Hash functions based on modular exponentiation must use a **generator of the full group** (or at least a group with large enough order that the discrete log is hard and the output space is large)
- **Distinguishability oracles** that leak whether a bit is pseudorandom vs truly random are dangerous — combine with timing or output analysis they can leak entire keys bit-by-bit

---

## Caveats

1. **Handshake pattern**: The server sends `"Here is something for you:"` on one line, then the encrypted hex on the next line. Must handle both reads separately.
2. **Oracle caching**: The oracle caches its result per offset. This doesn't affect our attack since we only need one query per offset.
3. **pwntools vs Crypto**: On Kali, `pycryptodome` is the installed package (not `pycrypto`), so import paths use `Cryptodome.Cipher` and `Cryptodome.Util.Padding`.
4. **256 queries**: Each query is a round trip, so the full recovery takes some seconds. No rate limiting on the oracle.