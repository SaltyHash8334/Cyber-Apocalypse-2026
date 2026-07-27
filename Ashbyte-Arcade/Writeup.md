# Ashbyte Arcade — Crypto

**Category:** Crypto | **Event:** Cyber Apocalypse 2026 | **Flag:** `HTB{tw0_r0und5_0f_5n0w_4nd_1mp0551bl3_d1ff5}`

## Overview

An arcade cabinet contains encrypted save states (death saves). We get 16 lives (ciphertexts of known plaintexts) per session and unlimited decryption oracle queries. The cipher is "SnowCipher" — a 2-round AES-128 variant. The goal: forge a ciphertext that decrypts to the winning state (sector=9, x=18, y=13, score≥10M, specific runes).

**Target:** `154.57.164.76:31189`  
**Tools:** C (gcc), Python, custom brute-force solver

## Challenge Analysis

### The State Format (16 bytes)

| Offset | Field | Description |
|--------|-------|-------------|
| 0 | SIG | 0x4E (fixed) |
| 1 | HP | 1-9 |
| 2 | SECTOR | 1-9 |
| 3 | X | 1-18 |
| 4 | Y | 1-13 |
| 5-7 | SCORE | 24-bit unsigned |
| 8-11 | RUNES | 4 bytes XORing to 0xA7 |
| 12 | TRACK | `(sector*19 + x*7 + y*13 + score_lo) & 0xFF` |
| 13 | DEATHS | 0-31 |
| 14 | FLAGS | `(hp + sector + x + y + track) & 0xFF` |
| 15 | LRC | XOR of bytes 0-14 |

### The SnowCipher

A custom 2-round AES-128:
```
Round 0: AddRoundKey(K0)
Round 1: SubBytes → ShiftRows → MixColumns → AddRoundKey(K1)
Round 2: SubBytes → ShiftRows → MixColumns → AddRoundKey(K2)
```
No final SubBytes/ShiftRows — identical to standard AES inner rounds. The key schedule is standard AES-128.

### API Endpoints

- `POST /api/start` — new session with 16 lives and random 16-byte key
- `POST /api/death` — submit valid state → get encrypted ciphertext (costs 1 life)
- `POST /api/load` — submit ciphertext → if decrypts to winning state, get flag

### Key Constraint

We only get 16 known (plaintext, ciphertext) pairs per session (one per life). Each session has a different random key. The `/api/load` endpoint is an unlimited decryption oracle.

## Attack Strategy: Group-Level Brute-Force

### Core Insight

For a 2-round AES, the intermediate state `state5 = IB(ISR(IMC(C ^ rk2)))` equals `state4 ^ rk1 = MC(SR(SB(P ^ rk0))) ^ rk1`.

The **difference** between two pairs eliminates `rk1`:
```
state5_i ^ state5_j = state4_i ^ state4_j
```

Each byte of `state4` depends on exactly 4 specific bytes of `rk0`. This partitions `rk0` into 4 independent groups of 4 bytes each:

| Group | state4 byte | rk0 bytes | Affected by |
|-------|-------------|-----------|-------------|
| 0 | byte 0 | k[0], k[5], k[10], k[15] | pt[0], pt[5], pt[10], pt[15] |
| 1 | byte 5 | k[4], k[9], k[14], k[3] | pt[4], pt[9], pt[14], pt[3] |
| 2 | byte 10 | k[8], k[13], k[2], k[7] | pt[8], pt[13], pt[2], pt[7] |
| 3 | byte 15 | k[12], k[1], k[6], k[11] | pt[12], pt[1], pt[6], pt[11] |

### The Key Equation

