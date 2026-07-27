# The Ashen Field — Crypto Write-Up

**Challenge:** The Ashen Field
**Category:** Crypto
**Files:** `source.sage`, `output.txt`
**Flag Format:** HTB{...}

---

## How We Solved It — Reasoning

### Overview

The challenge provides a SageMath script implementing a multivariate cryptosystem over GF(2) with 137 variables (n=137). The public key is a vector of 137 polynomials, and we're given the encryption of a random KEY under this public key, along with an AES-ECB encrypted flag using that KEY. The goal is to recover the KEY and decrypt the flag.

The cryptosystem is a variant of **Hidden Field Equations (HFE)**, as noted in the flag text itself (`th4nks_f4l4y_f0r_y0ur_4tt4ck_0n_HFE` — a reference to Faugère's F4/F5 algorithm and the Joux-Faugère attack on HFE).

### Step 1: Understanding the Cryptosystem

The `keygen()` function builds a public key through a composition of:

1. **Inner affine transformation:** `Rv = S_A * X + S_B` (137 variables mapped affinely)
2. **Polynomial conversion:** `F = Rv[::-1]` cast to a polynomial in `t` over `R = GF(2)[x₁,...,x₁₃₇]`
3. **HFE-style central map:** `F = F^4 + F^2 + 1` in the quotient ring `Q = R[t]/(g(t))` where `g` is an irreducible degree-137 polynomial over GF(2)
4. **Outer affine transformation:** `PK = T_A * raw_PK + T_B`

The critical insight: in characteristic 2, the Frobenius endomorphism means `(a+b)² = a² + b²`, so squaring an affine function produces only squared-variable terms and no cross-terms. Therefore **every polynomial in the public key contains only `xᵢ⁴`, `xᵢ²`, and constant `1` terms — NO product terms like `xᵢ·xⱼ`**.

When evaluated at binary points `{0,1}`, `xᵢ⁴ = xᵢ` and `xᵢ² = xᵢ` in GF(2). So the effective map `PK(msg)` is **purely affine**: `c = M·m + b` over GF(2), where:
- `M[i,j] = 1` iff variable `xⱼ` appears an odd number of times (as xⱼ⁴ XOR xⱼ²) in polynomial `i`
- `b[i] = 1` iff polynomial `i` has the constant `1` term

### Step 2: Building the Linear System

From `output.txt`, we parsed the 137 polynomials in the public key vector and extracted:

- **Matrix M** (137×137 over GF(2)): each variable appears in many polynomials, with cancellation when a variable appears as both `xᵢ⁴` and `xᵢ²` in the same polynomial
- **Vector b** (length 137 over GF(2)): 66 out of 137 polynomials have constant term `1`

The ciphertext `ct` = `PK(KEY_bits)` = `M·KEY_bits + b`, giving us:

```
M·KEY_bits = ct + b  (over GF(2))
```

### Step 3: Solving for KEY Bits

Gaussian elimination over GF(2) gave:

- **Rank:** 135 (not full rank — inherent to the HFE construction)
- **Free variables:** columns 133 and 136 (0-indexed)
- **Unique solution with ct match:** one specific free-variable assignment produced the correct ciphertext

The solved bits are in `KEY.bits()` order, which in SageMath returns **least significant bit first**.

| Free var settings | KEY (LSB→MSB) | Matches ct? |
|---|---|---|
| col133=0, col136=0 | 274733587… | ❌ |
| col133=1, col136=0 | 384280124… | ❌ |
| **col133=0, col136=1** | **110621486…** | **✅** |
| col133=1, col136=1 | 125474680… | ❌ |

**Critical fix:** The free variable values propagate through pivot rows' equations. Setting free variables without updating dependent pivot variables produced incorrect keys. Only when we properly computed `pivot = RHS ⊕ Σ(free_var_in_row)` for each pivot row did the correct solution emerge.

The correct KEY (`fv=2`, col133=0, col136=1):
```
KEY = 110621486878801192554077110255801668498185
```
This has bit_length = 137, satisfying the source code's `KEY.nbits() == N` constraint (MSB = 1).

### Step 4: AES Decryption

With the correct KEY integer, we derive the AES-256 key:

```python
AES_KEY = hashlib.sha256(str(KEY).encode()).digest()
cipher = AES.new(AES_KEY, AES.MODE_ECB)
flag = unpad(cipher.decrypt(enc_flag), 16)
```

### The Flag

```
HTB{e1th3r_gr0bn3r_0r_v4r13ty___1t_st1ll_w0rks!th4nks_f4l4y_f0r_y0ur_4tt4ck_0n_HFE}
```

The flag name-checks both **Gröbner basis** and **variety** approaches to the HFE cryptosystem, and thanks **F4L4Y** (Faugère's F4/F5 Gröbner basis algorithm) for the attack on HFE.

---

## Key Lessons

1. **SageMath `Integer.bits()` returns LSB first**, not MSB first — constructing the integer from solved bits requires `sum(bits[i] · 2ⁱ)`, not MSB-first binary string parsing
2. **When solving rank-deficient linear systems**, free variable propagation through pivot equations must be handled explicitly
3. **HFE over GF(2) with the `F⁴+F²+1` central map** reduces to affine when restricted to single-variable power terms due to Frobenius in characteristic 2
4. The affine structure means standard linear algebra solves the system despite the multivariate polynomial facade

## Files

- `solve.py` — full solution script (linear system → KEY → AES decrypt)
- `output.txt` — challenge data (PK vector, ciphertext, encrypted flag)
- `source.sage` — challenge source code

## References

- Faugère, J.-C., & Joux, A. (2003). *Algebraic cryptanalysis of Hidden Field Equation (HFE) cryptosystems using Gröbner bases*
- HFE cryptosystem (Patarin, 1996)