# Overstrike — Crypto Challenge

**CTF:** Cyber Apocalypse 2026 (CApoc)  
**Category:** Crypto  
**Challenge:** Overstrike  
**Flag:** `HTB{0v3rstr1k3_r3cut_th3_w0rld_s34l_by_f0rg1ng_th3_mark}`

---

## Scenario

The Signet is shattered, and its silence has a shape now: House Vaultrune's quiet trade in re-cut marks, genuine authority lifted and struck anew, each forgery slipped into the Registry so the lie reads as record.

You are Elric. You go down under Crownspire, toward the sealed archive Chancellor Veylen Marr keeps, and find the old vow-stones of the Ash-Vault will not bridge the fire-rift for any mark you carry. They answer only to the true seal.

To reach the truth the forgers have filed as fact, you must do the very thing that condemns them: cross on a seal of your own making, one the world itself cannot tell from genuine.

---

## Analysis

### Artifact

`Overstrike.apk` is a Godot/C# Android application. The APK is a ZIP archive containing the managed assembly:

```text
assets/.godot/mono/publish/x86_64/Overstrike.dll
assets/.godot/mono/publish/arm64/Overstrike.dll
```

The packaged `assets/scripts/*.cs` files are one-byte placeholders, but the assembly metadata preserves the class and method names. Extract the x86-64 assembly:

```bash
unzip -p Overstrike.apk \
  assets/.godot/mono/publish/x86_64/Overstrike.dll > Overstrike.dll
```

Relevant symbols include:

```text
GameState
TrueSeal
Mix
UnsealRegistry
WorldIsAligned
CarriedMark
WorldSeal
BridgeBuilder
TileHash
Rebuild
```

### The verifier

`BridgeBuilder.Rebuild` stores the current seal and marks the bridge aligned only when it equals `GameState.TrueSeal`. The hard-coded target is:

```text
0xd9a1bb0cabb52586
```

The game exposes five ordinary pickup marks, but none is the required mark. The useful path is to recover a mark that produces the target seal.

`GameState.Mix` is a SplitMix64-style 64-bit permutation:

```python
def mix(x):
    x = (x + 0x9E3779B97F4A7C15) & ((1 << 64) - 1)
    x ^= x >> 30
    x = (x * 0xBF58476D1CE4E5B9) & ((1 << 64) - 1)
    x ^= x >> 27
    x = (x * 0x94D049BB133111EB) & ((1 << 64) - 1)
    x ^= x >> 31
    return x & ((1 << 64) - 1)
```

This is not encryption. The XOR-right-shifts are reversible, and both multiplication constants are odd, so they have modular inverses modulo `2^64`.

### Archive decryption

The sealed registry is an embedded static byte array. `UnsealRegistry` derives the decryption key from the mark:

```text
key = SHA256(BitConverter.GetBytes(mark))
```

It then XORs the ciphertext with SHA-256 blocks generated from the key and an incrementing little-endian 32-bit counter:

```text
SHA256(key || BitConverter.GetBytes(counter))
```

The first 56 bytes of the static array contain the encrypted flag; the remaining bytes are alignment/padding in the field storage.

---

## Solution / Exploitation

### Step 1 — Invert the seal mixer

For `y = x ^ (x >> s)`, inversion can be performed by repeatedly applying the same expression to the output:

```python
def invert_xor_right(value, shift):
    result = value
    for _ in range(8):
        result = value ^ (result >> shift)
    return result & ((1 << 64) - 1)
```

Undo `Mix` in reverse order: invert the final XOR, multiply by the inverse of the second multiplier, invert the middle XOR, multiply by the inverse of the first multiplier, invert the first XOR, then subtract the additive constant.

The recovered mark is:

```text
mark = 0xd7caad24dd98b676
signed Int64 = -2897313036411292042
```

The signed value is important because the C# code hashes `BitConverter.GetBytes(long)`, not an arbitrary textual representation.

### Step 2 — Verify the mark

Forward evaluation confirms the inversion:

```text
true_seal = 0xd9a1bb0cabb52586
mix(mark)  = 0xd9a1bb0cabb52586
```

### Step 3 — Decrypt and recover the flag

The following solver reimplements the relevant managed-code logic locally:

