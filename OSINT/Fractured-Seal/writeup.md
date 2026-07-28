# Fractured Seal — OSINT Challenge

**CTF:** Cyber Apocalypse 2026  
**Category:** OSINT  
**Challenge:** Fractured Seal  
**Flag:** `HTB{r3c0v3r1ng_RSA_k3ys___l1k3___Me0w___me0o00o0o0w___Me0w}`

---

## Scenario

> One of the Registry’s oldest key-scrolls survived the fall of Crownspire, though time and fire spared only fragments of its writing, and most in the vault dismissed it as useless. Caldrin didn’t. She always said a seal doesn’t have to be whole to still remember the door it once opened.

---

## Files

| File | Description |
|------|-------------|
| `encrypt.py` | Standard RSA-2048 encryption script |
| `fractured_seal.pem` | RSA private key with ~65% base64 content replaced by `*` (asterisks) |
| `flag.enc` | RSA-encrypted flag (256 bytes, 2048-bit ciphertext) |

## encrypt.py Analysis

```python
p = getPrime(1024)
q = getPrime(1024)
n = p * q
e = 0x10001
d = pow(e, -1, (p-1)*(q-1))

m = bytes_to_long(open(’flag.txt’, ‘rb’).read())
open(’seal.pem’, ‘wb’).write(RSA.construct((n, e, d)).export_key())
open(’flag.enc’, ‘wb’).write(long_to_bytes(pow(m, e, n)))
```

Standard RSA-2048 with `e=65537`. The full private key was exported as PEM, then parts of the PEM base64 were replaced with `*` characters (the “fracture”).

---

## PEM Structure Analysis

The PEM file has 27 lines (0–26), total 1595 bytes. Base64 body is 1509 characters, of which **981 (65%) are asterisks** and only 528 are valid base64 data.

**Valid data fragments recovered:**

| Lines | Content | Length |
|-------|---------|--------|
| 1–6 (prefix) | ASN.1 DER header + modulus **n** (complete, missing ~1 byte) | 357 base64 chars |
| 14–16 | Partial private component — **high 584 bits of p** | 106 base64 chars |

### Component: **n (modulus)** — Recovered (1 byte ambiguous)

```
n = 0x8cae7d6a15e55fb6...2095  (partial, 2040 bits)
missing_last_byte = base64_char_357 value 29 << 2 | guess_2_bits
guess = 0x75 (correct value determined by Sage solving)
```

The 357th base64 char (`d`) encodes only 6 of the 8 bits of n’s last byte. The remaining 2 bits (4 possibilities) were tried; `0x75` was the correct one.

### Component: **p (prime factor)** — Partially Recovered

High 584 bits of 1024-bit prime p:
```
p_high = 0xc0b1b1709eff60282af463c34b5e9c1cf35912662b6d25fe6adcd40413b68c...dffae  (584 bits)
k = 440 unknown bits
p0 = p_high << 440
```

### Components still missing (replaced by asterisks)

- **d** — private exponent (except 2-byte tail `da6b`)
- **q** — second prime factor
- **dp, dq, qinv** — CRT parameters

### Emoji Artifact

Lines 18–21 contain unicode/emoji characters embedded among asterisks:

```
  ／l、
（ﾟ､｡７
  l、 ~ヽ
  じしf_, )ノ
```

This is ASCII art decoration (no cryptographic significance).

---

## Solution Approach: Coppersmith’s Method

The challenge requires recovering the missing 440 low bits of `p` using Coppersmith’s partial-key-recovery attack. The polynomial:

```
f(x) = p0 + x  (mod n)
```

We need `x` such that `f(x0) ≡ 0 (mod p)` with `|x0| < 2^440`.

The bound `X = 2^440` satisfies `X < n^0.25`, so Coppersmith’s theorem applies.

### SageMath Solution Script

