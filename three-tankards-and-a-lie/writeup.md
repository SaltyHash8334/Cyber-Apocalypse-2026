# Three Tankards and a Lie — Coding Challenge

**CTF:** Cyber Apocalypse 2026  
**Category:** Coding  
**Challenge:** Three Tankards and a Lie  
**Flag:** `HTB{n3v3r_pl4ys_4lw4ys_w4tch3s}`

---

## Scenario

The Drowned Bell doesn't ask questions — which is exactly why Ferro Quicktongue runs his nightly game there. Three dented tankards, one bent copper chit, and a room full of off-duty gate clerks betting against a man whose hands move faster than his mouth.

What catches Rin Kagetsura's eye is a courier running the exact same swap-and-shuffle — except the thing sliding under those tankards isn't a copper chit, it's a folded scrap bound for someone at Suncourt. Every swap happens in plain sight — the only question left is which tankard the message is sitting under once the shuffling finally stops.

**Target:** `154.57.164.67:32182`

---

## Analysis

### Problem Specification

| Variable | Meaning |
|----------|---------|
| N | Number of tankards (positions 1..N) |
| M | Number of swaps performed |
| Q | Number of starting positions to track |

- Tankard `i` starts holding item `i`
- Each swap `a b` exchanges the contents of positions `a` and `b`
- For each query `p`, print the final position of the item that started at `p`

**Constraints:** N ≤ 2000, M ≤ 5000, Q ≤ 2000

### The Vulnerability

No vulnerability — this is a simulation problem. The challenge is implementing the swap-tracking efficiently enough to handle all test cases within the time limit.

A naive approach (scanning for item positions each query) would be O(M × N) or O(N × Q). We need **O(N + M + Q)** total.

---

## Solution

### Approach: Dual-Index Tracking

We maintain two complementary arrays:

1. **`pos[item]`** — the current position of each item
2. **`item_at[pos]`** — the item currently sitting at each position

Both start as identity mappings: `pos[i] = i` and `item_at[i] = i`.

On each swap `(a, b)`:

```
item_a = item_at[a]    # who's at position a?
item_b = item_at[b]    # who's at position b?
pos[item_a] = b        # item from position a moves to b
pos[item_b] = a        # item from position b moves to a
item_at[a], item_at[b] = item_at[b], item_at[a]  # swap position contents
```

**Complexity:** Each swap is O(1). Each query is O(1) — just `pos[p]`.

### Solution Code

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

### Execution

The challenge provides a Monaco code editor at the target URL. Code is submitted via POST to `/run`:

```
POST /run
Content-Type: application/json

{
  "code": "<source code>",
  "language": "python"
}
```

The server feeds random test cases to stdin, captures stdout, and compares against expected output. On all-pass, the response includes `challengeCompleted: true` and the flag.

---

## How We Solved It — Reasoning

The narrative — a shell game where the courier swaps tankards and the observer must track which one holds the message — maps directly to tracking items through position swaps.

The flavortext clue was explicit: *"working out where every marked tankard actually ends up, theirs included, is the only way to know which one held the real message."* This told us we needed to track final positions of specific starting items, not just the contents of a single position.

The dual-index approach was chosen for its O(1) per-swap and O(1) per-query performance. One array tracks where each item currently sits (`pos[item]`), and the other tracks what sits at each position (`item_at[pos]`). Every swap updates both in lockstep.

The key edge case is 1-indexing — positions are 1..N, not 0..N-1. Allocating `list(range(N + 1))` keeps index 0 as an unused sentinel, avoiding off-by-one errors throughout.

---

## Key Takeaways

- **Dual-index tracking** is the standard pattern for tracking items through position swaps — maintain both directions of the mapping
- **O(N + M + Q)** is achievable even with the simplest approach; no need for linked lists or complex data structures
- **1-indexed arrays in programming challenges** are a constant gotcha — always allocate N+1 slots
- **Reading input as split tokens** (`sys.stdin.read().split()`) handles any whitespace-separated format the test harness sends