```python
#!/usr/bin/env python3
import hashlib
import struct

MASK = (1 << 64) - 1
TRUE_SEAL = 0xd9a1bb0cabb52586
CIPHERTEXT = bytes.fromhex(
    "0d563344126e440f363dec5e87cad5b60401b6b596e4b87e79e0ecdc075299fb"
    "b36800572022033ca6607c32fd1f7cb3dc9d7873132f600b"
)


def invert_xor_right(value, shift):
    result = value
    for _ in range(8):
        result = value ^ (result >> shift)
    return result & MASK


def invert_mix(seal):
    x = invert_xor_right(seal, 31)
    x = x * pow(0x94D049BB133111EB, -1, 1 << 64) & MASK
    x = invert_xor_right(x, 27)
    x = x * pow(0xBF58476D1CE4E5B9, -1, 1 << 64) & MASK
    x = invert_xor_right(x, 30)
    return (x - 0x9E3779B97F4A7C15) & MASK


def decrypt(mark):
    # Match C# BitConverter.GetBytes(long), including signed Int64 handling.
    signed_mark = struct.unpack("<q", struct.pack("<Q", mark))[0]
    key = hashlib.sha256(struct.pack("<q", signed_mark)).digest()
    plaintext = bytearray()

    for counter in range((len(CIPHERTEXT) + 31) // 32):
        stream = hashlib.sha256(key + struct.pack("<i", counter)).digest()
        block = CIPHERTEXT[counter * 32:(counter + 1) * 32]
        plaintext.extend(a ^ b for a, b in zip(block, stream))

    return bytes(plaintext)


mark = invert_mix(TRUE_SEAL)
assert mark == 0xd7caad24dd98b676
plaintext = decrypt(mark).decode()
assert plaintext.startswith("HTB{") and plaintext.endswith("}")
print(f"true_seal = 0x{TRUE_SEAL:016x}")
print(f"recovered mark = 0x{mark:016x}")
print(f"plaintext = {plaintext}")
```

### Verified output

```text
true_seal = 0xd9a1bb0cabb52586
recovered mark = 0xd7caad24dd98b676
plaintext = HTB{0v3rstr1k3_r3cut_th3_w0rld_s34l_by_f0rg1ng_th3_mark}
```

---

## How We Solved It — Reasoning

The story says that ordinary marks cannot cross the rift and that the forged seal must be indistinguishable from the genuine one. That points away from searching the level for another pickup and toward inspecting the verifier itself.

The first hypothesis was that the correct mark might be hidden among the visible pickups. Static analysis ruled this out: `BuildMarks` creates five values, while `BridgeBuilder.Rebuild` compares the resulting seal against an immutable `TrueSeal`. The normal gameplay path cannot supply the required value.

The next hypothesis was that `Mix` might be a one-way hash. Its CIL disproved that. It contains only addition, XOR-right-shift operations, and multiplication by odd constants. Each operation is invertible over 64-bit arithmetic, so the apparent cryptographic seal is actually a public permutation. Inverting it directly recovers the unique mark that maps to `TrueSeal`.

The recovered value was then validated independently rather than accepted from the arithmetic alone:

1. Running the original mixer forward reproduces `0xd9a1bb0cabb52586` exactly.
2. Using the same signed little-endian representation and SHA-256 counter construction from `UnsealRegistry` decrypts the embedded bytes into a valid `HTB{...}` flag.

Those checks correlate the verifier constant, the recovered mark, and the archive plaintext. This rules out a coincidental inversion or an endianness/sign error.

---

## Key Takeaways

- SplitMix64 provides fast diffusion, not secrecy; its public permutation is directly invertible.
- Odd multipliers always have modular inverses modulo `2^64`.
- Reproducing C# cryptographic code requires matching signed integer and little-endian byte semantics.
- Embedded encrypted data is a useful verification oracle: the correct mark decrypts to structured plaintext, while incorrect marks do not.
- Static analysis was sufficient; no Android device or network service was required.

---

## Flag

`HTB{0v3rstr1k3_r3cut_th3_w0rld_s34l_by_f0rg1ng_th3_mark}`

---

## Exact constants

```text
TRUE_SEAL = d9a1bb0cabb52586
MIX_ADD   = 9e3779b97f4a7c15
MIX_MUL1  = bf58476d1ce4e5b9
MIX_MUL2  = 94d049bb133111eb
MARK      = d7caad24dd98b676
```

---

## References

- `Overstrike.apk` — supplied challenge artifact
- SplitMix64 reference implementation: https://prng.di.unimi.it/splitmix64.c
- C# `BitConverter.GetBytes`: https://learn.microsoft.com/en-us/dotnet/api/system.bitconverter.getbytes
- SHA-256: https://www.rfc-editor.org/rfc/rfc6234
