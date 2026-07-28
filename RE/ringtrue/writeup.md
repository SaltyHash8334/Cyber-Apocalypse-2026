# Ringtrue — Reverse Engineering Challenge

**Category:** Reverse Engineering / Binary  \
**Challenge:** Cyber Apocalypse 2026 — House Eastreach Relic Division  \
**Flag:** `HTB{h3y_s1gn3t_1_4m_y0ur_k1ng}`

---

## How We Solved It — Reasoning

### Overview

The challenge presents a "Resonance Console" — a device built around a broken shard of Dragon Astrael's Signet. The binary (`ringtrue`, ELF x86-64) simulates a boot sequence and then prompts for **8 tone samples**. If the right 8 tones are played, the "vault-seal" opens and the sealed vow (the flag) is revealed.

The lore tells us Eastreach taught the device to listen for **one voice** and left everything "written plainly inside." That means the neural network weights, the reference signature, and the encrypted flag are all embedded in the binary.

### Step 1 — Reconnaissance

Running `strings` on the binary reveals key metadata:

- "**MLP 8-8-8-8, int8, leaky**" — a 3-layer perceptron with 8 neurons each
- "**weights symmetric int8 (zp=0), per-tensor scale**" — quantization params
- "**vault-seal = xor-stream, key = attunement-derived**" — flag is XOR-encrypted with a key derived from the tones
- "**astrael-echo: reference signature pinned**" — target output is embedded
- Symbol names: `L0_W`, `L1_W`, `L2_W` (weights), `L0_B`, `L1_B`, `L2_B` (biases), `ECHO_S` (reference), `VOW_CIPHER` (encrypted flag), `VOW_LEN` (30)

### Step 2 — Reverse Engineering the MLP Forward Pass

Using Capstone to disassemble the `dense()` function:

```
dense(rdi=weights, rsi=biases, rdx=input, rcx=output):
  for i in 0..7:                          # output neuron loop
    acc = sign_extend(bias[i])            # int32 → int64
    for j in 0..7:                        # input element loop
      w = sign_extend(byte[weights + i*8 + j])  # one int8 byte → int64
      acc += w * input[j]                        # int64 arithmetic
    output[i] = acc                        # int64 result
```

The key detail: **weights are single int8 bytes** loaded one at a time, not int64 values. And each layer applies a **leaky ReLU with negative slope of 2** (if output[i] < 0: output[i] *= 2).

Three `dense()` calls implement the full 8→8→8→8 MLP, and then a comparison matches the final output against `ECHO_S` (8 int64 values).

### Step 3 — Extracting Parameters

Data extracted from the `.data` section (VMA offsets):

| Symbol | Offset | Content |
|--------|--------|---------|
| `L0_W` | 0x5180 | 8×8 int8 weight matrix (64 bytes) |
| `L1_W` | 0x5140 | 8×8 int8 weight matrix (64 bytes) |
| `L2_W` | 0x5100 | 8×8 int8 weight matrix (64 bytes) |
| `L0_B` | 0x50e0 | 8 int32 biases |
| `L1_B` | 0x50c0 | 8 int32 biases |
| `L2_B` | 0x50a0 | 8 int32 biases |
| `ECHO_S` | 0x5060 | 8 int64 reference output values |
| `VOW_CIPHER` | 0x5030 | 32 bytes of encrypted flag |
| `VOW_LEN` | 0x5020 | 30 (flag length) |

### Step 4 — Inverting the Network (Key Breakthrough)

Since the entire computation is **integer arithmetic** with piecewise-linear activations (leaky ReLU), we can invert it exactly by solving systems of linear equations:

**Phase 1 — Invert Layer 2:**
```
ECHO_S[i] = L2_B[i] + Σ_j(L2_W[i][j] × L1_act[j])
```
This is an 8×8 linear system: solve `L2_W × L1_act = ECHO_S - L2_B`. The matrix has full rank (8), giving an exact integer solution.

**Phase 2 — Invert Leaky ReLU:**
```
If L1_act[i] ≥ 0: L1_out[i] = L1_act[i]
If L1_act[i] < 0:  L1_out[i] = L1_act[i] / 2
```

**Phase 3 — Invert Layer 1:**
Solve `L1_W × L0_act = L1_out - L1_B` → exact integer solution again.

**Phase 4 — Invert Layer 0:**
Solve `L0_W × input = L0_out - L0_B` → exact integer solution.

The solved tones:
```
[83, 97, 108, 116, 67, 114, 119, 110]
```

