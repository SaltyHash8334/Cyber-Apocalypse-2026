# Granary Seal — Coding

**Category:** Coding  
**Challenge Cyber-Apocalypse-2026**  
**Target:** `154.57.164.67:31840`  
**Flag:** `HTB{th3_0ld_s3qu3nc3_n3v3r_h3s1t4t3s}`

---

## Overview

A Flask/Werkzeug web application hosts a competitive-programming style challenge. The server provides a problem description, a Monaco code editor, and a `/run` API endpoint that executes submitted code against hidden test cases. The challenge validates ration orders against a set of known custodians.

## Problem Description

The palace granaries accept ration orders passed through three hands: a **clerk** (who raises the order), a **countersigner** (who clears it), and a **courier** (who carries it). Since the Signet shattered, forged orders slip through with convincing wax and forged signatures. Lysa Harrowmere maintains three custody rolls — one per role — listing the names the gatehouse has actually watched work.

**Input format:**
1. Integer `C` — number of clerks on the custody roll  
2. `C` lines — clerk names  
3. Integer `CS` — number of countersigners on the custody roll  
4. `CS` lines — countersigner names  
5. Integer `R` — number of couriers on the custody roll  
6. `R` lines — courier names  
7. Integer `N` — number of orders in the batch  
8. `N` lines — each with three space-separated names: `clerk countersigner courier`

Constraints: `1 ≤ C, CS, R ≤ 20`, `1 ≤ N ≤ 5000`

**Output:** A single integer — the count of orders where every hand appears on its role's custody roll.

## Solution

The problem is a straightforward set-membership check. Read the names into three Python sets (one per role), iterate the N orders, and count how many have all three names present in their respective sets.

### Code

```python
def solve():
    import sys
    data = sys.stdin.read().splitlines()
    it = iter(data)
    
    C = int(next(it))
    clerks = set(next(it).strip() for _ in range(C))
    
    CS = int(next(it))
    countersigners = set(next(it).strip() for _ in range(CS))
    
    R = int(next(it))
    couriers = set(next(it).strip() for _ in range(R))
    
    N = int(next(it))
    count = 0
    for _ in range(N):
        line = next(it).strip()
        c, cs, cr = line.split()
        if c in clerks and cs in countersigners and cr in couriers:
            count += 1
    
    print(count)

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

1. **Discovered the service:** Initial TCP connection with `nc` timed out. Sending `test\n` returned an HTTP 400 error page, revealing a **Werkzeug/Python HTTP server**.

2. **Retrieved the full page:** A proper `GET / HTTP/1.0` request returned the challenge description with example input/output, a Monaco editor, and the `/run` API endpoint.

3. **Analyzed the problem:** The description was pure flavour text — the core was simple set membership. Three categories of custodians (clerks, countersigners, couriers), each with a named roster, and N orders each referencing three names. Valid orders have all three names in their respective rosters.

4. **Written and submitted:** Composed the Python solution, POSTed it as JSON to `/run`. The server accepted it immediately, passed all test cases, and returned the flag.

## Caveats

- The service requires proper HTTP framing (`\r\n` line endings) — raw `nc` connections hang waiting for the request line.
- The `/run` endpoint accepts plain JSON and returns either a failed test case with details, or `challengeCompleted: true` with the flag.

## Flag

| Flag | Method |
|------|--------|
| `HTB{th3_0ld_s3qu3nc3_n3v3r_h3s1t4t3s}` | Set membership validation |
