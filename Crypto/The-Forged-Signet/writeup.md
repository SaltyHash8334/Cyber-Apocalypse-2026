# The Forged Signet — Crypto Challenge

**CTF:** Cyber Apocalypse 2026 (CApoc)
**Category:** Crypto
**Challenge:** The Forged Signet
**Flag:** `HTB{th3_f1rst_m4rk_1s_4_h1dd3n_x0r_p3r10d_e33ef50dc07825b3aa2838a251a07cbf}`

---

## Scenario

A long time ago the Brine Signet was the one thing nobody could fake; it was how the Crown proved a seal was real. Then it broke, and everything went bad. Every seal is still checked against a secret called the **First Mark**, a hidden string `s` that lives inside the verifier. The verifier has a flaw it was never meant to have: if two inputs differ by exactly the Mark, it cannot tell them apart, so `f(x)` and `f(x XOR s)` always look the same to it.

**Target:** `154.57.164.81:31015`

---

## How We Solved It — Reasoning

### The Core Insight

This challenge implements **Simon's algorithm** — a quantum algorithm for finding the hidden period `s` of a function `f` satisfying `f(x) = f(x ⊕ s)`. The server exposes a quantum oracle simulator behind an HTTP API. By querying the oracle with `L=H` (Hadamard gates), each measurement `y` satisfies `y · s = 0 (mod 2)`. Collecting enough such measurements and solving the resulting linear system over GF(2) uniquely determines `s`.

### Key Observations

1. **The circuit is:** `H⊗n → Uf → measure+discard(output) → L⊗n → measure(input)` — exactly Simon's algorithm, but with a configurable final gate layer L.

2. **Why L=H works:** After the output register is measured and discarded, the input register collapses to `(|x₀⟩ + |x₀⊕s⟩)/√2`. Applying H⊗n again gives:
   ```
   H⊗n(|x₀⟩ + |x₀⊕s⟩) = Σ_y (-1)^(x₀·y) [1 + (-1)^(s·y)] |y⟩
   ```
   The amplitude vanishes when `s·y = 1 (mod 2)`, so every measured `y` satisfies `y · s = 0`.

3. **Why L=I fails:** With identity gates, each shot randomly yields either `x₀` or `x₀⊕s` — but each independent circuit run picks a new random `x₀`, so across shots we just get uniform random bitstrings with no exploitable structure.

4. **n=64 bits:** With `n=64`, we need 63 linearly independent equations to uniquely determine `s` (nullity = 1, excluding the zero vector).

---

## Solution

### 1. Reconnaissance

The target runs Gunicorn (HTTP server) on port 31015, serving a web app for interacting with the quantum oracle.

```bash
$ nmap -sV -p 31015 154.57.164.81
PORT      STATE SERVICE VERSION
31015/tcp open  http    Gunicorn
```

### 2. Oracle Metadata

```bash
$ curl -s http://154.57.164.81:31015/api/oracle | python3 -m json.tool
{
    "circuit": "each run: H^n . U_f . measure&discard(output) . L . measure(input) ; you choose the single-qubit layer L",
    "endpoints": {
        "/api/forge": "POST {s} -> submit the n-bit First Mark to forge the seal",
        "/api/run": "POST {layer, shots} -> measured input bitstrings"
    },
    "n": 64,
    "promise": "the verifier f obeys  f(x) = f(x XOR s)  for a secret non-zero First Mark s of n bits",
    "serial": "75c4e456cea69693"
}
```

The `circuit` field confirms the Simon's algorithm structure. The `promise` field confirms `f(x) = f(x ⊕ s)`.

### 3. Query Oracle with L=H

Query the oracle with `layer="H"` and `shots=256` to collect measurement results. Each measurement `y` satisfies `y · s = 0 (mod 2)`. With 4 batches of 256 shots = 1024 measurements, we get a rank-63 matrix (nullity = 1) — enough to uniquely determine `s`.

```python
import requests
import numpy as np

url = "http://154.57.164.81:31015/api/run"
all_samples = []

for i in range(4):
    resp = requests.post(url, json={"layer": "H", "shots": 256})
    data = resp.json()
    all_samples.extend(data['samples'])

# Convert to binary vectors
M = np.array([[int(c) for c in s] for s in all_samples], dtype=np.uint8)
```

### 4. Solve Linear System over GF(2)

Perform Gaussian elimination over GF(2) to find the nullspace of the measurement matrix. With rank 63 and 64 columns, the nullspace is 1-dimensional — exactly the hidden string `s`.

```python
def gf2_nullspace(M):
    """Find nullspace basis vectors of M over GF(2)."""
    n_samples, n = M.shape
    A = M.copy()
    pivot_rows = []
    
    # Forward elimination
    col = 0
    row = 0
    while col < n and row < n_samples:
        pivot = None
        for r in range(row, n_samples):
            if A[r, col] == 1:
                pivot = r
                break
        if pivot is None:
            col += 1
            continue
        if pivot != row:
            A[[row, pivot]] = A[[pivot, row]]
        for r in range(n_samples):
            if r != row and A[r, col] == 1:
                A[r] ^= A[row]
        pivot_rows.append((row, col))
        row += 1
        col += 1
    
    rank = len(pivot_rows)
    pivot_cols = {c for _, c in pivot_rows}
    free_cols = [c for c in range(n) if c not in pivot_cols]
    
    nullspace = []
    for free_c in free_cols:
        v = np.zeros(n, dtype=np.uint8)
        v[free_c] = 1
        for (pivot_row_idx, pivot_col) in reversed(pivot_rows):
            val = 0
            for c in range(n):
                if c != pivot_col and A[pivot_row_idx, c] == 1:
                    val ^= v[c]
            v[pivot_col] = val
        nullspace.append(v)
    
    return nullspace

nullspace = gf2_nullspace(M)
# nullspace[0] = 0110110000100010101111010110010001100101000011101101000000000101
```

