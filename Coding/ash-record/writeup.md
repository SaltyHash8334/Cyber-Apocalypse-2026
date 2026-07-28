# Ash Record — Coding

**Category:** Coding  
**Challenge:** Cyber-Apocalypse-2026  
**Target:** `154.57.164.69:31329`  
**Flag:** `HTB{th3_h4ml3t_w4s_k3pt_n0t_burn3d}`

---

## Overview

A Python 3 web application hosts a competitive-programming style coding challenge. The server presents a problem description with a Monaco code editor and a `/run` API endpoint. The challenge asks for the longest prefix of a suspected extraction sequence that can be confirmed by matching residues with a minimum time gap constraint — a constrained subsequence matching problem.

## Problem Description

Elowen Ashglass has recovered residues across a hamlet site, each with a timestamp and a material type. She also has a suspected extraction sequence — the order materials would appear in if the population was moved out calmly, stage by stage. She needs the longest prefix of that sequence that can be confirmed: a subsequence of residues matching it in order, where no two consecutively matched residues are closer together than a minimum gap.

**Input format:**
1. First line: `N P min_gap` — number of residues, extraction sequence length, minimum required gap
2. Second line: `P` space-separated strings — the material types in the extraction sequence, in order
3. Next `N` lines: each is a timestamp (integer) and material type (string)

**Constraints:**
- `1 ≤ N ≤ 5000`
- `1 ≤ P ≤ 12`
- `1 ≤ min_gap ≤ 10`
- `1 ≤ timestamp ≤ 10000`
- Material type strings are lowercase, length ≤ 10

**Output:** A single integer — the maximum number of consecutive steps from the beginning of the extraction sequence that can be confirmed by a subsequence of residues satisfying the time gap constraint.

## Solution

### Algorithm: DP with Earliest-Timestamp Greedy

The problem is a variant of subsequence matching with a minimum-gap constraint between consecutive matches.

**Key insight:** For each prefix length `j`, we only need to track the *earliest* possible timestamp at which `seq[0..j]` can be confirmed. A smaller timestamp gives a larger set of eligible later residues (since `t_next ≥ dp[j] + min_gap` is easier to satisfy with a smaller `dp[j]`). This earliest-timestamp greedy is optimal — using an earlier timestamp for `dp[j]` never blocks extending to a longer prefix.

**Algorithm steps:**
1. Parse input: N, P, min_gap, extraction sequence, and residues
2. Sort residues by timestamp ascending
3. Initialize `dp[j] = INF` for `j = 0..P-1`, where `dp[j]` = earliest timestamp confirming `seq[0..j]`
4. For each residue `(t, m)` in timestamp order:
   - Iterate `j` from current max prefix down to 0 (right-to-left to avoid reusing the same residue)
   - If `m == seq[j]`:
     - `j == 0`: `dp[0] = min(dp[0], t)` — can always start a match here
     - `j > 0`: if `dp[j-1]` is finite and `t - dp[j-1] >= min_gap` and `t < dp[j]`, update `dp[j] = t`
5. The answer is the largest `k` where `dp[k-1]` is finite (first `k` elements confirmed)

**Complexity:** O(N × P) = 5000 × 12 = 60K operations — runs in milliseconds.

### Code

```python
import sys

def solve():
    data = sys.stdin.buffer.read().decode().strip().split()
    if not data:
        return
    
    N = int(data[0])
    P = int(data[1])
    min_gap = int(data[2])
    
    # Extraction sequence
    seq = data[3:3+P]
    
    # Residues
    residues = []
    idx = 3 + P
    for _ in range(N):
        t = int(data[idx])
        m = data[idx + 1]
        residues.append((t, m))
        idx += 2
    
    # Sort by timestamp
    residues.sort(key=lambda x: x[0])
    
    INF = 10 ** 9
    dp = [INF] * P  # dp[j] = earliest timestamp confirming seq[0..j]
    k = 0  # current confirmed prefix length
    
    for t, m in residues:
        # Try to extend each existing prefix, going right-to-left
        max_j = min(P - 1, k)  # may extend one past current k
        for j in range(max_j, -1, -1):
            if m != seq[j]:
                continue
            if j == 0:
                if t < dp[0]:
                    dp[0] = t
            else:
                if dp[j-1] < INF and t - dp[j-1] >= min_gap and t < dp[j]:
                    dp[j] = t
        
        # Update confirmed prefix length
        while k < P and dp[k] < INF:
            k += 1
    
    print(k)

if __name__ == "__main__":
    solve()
```

### Execution

Submitted via `POST /run` as JSON:

```json
{
  "code": "<python-code>",
  "language": "python"
}
```

### How We Solved It — Reasoning

1. **Identified the service type:** `curl` to the target port returned a full-page web app with a Monaco code editor, problem description (narrative "Ash Record" flavour text), and a `/run` API endpoint. The problem was a classic competitive-programming style coding challenge wrapped in a story.

2. **Parsed the problem:** The narrative described residues (timestamp + material type), an extraction sequence (ordered list of material types), and a minimum gap constraint. The goal was the longest prefix of the extraction sequence confirmable as a subsequence — a constrained longest-prefix subsequence match.

3. **Recognised the constraint pattern:** With `P ≤ 12` and `N ≤ 5000`, an O(N·P) DP was the obvious approach. The gap constraint meant we couldn't just greedily match the first available residue for each sequence position — consecutive matches needed to be spaced by at least `min_gap` timestamps apart.

4. **Proved the earliest-timestamp greedy:** For each prefix length `j`, we keep `dp[j]` = the earliest timestamp at which `seq[0..j]` can be matched. An earlier `dp[j]` gives a *larger* feasible window for the next match (everything at or after `dp[j] + min_gap`), so it can never block extension. This is a standard optimal substructure: to maximise how far you can reach, you want the earliest finishing time at each intermediate step.

5. **Implemented and verified:** Wrote the Python solver and tested against the provided example. The example (`ash, rope, oil, ash` with min_gap=3 across residues at timestamps 1, 4, 7, 10, 11) correctly returned 4. Submitted via `POST /run` — all test cases passed.

6. **Edge cases considered:** Duplicate material types (e.g. `ash` appearing twice in the sequence), residues not in timestamp order (sorted them first), multiple residues with the same timestamp, and the right-to-left iteration order to prevent using a single residue for multiple sequence positions in the same iteration.

## Caveats

- The service speaks **HTTP**, not raw TCP — `nc` returns a 400 error because it expects proper HTTP request syntax. Always use `curl` or direct HTTP POST.
- The right-to-left iteration in the DP is critical: without it, a single residue could be used to match both `seq[j-1]` and `seq[j]` in the same pass (since the update to `dp[j-1]` would make `dp[j-1]` available for `dp[j]` in the same iteration).
- The answer is the *prefix* length, not any subsequence — once a gap prevents confirming step `k`, we cannot claim steps `k+1` even if they happen to match individually. The problem is fundamentally about confirming an ordered prefix of the extraction scenario.

## Flag

| Flag | Method |
|------|--------|
| `HTB{th3_h4ml3t_w4s_k3pt_n0t_burn3d}` | DP with earliest-timestamp greedy subsequence matching |