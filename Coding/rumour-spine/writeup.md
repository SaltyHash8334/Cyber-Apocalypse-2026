# Rumour Spine — Coding Challenge

**CTF:** Cyber Apocalypse 2026  
**Category:** Coding  
**Challenge:** Rumour Spine  
**Flag:** `HTB{0n3_c00rd1n4t3d_m0m3nt}`

---

## Scenario

Suncourt doesn't march first — it primes first. Miren Vale has traced the Quiet March's invitation chain: a coordinator who never touches ash, only schedules, feeding a priming signal through a network of quiet hands — candle-buyers, shrine-attendants, penitent-scribes — until a target district falls still the night before the March arrives. Every hand who passes the signal on is a link Miren can expose.

Miren means to break the chain before the district goes quiet. She can expose any quiet hand along the way — one flagged, one route burned, at the cost of one favour she can never call in again. The coordinator and the target district can't be touched directly: too visible, too soon. What is the minimum number of quiet hands that must be exposed to sever every path the priming signal could still travel?

**Target:** `154.57.164.77:31419`

---

## Analysis

### Problem Specification

| Variable | Meaning |
|----------|---------|
| N | Number of quiet hands (nodes, labelled 0..N-1) |
| E | Number of directed links (edges) |
| S | Coordinator node (source, always 0) |
| T | Target district node (sink, always N-1) |

- The priming signal travels along directed edges from S to T
- We must find the **minimum number of intermediate nodes** to remove to disconnect all paths from S to T
- S and T themselves cannot be removed

**Constraints:** 6 ≤ N ≤ 120, 1 ≤ E ≤ 360, S = 0, T = N-1

### The Vulnerability

No vulnerability — this is an algorithmic graph problem. The challenge is recognising that the minimum number of nodes to remove to disconnect a directed graph is the **minimum vertex cut**, which reduces to **minimum edge cut** (max flow) via vertex splitting.

---

## Solution

### Approach: Minimum Vertex Cut via Max Flow (Dinic's Algorithm)

The minimum s-t vertex cut (minimum vertices whose removal disconnects source from sink, excluding source and sink themselves) is solved by:

1. **Vertex splitting** — Each node v is split into `v_in` and `v_out`, connected by an edge of capacity:
   - `INF` for S and T (they can't be removed)
   - `1` for all other nodes (exposing that hand costs one favour)

2. **Edge transformation** — Each original directed edge `u → v` becomes `u_out → v_in` with `INF` capacity

3. **Max flow** — Source = `S_out`, Sink = `T_in`. The max flow value equals the minimum vertex cut

This works because each intermediate node becomes a "pipe" with capacity 1 — the max flow algorithm can push at most 1 unit through it, and the min-cut will select nodes with the smallest total capacity. Since S and T have infinite capacity, they are never selected.

**Algorithm:** Dinic's max flow — O(E·V²) where V ≤ 240 (N × 2 after splitting). Well within limits for the constraint size.

### Solution Code

```python
import sys

def solve():
    data = sys.stdin.read().strip().split()
    if not data:
        return
    it = iter(data)
    N = int(next(it))
    E = int(next(it))
    S = int(next(it))
    T = int(next(it))

    V = N * 2
    INF = 10**9

    class Dinic:
        def __init__(self, n):
            self.n = n
            self.graph = [[] for _ in range(n)]

        def add_edge(self, fr, to, cap):
            self.graph[fr].append([to, cap, len(self.graph[to])])
            self.graph[to].append([fr, 0, len(self.graph[fr]) - 1])

        def bfs(self, s, t):
            level = [-1] * self.n
            q = [s]
            level[s] = 0
            for v in q:
                for to, cap, rev in self.graph[v]:
                    if cap > 0 and level[to] < 0:
                        level[to] = level[v] + 1
                        q.append(to)
            self.level = level
            return level[t] >= 0

        def dfs(self, v, t, f):
            if v == t:
                return f
            for i in range(self.it[v], len(self.graph[v])):
                self.it[v] = i
                to, cap, rev = self.graph[v][i]
                if cap > 0 and self.level[v] < self.level[to]:
                    ret = self.dfs(to, t, min(f, cap))
                    if ret > 0:
                        self.graph[v][i][1] -= ret
                        self.graph[to][rev][1] += ret
                        return ret
            return 0

        def max_flow(self, s, t):
            flow = 0
            while self.bfs(s, t):
                self.it = [0] * self.n
                while True:
                    f = self.dfs(s, t, INF)
                    if f == 0:
                        break
                    flow += f
            return flow

    dinic = Dinic(V)

    # Vertex splitting edges
    for v in range(N):
        cap = INF if v == S or v == T else 1
        dinic.add_edge(v * 2, v * 2 + 1, cap)

    # Original edges
    for _ in range(E):
        u = int(next(it))
        v = int(next(it))
        dinic.add_edge(u * 2 + 1, v * 2, INF)

    source = S * 2 + 1
    sink = T * 2

    ans = dinic.max_flow(source, sink)
    print(ans)

if __name__ == '__main__':
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

The flavortext maps cleanly to a graph theory problem: the coordinator (source S) feeds a "priming signal" through a network of "quiet hands" (nodes) along directed links (edges) until it reaches a target district (sink T). For each hand Miren exposes (node removed), the paths through that hand are severed. She can't touch S or T directly. The question — *"What is the minimum number of quiet hands that must be exposed to sever every path?"* — is the textbook definition of the **minimum vertex cut** problem.

The key recognition was that vertex cut in a directed graph reduces to edge cut via **vertex splitting**: each node becomes an edge with capacity 1 (the cost of removing it), and the max flow through this transformed graph gives the answer. Dinic's algorithm was chosen for its O(E·V²) performance on the small constraint size (N ≤ 120, E ≤ 360).

The example confirmed the approach — a 7-node diamond-shaped graph where exposing nodes 1 and 2 (answer 2) severs all four paths from 0 to 6, but no single node lies on all paths.

---

## Key Takeaways

- **Minimum vertex cut** in directed graphs reduces to max flow via **vertex splitting** — each node v becomes v_in → v_out with capacity equal to the cost of removing it
- **S and T must have INF capacity** in the split to prevent the algorithm from selecting them (they can't be removed)
- **Dinic's algorithm** is efficient enough for N ≤ 120; no need for Push-Relabel or other advanced max-flow algorithms
- **Reading all input at once** (`sys.stdin.read().split()`) handles any whitespace separation the test harness sends