```python
n = 0x8cae7d6a15e55fb6bea05fa3b7989cba4a4678d5d78103b80f78cf78b68602ea4\
    99ba6baf4f9d609fd16dd00e19823f12d8592af50b80efc4fdb7525bcff0b71fbbec3f\
    a084b48c31e47e7bfe9b5424bbf8a2f707a1cbf89136e51e687f3c2b8000d1716f268\
    6a4e48d08822e64b0ffc2b3218c4496d47bcf7c78b66360e600a00dbe7e22e38df603\
    047962c97aef6b9e56f4eda27b648d67643ce7d9006f42108ff0cadcbb801eab6e13b\
    1d59b6283c9d5348e388c173e6e870714e678d79c775945fac24da51604b2f1984f0f\
    09b22ecbc849aa12fe883d4e7304d2f18a3cc684eda1cec71457d760da0a8e7aa0a7e\
    2ce98bc1019078ed98a351dbf12095
n = (n << 8) | 0x75  # Last missing byte of the modulus

p_high = 0xc0b1b1709eff60282af463c34b5e9c1cf35912662b6d25fe6adcd40413b68c\
         b620d106d77e2b532077d4fb9e43a88c4540e72ef826a1acf41044ea1900020f8\
         342dd68677edd1dffae
k = 440
X = 2^k
p0 = p_high << k

ct = 0x59a7e6ddce6eb48c5e82224503e94df9cccb26012594d1f87c7a42262a566f58\
     3835b1a1e652360c0bb222e9013ebb67a72c5623664507ff009b636e0cbfebc34cd4\
     3981e0cdb79dfb45c5c4530d23301088d8ab179c70050326d48983d253794e53b732\
     e82fa4688e171bd7fe681b3aace0fa317dfacfdc8ff0396df90574116552a3f52a80\
     2e9e7e6e72b89a85257a842233ca90864e089c3f07c5355f7d2e27b309c5da753bc\
     c6fa8d041689f0135c6a53f10677a37ea9e9f0d3c2fcc56c525deee8e5efad881a9\
     050dcbcdbecef09c4dee5a466650473594c691077725b12e1c384d4573e3afc01f15\
     da65776526b418db5b4e4388bc17abf4ba5f224f3c
e = 65537

F = Zmod(n)
P.<x> = PolynomialRing(F, implementation=’NTL’)
f = p0 + x

roots = f.small_roots(X=X, beta=0.5, epsilon=0.03, mm=9, tt=9)
x0 = roots[0]
p = p0 + int(x0)
q = n // p

d = inverse_mod(e, (p-1)*(q-1))
pt = pow(ct, d, n)
flag = int(pt).to_bytes((int(pt).bit_length()+7)//8, ‘big’)
print(flag.decode())
```

Parameters: `beta=0.5`, `epsilon=0.03`, `mm=9`, `tt=9`, lattice dimension = 18.

---

## How We Solved It — Reasoning

**Step 1 — Reconnaissance:** The PEM file had clear structural damage — asterisks replaced most of the base64 content, but key fragments remained. By parsing the valid base64 prefix, we extracted most of the modulus `n` (last byte ambiguous with 4 possibilities).

**Step 2 — Fragment Recovery:** Lines 14–16 contained additional valid base64 yielding the high 584 bits of prime `p` (encoded as an ASN.1 INTEGER in the DER structure). The `da 6b` prefix confirmed this was the tail of `d`, followed by `p`. The 357th base64 char encoded only 6/8 bits of n’s last byte, leaving 2 bits (4 values) to try.

**Step 3 — Attack Selection:** With 584 of 1024 bits of `p` known (57%), Coppersmith’s method can recover the remaining 440 bits. The bound `X=2^440 < n^0.25=2^512` satisfies the Howgrave-Graham condition.

**Step 4 — Implementation:** SageMath’s `f.small_roots(X=2^440, beta=0.5, epsilon=0.03, mm=9, tt=9)` built an 18×18 lattice, ran LLL, and found the root `x0 = 510368856612003907401221109096599223139656586863610951944254547606743207569868452222666617647861792610706314208660821023160189978527`.

**Step 5 — Decryption:** With `p = p0 + x0` recovered, we computed `q = n // p`, `phi = (p-1)*(q-1)`, `d = e^(-1) mod phi`, and decrypted the RSA ciphertext.

**Why this is OSINT:** The challenge simulates an intelligence-gathering scenario where fragments of corrupted data (a “fractured seal”) must be assembled to recover a cryptographic key. The OSINT skill is in recognizing which fragments are useful and applying the right mathematical tool (Coppersmith) without complete information.

---

## Key Takeaways

- **RSA partial key recovery** via Coppersmith is potent when ≥50% of a prime factor is known
- **The modulus was missing its last byte** — the 357th base64 char only encodes 6 of 8 bits; 4 possible values for the last byte
- **SageMath’s `small_roots`** uses carefully tuned BKZ preprocessing that fpylll’s LLL lacks
- The Coppersmith-Howgrave-Graham construction needs `mm ≥ ceil(β²/ε)` for theoretical guarantee; `mm=9`, `tt=9` (dim=18) solved 440 unknown bits

---

*Writeup by SaltyHash443 — Cyber Apocalypse 2026*