**Output:**
```
Rank: 63
Nullity: 1
Nullspace basis: 0110110000100010101111010110010001100101000011101101000000000101
```

### 5. Submit the First Mark

Submit the recovered `s` to the forge endpoint:

```bash
$ curl -s -XPOST http://154.57.164.81:31015/api/forge \
  -H 'content-type: application/json' \
  -d '{"s":"0110110000100010101111010110010001100101000011101101000000000101"}'
{
    "flag": "HTB{th3_f1rst_m4rk_1s_4_h1dd3n_x0r_p3r10d_e33ef50dc07825b3aa2838a251a07cbf}",
    "forged": true
}
```

---

## API Reference

| Endpoint | Method | Payload | Response |
|----------|--------|---------|----------|
| `/api/oracle` | GET | — | `{serial, n, promise, circuit, endpoints}` |
| `/api/run` | POST | `{layer: "H"\|"I"\|..., shots: 1-256}` | `{layer, shots, samples: ["0101...", ...]}` |
| `/api/forge` | POST | `{s: "64-bit binary string"}` | `{forged: bool, flag?: string}` |

---

## Mathematical Background

Simon's algorithm solves the hidden subgroup problem for `Z₂ⁿ`. Given an oracle computing `f(x)` where `f(x) = f(x ⊕ s)` for some unknown `s`, the algorithm:

1. Prepares superposition: `H⊗n|0⟩ = 1/√(2ⁿ) Σₓ |x⟩`
2. Applies oracle: `1/√(2ⁿ) Σₓ |x⟩|f(x)⟩`
3. Measures output register → collapses input to `(|x₀⟩ + |x₀⊕s⟩)/√2`
4. Applies `H⊗n` to input → Σ_y (-1)^(x₀·y)[1+(-1)^(s·y)] |y⟩
5. Measures input → any y with `s·y = 0`

Each measurement gives one linear equation `s · y = 0` over GF(2). After O(n) measurements, Gaussian elimination recovers `s`.

---

## Solution Script

```python
#!/usr/bin/env python3
"""Simon's algorithm solver for The Forged Signet (CApoc 2026)."""
import requests
import numpy as np

TARGET = "http://154.57.164.81:31015"

def collect_samples(n_batches=4, shots=256):
    """Collect measurement samples with L=H."""
    samples = []
    for i in range(n_batches):
        r = requests.post(f"{TARGET}/api/run",
                         json={"layer": "H", "shots": shots})
        data = r.json()
        samples.extend(data["samples"])
        print(f"Batch {i+1}: {len(data['samples'])} samples")
    return np.array([[int(c) for c in s] for s in samples], dtype=np.uint8)

def gf2_nullspace(M):
    """Compute nullspace of M over GF(2)."""
    n_samples, n = M.shape
    A = M.copy()
    pivot_rows = []
    col = row = 0
    while col < n and row < n_samples:
        for r in range(row, n_samples):
            if A[r, col]:
                A[[row, r]] = A[[r, row]]
                break
        if A[row, col]:
            for r in range(n_samples):
                if r != row and A[r, col]:
                    A[r] ^= A[row]
            pivot_rows.append((row, col))
            row += 1
        col += 1
    
    pivot_cols = {c for _, c in pivot_rows}
    free_cols = [c for c in range(n) if c not in pivot_cols]
    
    nullspace = []
    for free_c in free_cols:
        v = np.zeros(n, dtype=np.uint8)
        v[free_c] = 1
        for pr, pc in reversed(pivot_rows):
            val = 0
            for c in range(n):
                if c != pc and A[pr, c]:
                    val ^= v[c]
            v[pc] = val
        nullspace.append(v)
    return nullspace

def forge(s_bits):
    """Submit recovered s to get flag."""
    r = requests.post(f"{TARGET}/api/forge", json={"s": s_bits})
    return r.json()

if __name__ == "__main__":
    print("[*] Collecting oracle measurements...")
    M = collect_samples(n_batches=4, shots=256)
    print(f"[*] Got {len(M)} samples, computing nullspace...")
    
    ns = gf2_nullspace(M)
    print(f"[*] Nullity: {len(ns)}")
    
    if len(ns) == 1:
        s = ''.join(str(int(b)) for b in ns[0])
        print(f"[*] Recovered s: {s}")
        result = forge(s)
        if result.get("forged"):
            print(f"[+] FLAG: {result['flag']}")
        else:
            print("[-] Wrong s!")
    else:
        print(f"[-] Need more samples (nullity = {len(ns)})")
```

---

## References

- Simon's Algorithm: https://en.wikipedia.org/wiki/Simon%27s_algorithm
- Gaussian Elimination over GF(2): https://en.wikipedia.org/wiki/Gaussian_elimination
- Quantum Computing for Computer Scientists (Yanofsky & Mannucci, 2008)