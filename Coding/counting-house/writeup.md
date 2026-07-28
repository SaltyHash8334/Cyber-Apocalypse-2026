# The Counting House — Coding

**Category:** Coding / Quantum Cryptography  
**Challenge:** Cyber-Apocalypse-2026  
**Target:** `154.57.164.77:30867`  
**Flag:** `HTB{f0rg3d_r34d_4nd_s3ttl3d_t0_th3_s1lv3r_8179fec656e132293543d3c0c795afbf}`

## Overview

The Counting House exposes a stateful HTTP API for a sealed-bid auction. To claim the deed, we must:

1. Forge a bearer note to obtain a seat.
2. Read the six sealed bids.
3. Pass a 24-round seal check.
4. Submit a forged note for the exact clearing price.

The challenge combines GF(2) linear algebra, quantum basis leakage, and an EPR attack against a bit-commitment-style protocol.

## Reconnaissance

Nmap identified a Gunicorn HTTP service:

```text
30867/tcp open  http  Gunicorn
```

`GET /api/market` disclosed the important parameters:

```json
{
  "note": {"checks": 4, "entry_value": 4919, "qubits": 8},
  "book": {"bits": 16, "rivals": 6},
  "settlement": {"seal_rounds": 24, "seal_strands": 8, "tries": 6}
}
```

## Forging the Bearer Note

For a value `v`, the note is the uniform superposition over `ker(H_v)`, where `H_v` is a 4-row, 8-column binary matrix. Row `i` is derived from:

```text
sha256("eastreach-note-v{v}-{i}")
```

A key implementation detail is that the server's “8-bit truncation” uses the **last digest byte**:

```python
h = hashlib.sha256(f"eastreach-note-v{value}-{i}".encode()).digest()
row = [(h[-1] >> bit) & 1 for bit in range(8)]
```

Rows that do not increase the GF(2) rank are skipped. Gaussian elimination then provides a nullspace basis. The preparation circuit is constructed by:

- Applying `H` to every free-column qubit.
- Applying `CX` from each free column to every pivot column set in its generator vector.

The note circuit uses nested arrays and integer qubits `0..7`, for example:

```json
[["H", 2], ["CX", 2, 1]]
```

Submitting the circuit for the entry value `4919` to `/api/enter` returned:

```json
{"seated": true}
```

## Reading the Sealed Book

The book endpoint measures one bidder/position qubit in either the Z or X basis:

```json
{
  "token": "...",
  "bidder": 0,
  "position": 0,
  "basis": "Z",
  "shots": 1
}
```

Each position's basis can be identified by repeated Z measurements on bidder 0:

- Repeated identical results indicate a Z-basis bit.
- Varying results indicate an X-basis bit.

The important side channel is that each `/api/book` request re-prepares the state. The same position can therefore be measured repeatedly across separate requests without a prior measurement collapsing future requests.

The basis pattern is position-dependent and changes with the session, but remains consistent across bidders within that session. After identifying the pattern, all six bids can be reconstructed using the appropriate basis.

The work-tape is **MSB-first**: position 0 is the most significant bit.

```python
bid_value |= bit << (15 - position)
```

Using `bit << position` produces incorrect prices. The clearing price is the maximum reconstructed bid.

## Breaking the Seal Check

The seal check has 24 rounds, each containing eight two-qubit strands. The verifier commits to a random Z/X challenge after the circuits are submitted, then checks the outcomes returned by the client.

This is vulnerable to the standard EPR attack. For every strand, submit a Bell-state preparation circuit:

```json
[["H", "a"], ["CX", "a", "b"]]
```

This prepares:

```text
|Φ⁺⟩ = (|00⟩ + |11⟩) / √2
```

The Bell state has perfect correlations in both bases:

```text
|Φ⁺⟩ = (|00⟩ + |11⟩) / √2
     = (|++⟩ + |--⟩) / √2
```

Therefore, after the challenge is revealed, measure the kept `a` qubits in the requested basis and return those outcomes:

```python
bell = [["H", "a"], ["CX", "a", "b"]]
slots = [bell.copy() for _ in range(8)]

for _ in range(24):
    commit = post("/api/seal/commit", {"token": token, "slots": slots})
    basis = "X" if commit["challenge"] == 1 else "Z"
    peek = post("/api/seal/peek", {"token": token, "basis": basis})
    post("/api/seal/open", {
        "token": token,
        "values": peek["a_outcomes"]
    })
```

All 24 rounds held and the response reported `sealed: true`.

## Settlement

The successful run reconstructed the following bid interpretations:

```text
LSB-first values: [5374, 35326, 5521, 14762, 4687, 27996]
MSB-first values: [32552, 32657, 35240, 21916, 62024, 15030]
```

Thus the correct clearing price was:

```text
62024
```

A second forged CSS note was generated for value `62024` and submitted to `/api/settle` after the seal was passed. The server returned `settled: true` and released the deed containing the flag.

## How We Solved It — Reasoning

1. **Identified the protocol:** The page documented a stateful API rather than a normal `/run` coding task, so the solution had to interact with the auction stages directly.
2. **Reduced note forgery to GF(2) algebra:** The public note construction defines a kernel state, allowing a nullspace basis to be converted into a Hadamard/CNOT preparation circuit.
3. **Resolved the hash ambiguity experimentally:** First-byte truncation produced valid-looking circuits but never seated the session. Testing the digest positions showed that the server uses `sha256(...).digest()[-1]`.
4. **Exploited repeated book preparation:** Repeated measurements distinguished deterministic Z-basis positions from probabilistic X-basis positions. The same discovered basis pattern was then used for every bidder.
5. **Fixed the bit ordering:** Initial clearing prices were rejected. Comparing LSB-first and MSB-first reconstructions showed that position 0 is the most significant bit; the MSB-first maximum was `62024`.
6. **Defeated the seal with entanglement:** Bell pairs provide identical outcomes regardless of whether the verifier chooses Z or X, so the challenge can be answered after it is disclosed.
7. **Settled exactly:** The note forged for the MSB-first maximum bid passed the final verification and returned the deed flag.

## Pitfalls

- Use the **last** SHA-256 byte, not the first.
- Use nested gate arrays, not strings such as `"H 4"`.
- Use integer qubit indices for bearer notes.
- Use string labels `"a"` and `"b"` for seal strands.
- Discover basis positions in the same session used to read bids.
- Encode bids MSB-first with `bit << (15 - position)`.
- The session allows only six settlement attempts, so do not guess prices.

## Flag

```text
HTB{f0rg3d_r34d_4nd_s3ttl3d_t0_th3_s1lv3r_8179fec656e132293543d3c0c795afbf}
```

## Verification

The flag was obtained from a live successful settlement response:

```json
{
  "deed": "HTB{f0rg3d_r34d_4nd_s3ttl3d_t0_th3_s1lv3r_8179fec656e132293543d3c0c795afbf}",
  "settled": true
}
```
