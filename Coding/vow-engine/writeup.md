# Vow Engine — Coding Challenge

**Category:** Coding Challenge / Graph Algorithms  
**Target:** `154.57.164.81:31944`  
**Flag:** `HTB{th3_b3ll_r1ngs_wr0ng_0n_purp0s3}`

---

## How We Solved It — Reasoning

### Overview

The challenge presents a graph problem framed in fantasy lore. We have:
- **N witnesses** (nodes, 0 to N-1), up to 1500
- **M seal-marks** (undirected edges with weight w ∈ [0, 63])
- **Q challenges** (queries) asking: *"Is cadence value T achievable on ANY oath-path between witnesses u and v?"*

Edge weights represent XOR "cadence keys." The "weight of an oath" along a path is the XOR of every edge weight on that path. Since the graph has redundant links (cycles), there can be multiple paths between two witnesses, each producing a different XOR value.

### Key Insight — XOR Paths and Linear Basis

This is a classic **XOR path problem on undirected graphs**. The core observation:

1. Pick an arbitrary spanning tree/BFS tree for each connected component.
2. For each node, compute `xor_to_root[node]` — the XOR from the component's root to that node along tree edges.
3. The tree-path XOR between nodes `u` and `v` is `xor_to_root[u] ^ xor_to_root[v]`.
4. Every **non-tree edge** creates a cycle. The XOR value of going around that cycle is:
   ```
   cycle_xor = xor_to_root[u] ^ w ^ xor_to_root[v]
   ```
5. The set of achievable XOR values between `u` and `v` is:
   ```
   { base_xor ^ C | C is any XOR combination of cycle basis values }
   ```
   where `base_xor = xor_to_root[u] ^ xor_to_root[v]`.

So the problem reduces to:
- Build a **linear basis (XOR basis)** over GF(2) from all cycle XOR values in each component.
- For each query `(u, v, T)`, check if `T ^ base_xor` can be represented by the component's cycle basis.

Since edge weights are only 6-bit (0–63), the basis has at most 6 vectors.

### Algorithm

1. **DFS/BFS** to assign `comp_id` (component) and `xor_to_root` (tree-path XOR from root) to every node.
2. **Collect cycles**: For each input edge `(u, v, w)` in the same component, compute `cycle_xor = xor_to_root[u] ^ w ^ xor_to_root[v]`. Insert non-zero values into a 6-bit XOR linear basis per component.
3. **Query**: For each `(u, v, T)`:
   - If `u` and `v` are in different components → `NO` (no path exists).
   - `target = T ^ xor_to_root[u] ^ xor_to_root[v]`
   - Check if `target` can be reduced to 0 using the component's basis → `YES`/`NO`.

### Implementation

```python
import sys

def solve():
    data = sys.stdin.buffer.read().split()
    it = iter(data)

    N = int(next(it))
    M = int(next(it))
    Q = int(next(it))

    edges = []
    adj = [[] for _ in range(N)]
    for _ in range(M):
        u = int(next(it))
        v = int(next(it))
        w = int(next(it))
        edges.append((u, v, w))
        adj[u].append((v, w))
        adj[v].append((u, w))

    xor_to_root = [0] * N
    comp_id = [-1] * N

    # Iterative DFS to assign xor_to_root and component IDs
    for start in range(N):
        if comp_id[start] != -1:
            continue
        stack = [(start, -1)]
        comp_id[start] = start
        while stack:
            u, p = stack.pop()
            for v, w in adj[u]:
                if v == p:
                    continue
                if comp_id[v] == -1:
                    comp_id[v] = start
                    xor_to_root[v] = xor_to_root[u] ^ w
                    stack.append((v, u))

    # Build XOR linear basis per component (6-bit, values 0-63)
    comp_basis = {}
    for u, v, w in edges:
        if comp_id[u] == comp_id[v]:
            cycle_xor = xor_to_root[u] ^ w ^ xor_to_root[v]
            if cycle_xor != 0:
                cid = comp_id[u]
                if cid not in comp_basis:
                    comp_basis[cid] = [0] * 6
                basis = comp_basis[cid]
                x = cycle_xor
                for b in range(5, -1, -1):
                    if (x >> b) & 1:
                        if basis[b]:
                            x ^= basis[b]
                        else:
                            basis[b] = x
                            break

    out_lines = []
    for _ in range(Q):
        u = int(next(it))
        v = int(next(it))
        T = int(next(it))

        if comp_id[u] != comp_id[v]:
            out_lines.append("NO")
            continue

        target = T ^ xor_to_root[u] ^ xor_to_root[v]

        if target == 0:
            out_lines.append("YES")
            continue

        cid = comp_id[u]
        if cid not in comp_basis:
            out_lines.append("NO")
            continue

        basis = comp_basis[cid]
        x = target
        for b in range(5, -1, -1):
            if (x >> b) & 1:
                if basis[b]:
                    x ^= basis[b]
                else:
                    break
        out_lines.append("YES" if x == 0 else "NO")

    sys.stdout.write("\n".join(out_lines))


if __name__ == "__main__":
    solve()
```

### Why This Works

- **Tree paths** give us a single fixed XOR value between any two nodes.
- **Cycles** let us flip additional bits — going around a cycle XORs its cycle value into the path.
- The **linear basis** computes the vector space (over GF(2) with 6-bit XOR) spanned by all cycles in the component.
- Any achievable XOR between `u` and `v` is `base_xor ^ cycle_combination`. Since we can represent `cycle_combination` using the basis, checking if `T ^ base_xor` is representable answers the query.

### Verification with Example

```
5 5 4
0 1 3
1 2 5
2 3 4
3 4 7
0 2 10
0 3 2   → YES  (path 0→1→2→3: 3^5^4=2)
0 3 5   → NO   (no path yields XOR 5)
0 1 15  → YES  (path 0→2→1: 10^5=15)
0 4 4   → NO   (no path yields XOR 4)
```

---

### Flag

```
HTB{th3_b3ll_r1ngs_wr0ng_0n_purp0s3}
```

### Caveats

- Edge weights are only 6-bit (0–63), so the linear basis caps at 6 elements — the check is O(1) per query.
- Tree edges produce `cycle_xor = 0` (since `xor_to_root[u] ^ w ^ xor_to_root[v] = 0` by construction), which is harmlessly discarded by the `if cycle_xor != 0` guard.
- The graph may be disconnected; each component gets its own basis, and cross-component queries immediately return `NO`.
