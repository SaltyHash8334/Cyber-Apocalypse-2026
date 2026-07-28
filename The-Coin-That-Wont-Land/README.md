# The Coin That Won't Land — Coding

**Category:** Coding (Quantum) | **Event:** Cyber Apocalypse 2026 | **Flag:** `HTB{epr_0ath_0p3ns_b0th_w4ys_3576458e7425f7afca49ed2f9a9d9a85}`

---

## Overview

**Target:** `154.57.164.78:30800` (Gunicorn HTTP server)

The Oathbinding Court runs a quantum bit-commitment protocol: you submit quantum circuits to entangle your "a" qubit with the Warden's "b" qubit, the court challenges you with a random measurement basis (Z or X) for all 8 strands, you measure your kept a-qubits in that basis, then open with the outcomes. The Warden measures b in the same basis — if all 8 strands match across all 32 rounds, you pass.

The challenge is to cheat: you must pass every round with probability 1, not 1/2²⁵⁶ (which is what honest random guessing would give).

## Protocol (from `/api/oath`)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/new` | POST | Open a hearing → `{token, strands:8, rounds:32}` |
| `/api/commit` | POST | Submit 8 quantum circuits → `{challenge, round}` |
| `/api/peek` | POST | Measure kept a-qubits in chosen basis → `{a_outcomes}` |
| `/api/open` | POST | Submit 8 measurement values → `{round_held, rounds_done}` |

**Gates available:** Single-qubit `I X Y Z H S SDG` on a or b, plus `CX` (control, target) among {a, b}.

**Challenge bit:** `c=0` → Z basis, `c=1` → X basis.

## Solution

### The Vulnerability: EPR Pairs Have Perfect Correlation in All Bases

The Bell state |Φ⁺⟩ = (|00⟩ + |11⟩)/√2 has the property that measuring both qubits in **any** basis yields identical outcomes. This is the EPR paradox: the correlations are basis-independent.

Construct |Φ⁺⟩ from |00⟩ with two gates:
1. **H on a** — puts a in superposition: (|0⟩ + |1⟩)/√2 ⊗ |0⟩
2. **CX(a→b)** — entangles: (|00⟩ + |11⟩)/√2 = |Φ⁺⟩

### Circuit

```python
bell = [["H", "a"], ["CX", "a", "b"]]
```

Apply this same circuit to all 8 strands every round.

### Solver

```python
import json, urllib.request

BASE = "http://154.57.164.78:30800"

def api(endpoint, method="GET", data=None):
    url = f"{BASE}{endpoint}"
    req = urllib.request.Request(url, method=method)
    if data is not None:
        req.add_header("Content-Type", "application/json")
        req.data = json.dumps(data).encode()
    with urllib.request.urlopen(req, timeout=30) as resp:
        return json.loads(resp.read())

new = api("/api/new", "POST")
token = new["token"]
bell = [["H", "a"], ["CX", "a", "b"]]

for _ in range(32):
    commit = api("/api/commit", "POST", {"token": token, "slots": [bell] * 8})
    basis = "X" if commit["challenge"] == 1 else "Z"
    peek = api("/api/peek", "POST", {"token": token, "basis": basis})
    result = api("/api/open", "POST", {"token": token, "values": peek["a_outcomes"]})
    if "flag" in result:
        print(result["flag"])
```

### Flag

```
HTB{epr_0ath_0p3ns_b0th_w4ys_3576458e7425f7afca49ed2f9a9d9a85}
```

---

## How We Solved It — Reasoning

### 1. Service Discovery

`nc` to port 30800 hung — classic sign of an HTTP service. `nmap -sV` identified Gunicorn (Python WSGI server). `curl /` returned a rich HTML page with a Monaco-style interactive explainer. The JavaScript at `/app.js` revealed the full API surface: `/api/new`, `/api/commit`, `/api/peek`, `/api/open`, and `/api/oath`.

### 2. Protocol Analysis

The `/api/oath` endpoint documented a quantum bit-commitment ritual:

- **Commit phase:** You submit 8 quantum circuits (2-qubit, a and b). You keep a, Warden seals b. Returns challenge bit c.
- **Peek phase:** You measure your a-qubits in basis c (Z if c=0, X if c=1).
- **Open phase:** You submit the measurement outcomes. Warden measures b in basis c and checks they match.

The challenge name — "The Coin That Won't Land" — was the first clue. In quantum coin-flipping and bit-commitment, the "coin" is the measurement basis the verifier picks after you commit. A fair coin lands randomly; the question is whether you can force it to always "land your way."

### 3. Recognising the Quantum Vulnerability

The 8 strands × 32 rounds structure = 256 independent bits to guess. The probability of guessing all correctly is 2⁻²⁵⁶ — impossible.

But quantum bit commitment has a famous impossibility result (Mayers 1997, Lo-Chau 1997): **unconditionally secure quantum bit commitment is impossible** because an adversary with EPR pairs can always delay measurement until after the basis is revealed.

The key insight: if Alice and Bob share a Bell pair |Φ⁺⟩ = (|00⟩ + |11⟩)/√2, then measuring Alice's qubit in **any** basis and Bob's qubit in the **same** basis always yields identical outcomes. The correlation is basis-independent.

### 4. Exploit Strategy

1. For every strand in every round, prepare the Bell state |Φ⁺⟩ using H on a followed by CX(a→b).
2. Wait for the court's challenge bit c.
3. Measure a in the basis dictated by c (Z if c=0, X if c=1).
4. Submit those measurement outcomes — they will **always** match what the Warden measures on b.

This is the "EPR oath" referenced in the flag: Einstein-Podolsky-Rosen pairs "open both ways" — Z or X, the outcomes match perfectly.

### 5. Key Insight

The flag itself — `epr_0ath_0p3ns_b0th_w4ys` — confirms the intended solution path. The EPR paradox means entangled particles exhibit correlations regardless of which measurement basis you choose. By preparing Bell states and measuring only after the basis is revealed, we bypass the randomness that an honest protocol participant would face.

---

## Execution

```bash
python3 solver.py
```

Output:
```
Round 1/32: challenge=1(X) held=True
...
Round 32/32: challenge=0(Z) held=True
*** FLAG: HTB{epr_0ath_0p3ns_b0th_w4ys_3576458e7425f7afca49ed2f9a9d9a85} ***
```

---

## Caveats

1. **`nc` hangs — it's HTTP.** Always `nmap -sV` first. The Werkzeug/Gunicorn detection is the signal to use `curl`.
2. **API key names matter.** The commit response uses `challenge` (not `c`), and peek returns `a_outcomes` (not `values` or `outcomes`). One wrong key name and the open endpoint rejects with a 400.
3. **The Warden measures b, not a.** The protocol documentation says "the Warden measures each b in basis c." The values you submit must predict *b's* measurement, not a's. Bell state correlations make them identical, so the distinction doesn't matter in practice — but understanding the data flow is essential for reasoning about the exploit.
4. **Circuit is always |Φ⁺⟩.** No need to vary circuits per round or strand. The same Bell state circuit works every time because the correlations are basis-independent — that's the whole point of the EPR attack.

---

## Flag Summary

| Flag | Method |
|------|--------|
| `HTB{epr_0ath_0p3ns_b0th_w4ys_3576458e7425f7afca49ed2f9a9d9a85}` | Complete 32 rounds using Bell state (EPR) correlations |
