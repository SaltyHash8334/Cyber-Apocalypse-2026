# Three Tankards and a Lie — Coding Challenge

**Category:** Coding / Simulation  
**Challenge:** Three Tankards and a Lie (Cyber Apocalypse 2026)  
**Target:** `154.57.164.67:32182`  
**Flag:** `HTB{n3v3r_pl4ys_4lw4ys_w4tch3s}`

---

## Overview

A shell-game simulation: N tankards (positions 1..N) each start holding their own item. M swaps exchange the contents of two positions. Q queries ask where a given starting item ends up after all swaps.

Essentially: given a sequence of position swaps and a list of starting items, compute the final position of each queried item.

---

## How We Solved It — Reasoning

### Understanding the Problem

The narrative sets up a shell game — three tankards, a bent copper chit, and a courier running swaps. The key insight from the flavor text:

> "Rin doesn't care about their coin — but working out where every marked tankard actually ends up, theirs included, is the only way to know which one held the real message."

This maps directly to tracking items through a sequence of position swaps. The input format confirmed it:

- `N M Q`: N tankards, M swaps, Q queries
- Each swap line: `a b` — swap contents of positions `a` and `b`
- Each query line: `p` — "where does the item that started at position `p` end up?"

### Approach

We used dual-index tracking with two arrays:

1. **`pos[item]`** — the current position of each item (indexed by item ID)
2. **`item_at[pos]`** — the item currently sitting at each position (indexed by position)

**Initial state** (N=5):  
- `pos = [0, 1, 2, 3, 4, 5]` (item `i` starts at position `i`)  
- `item_at = [0, 1, 2, 3, 4, 5]` (position `i` holds item `i`)

**Per swap (a, b):**
```
item_a = item_at[a]    # who's at position a?
item_b = item_at[b]    # who's at position b?
pos[item_a] = b        # item from position a now goes to b
pos[item_b] = a        # item from position b now goes to a
item_at[a], item_at[b] = item_at[b], item_at[a]  # swap position contents
```

Each swap is O(1).

**After all M swaps:** for each query `p`, simply read `pos[p]` — O(1) per query.

Total complexity: **O(N + M + Q)** time, **O(N)** space.

### Edge Cases

- `M = 0`: no swaps — every query `p` returns `p`
- Large ranges: N ≤ 2000, M ≤ 5000 — well within Python's capacity

---

## Solution Code (Python)

```python
import sys

def solve():
    data = sys.stdin.read().strip().split()
    if not data:
        return
    it = iter(data)
    N = int(next(it))
    M = int(next(it))
    Q = int(next(it))
    
    pos = list(range(N + 1))      # pos[item] = current position
    item_at = list(range(N + 1))  # item_at[pos] = item at that position
    
    for _ in range(M):
        a = int(next(it))
        b = int(next(it))
        item_a = item_at[a]
        item_b = item_at[b]
        pos[item_a] = b
        pos[item_b] = a
        item_at[a], item_at[b] = item_at[b], item_at[a]
    
    out = []
    for _ in range(Q):
        p = int(next(it))
        out.append(str(pos[p]))
    sys.stdout.write("\n".join(out))

if __name__ == "__main__":
    solve()
```

---

## Execution

The challenge provides a Monaco code editor at `http://154.57.164.67:32182/`. The code runs by POSTing to `/run`:

```
POST /run
Content-Type: application/json

{
  "code": "<source code>",
  "language": "python"
}
```

The server feeds test cases (random input) to stdin, captures stdout, and compares against expected output. On all-pass, the response includes `challengeCompleted: true` and the flag.

---

## Flag

| Field | Value |
|-------|-------|
| **Flag** | `HTB{n3v3r_pl4ys_4lw4ys_w4tch3s}` |
| **Method** | Simulated the swap sequence with dual-index tracking |

---

## Caveats

1. **Reading input as split tokens** is critical — the server provides random test data, not the example input. Using `sys.stdin.read().split()` handles any whitespace-separated format.
2. **1-indexed arrays** matter — positions are 1..N, not 0..N-1. Allocate `list(range(N + 1))` to keep zero as a sentinel.
3. **The flag is a play on the shell game** — "never plays, always watches" (Rin's approach: observe the swaps instead of betting).