Which decodes to ASCII: **`SaltCrwn`** — the name the shard was taught to obey.

### Step 5 — Flag Decryption

When the tones match, the binary:
1. Takes the **low byte** of each input tone → `"SaltCrwn"`
2. Prepends 4 zero bytes for a 12-byte SHA-256 input: `"SaltCrwn\0\0\0\0"`
3. Computes SHA-256 of those 12 bytes
4. XORs `VOW_CIPHER` with the hash bytes to produce the plaintext

```python
import hashlib
sha_input = b"SaltCrwn" + b"\x00\x00\x00\x00"
flag = bytes(vow_cipher[i] ^ hashlib.sha256(sha_input).digest()[i] for i in range(30))
```

### Implementation

The full solver script:

```python
import struct, hashlib, numpy as np

# Load binary data
with open('ringtrue', 'rb') as f: data = f.read()

def read_i8(o): return struct.unpack('<b', data[o:o+1])[0]
def read_i32(o): return struct.unpack('<i', data[o:o+4])[0]
def read_i64(o): return struct.unpack('<q', data[o:o+8])[0]

# Extract weights, biases, echo
L0_W = np.array([[read_i8(0x5180+i*8+j) for j in range(8)] for i in range(8)], dtype=np.int64)
L0_B = np.array([read_i32(0x50e0+i*4) for i in range(8)], dtype=np.int64)
L1_W = np.array([[read_i8(0x5140+i*8+j) for j in range(8)] for i in range(8)], dtype=np.int64)
L1_B = np.array([read_i32(0x50c0+i*4) for i in range(8)], dtype=np.int64)
L2_W = np.array([[read_i8(0x5100+i*8+j) for j in range(8)] for i in range(8)], dtype=np.int64)
L2_B = np.array([read_i32(0x50a0+i*4) for i in range(8)], dtype=np.int64)
ECHO = np.array([read_i64(0x5060+i*8) for i in range(8)], dtype=np.int64)
VOW = data[0x5030:0x5030+32]

# Invert layer 2: L1_act = inv(L2_W) * (ECHO - L2_B)
L1_act = np.round(np.linalg.solve(L2_W.astype(float), (ECHO - L2_B).astype(float))).astype(np.int64)

# Invert leaky ReLU
L1_out = np.array([v if v >= 0 else v//2 for v in L1_act])

# Invert layer 1: L0_act = inv(L1_W) * (L1_out - L1_B)
L0_act = np.round(np.linalg.solve(L1_W.astype(float), (L1_out - L1_B).astype(float))).astype(np.int64)

# Invert leaky ReLU
L0_out = np.array([v if v >= 0 else v//2 for v in L0_act])

# Invert layer 0: input = inv(L0_W) * (L0_out - L0_B)
input_tone = np.round(np.linalg.solve(L0_W.astype(float), (L0_out - L0_B).astype(float))).astype(np.int64)

print(f"Tones: {list(input_tone)}")
print(f"ASCII: {''.join(chr(v) for v in input_tone)}")

# Decrypt the flag
sha_input = bytes(input_tone) + b'\x00' * 4
flag = bytes(VOW[i] ^ hashlib.sha256(sha_input).digest()[i] for i in range(30))
print(f"Flag: {flag.decode()}")
```

### Why This Works

- **Integer arithmetic is exact**: With no floating-point rounding, every operation is deterministic and reversible.
- **Full-rank weight matrices**: All three 8×8 weight matrices are invertible, giving unique exact solutions at each layer.
- **Leaky ReLU is invertible**: Since the negative slope is 2 (not 0), the activation has a clean inverse: positive stays, negative halves.
- **SHA-256 key derivation**: The flag is XOR-encrypted with SHA-256("SaltCrwn\0\0\0\0"), which is deterministic once the tones are known.

---

### Flag

```
HTB{h3y_s1gn3t_1_4m_y0ur_k1ng}
```

### Caveats

- The binary is x86-64 and won't run on ARM64 natively (qemu-user may be needed for testing).
- The 4 extra zero bytes in the SHA-256 input (12 bytes total, not 8) are critical — SHA-256 of just `"SaltCrwn"` gives a different hash.
- The weight matrices are stored as single int8 bytes (loaded with `movsx`), not as int64 values — easy to misinterpret when reading the `.data` hex dump.
- The leaky ReLU slope of 2 (not the usual 0.01) was confirmed by the `lea rcx, [rax+rax]` / `cmovs rax, rcx` pattern — doubling the register value when negative.