For byte 0 of state5:
```
ISB[imc_ct_i[0] ^ imc_rk2[0]] ^ ISB[imc_ct_j[0] ^ imc_rk2[0]] = 
  2*(SB[pt_i[0]^k0] ⊕ SB[pt_j[0]^k0]) 
  ⊕ 3*(SB[pt_i[5]^k5] ⊕ SB[pt_j[5]^k5]) 
  ⊕ (SB[pt_i[10]^k10] ⊕ SB[pt_j[10]^k10]) 
  ⊕ (SB[pt_i[15]^k15] ⊕ SB[pt_j[15]^k15])
```

Where `imc_ct` = IMC(ct[0:4]) and `imc_rk2` = IMC(rk2_row0), precomputed once.

The RHS depends on 4 rk0 bytes; the LHS depends on 1 IMC byte of rk2 row 0 and the ciphertexts.

### Brute-Force Algorithm

For each group g (0-3):
1. Precompute: for each x ∈ [0,255]: diff[x] = ISB[imc_ct_0[g]^x] ^ ISB[imc_ct_1[g]^x]
2. Build inverse lookup: for each diff value D, which x values produce it
3. For each of 2^32 rk0 group values:
   - Compute expected diff from the plaintexts
   - Look up matching x value
   - Verify against all 15 pair-difference equations
4. Output: correct rk0 bytes + imc byte

This takes ~45-60 seconds per group in C.

### Cancellation Effect

Byte 0 of all plaintexts is SIG = 0x4E (constant). This means k[0] **cancels** in all diff equations — any k[0] value produces the same diff. Similarly, byte 4 (Y position) was constant across our test states, so k[4] was also unconstrained.

### Recovering the Full Key

After all 4 groups:
- Group 0: k[5]=05, k[10]=0b, k[15]=2a, imc[0]=b5 (k[0] unconstrained)
- Group 1: k[9]=5c, k[14]=93, k[3]=f4, imc[1]=a6 (k[4] unconstrained)  
- Group 2: k[8]=a7, k[13]=b1, k[7]=e3, imc[2]=b6 (k[2]=99 or 9a)
- Group 3: k[12]=a1, k[1]=28, k[6]=0e, k[11]=58, imc[3]=c9

Total candidates: 256 (k0) × 2 (k2) × 256 (k4) = 131,072. Each verified against 2 known P-C pairs in under 10 seconds.

## Solution Code

The key solver is a C program that brute-forces each rk0 group (2^32 iterations) in ~50 seconds, using precomputed IMC of ciphertexts and inverse S-box lookups.

## Flag

```
HTB{tw0_r0und5_0f_5n0w_4nd_1mp0551bl3_d1ff5}
```

## How We Solved It — Reasoning

1. **Identified the cipher** as a 2-round AES variant by analyzing the S-box, MixColumns, ShiftRows, and key schedule — all standard AES components.
2. **Found the independent-group structure**: each state4 byte depends on only 4 key bytes (due to row-major matrix layout and MixColumns operating on rows).
3. **Leveraged the difference equation**: `state5_i ^ state5_j = state4_i ^ state4_j` eliminates the unknown rk1 round key.
4. **Precomputed IMC of ciphertexts**: since IMC is linear, `IMC(ct ^ rk2) = IMC(ct) ^ IMC(rk2)`, making the inner loop just XOR + ISB lookup.
5. **Inverse-lookup optimization**: precomputed which x values produce each diff value, reducing the inner loop to a table lookup instead of iterating 256 values per rk0 guess.
6. **Dealt with cancellation**: k[0] is unconstrained because SIG=0x4E is constant across all states; k[4] was accidentally constant (Y=1). These were brute-forced in a final pass (~131K candidates) verified against known P-C pairs.
7. **Encrypted the winning state** and submitted to `/api/load` to receive the flag.

## Caveats

- The Y coordinate generation had a bug: `((i * 13) % 13) + 1 = 1` for all i, making Y constant. If Y varied, k[4] would have been uniquely determined.
- The sector had only 2 values, giving 2 candidates for k[2].
- We only used the first byte of the ciphertext's IMC (byte 0), which depends on all 4 bytes of rk2 row 0. The remaining IMC bytes were determined by the other groups.