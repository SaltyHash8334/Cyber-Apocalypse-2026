# Toll Schedule — Coding

**Category:** Coding  
**Challenge:** Cyber-Apocalypse-2026  
**Target:** `154.57.164.65:32264`  
**Flag:** `HTB{th3_p4ss_pr3f3rs_0n3_b4nn3r}`

---

## Overview

A Werkzeug/Python 3.14 web application hosts a competitive-programming style coding challenge. The server presents a problem description with a Monaco code editor and a `/run` API endpoint. The challenge asks for the minimum total waiting time when assigning convoys to checkpoint clearances — an optimal matching (assignment) problem.

## Problem Description

Stonepass checkpoint admits convoys through available clearances. Each convoy arrives at a known time; each clearance opens at a known time and can handle exactly one convoy, no earlier than the convoy's arrival. Find the legal assignment of clearances to convoys that minimises the column's total waiting time.

**Input format:**
1. First line: `N G` — number of convoys and number of clearances (G ≥ N)
2. Second line: `N` space-separated integers — arrival times of each convoy (any order)
3. Third line: `G` space-separated integers — opening times of each clearance (any order)

**Constraints:**
- `1 ≤ N ≤ 1200`
- `N ≤ G ≤ 1500`
- `1 ≤ arrival time, opening time ≤ 200`
- A valid assignment is guaranteed

**Output:** A single integer — the minimum total waiting time across all convoys, where waiting time for a convoy = clearance opening time − convoy arrival time.

## Solution

### Algorithm: Greedy Sorted Matching

The problem is a minimum-cost bipartite matching where the cost of matching convoy `i` to clearance `j` is `clearance_j − arrival_i` (only feasible when `clearance_j ≥ arrival_i`). Minimising total wait is equivalent to minimising the sum of selected clearance times (since `sum(arrivals)` is constant), subject to the feasibility constraint.

**Key insight:** Sort both arrays ascending, then greedily pair the earliest remaining convoy with the earliest feasible clearance. The rearrangement inequality proves this is optimal — for two sorted sequences, the sum of pairwise differences is minimised when both are ordered the same way.

**Algorithm steps:**
1. Sort `arrivals` ascending
2. Sort `clearances` ascending
3. Two‑pointer scan: for each convoy in order, advance the clearance pointer past any openings that are *too early* (before the convoy's arrival), then assign the next feasible clearance and accumulate the wait
4. Print the total

### Code

```python
def solve():
    import sys
    data = sys.stdin.read().strip().split()
    if not data:
        return
    it = iter(data)
    N = int(next(it))
    G = int(next(it))
    arrivals = [int(next(it)) for _ in range(N)]
    clearances = [int(next(it)) for _ in range(G)]

    arrivals.sort()
    clearances.sort()

    total_wait = 0
    j = 0
    for a in arrivals:
        while j < G and clearances[j] < a:
            j += 1
        total_wait += clearances[j] - a
        j += 1

    print(total_wait)

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

1. **Identified the service type:** `nmap -sV` on the target revealed an open port running `Werkzeug httpd 3.1.3 (Python 3.14.0)`. This immediately told us we were dealing with a web-based coding challenge platform, not a raw binary or interactive service.

2. **Understood the problem structure:** Fetching `GET /` returned a full-page web app with a Monaco code editor, problem description, and a `/run` API endpoint. The problem description was flavoured narrative text ("Stonepass checkpoint", "convoys", "clearances") wrapping a canonical scheduling/matching problem.

3. **Modelled as minimum-cost assignment:** Reading the constraints — N convoys, G ≥ N clearances, each clearance must open at or after the convoy's arrival — the objective "minimise total waiting time" maps directly to minimising `sum(clearance_j − arrival_i)` across assigned pairs. Since `sum(arrival_i)` is constant (all convoys are assigned), this is equivalent to selecting N clearances with minimum total opening time that can be feasibly paired with the N convoys.

4. **Proved greedy optimality:** The classic rearrangement inequality states that for two sorted sequences, the sum of pairwise differences is minimised when both are ordered the same way. The two-pointer greedy — assign the earliest convoy to the earliest clearance that opens at or after its arrival — is the natural consequence. An exchange argument confirms: if any optimal solution pairs a later convoy with an earlier clearance than an earlier convoy, swapping the assignments cannot increase total cost and preserves feasibility.

5. **Written and submitted:** Composed the Python solver (sorting + two-pointer scan, O((N+G) log N)), POSTed to `/run`. The server passed all test cases and returned the flag in `challengeCompleted: true`.

## Caveats

- The service speaks **HTTP**, not raw TCP — `nc` connections hang because the server waits for a proper request line and headers. Always probe with `curl` or `nmap -sV` after a `nc` timeout.
- The `G ≥ N` constraint and the guarantee of a valid assignment mean the greedy never stalls; if either were relaxed the algorithm would need a more expensive DP or Hungarian approach.
- The two-pointer pass skips clearances that open *before* a convoy's arrival — those were left over from earlier convoys and are permanently unusable (a later convoy arrives even later, making the gap worse), so they can be safely discarded.

## Flag

| Flag | Method |
|------|--------|
| `HTB{th3_p4ss_pr3f3rs_0n3_b4nn3r}` | Greedy sorted matching |