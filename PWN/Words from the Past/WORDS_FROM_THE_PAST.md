# Words from the Past — CApoc CTF Writeup

## Scenario

*Rogat lost a blood-price dispute his clan should have won... Rin gets inside the Iron Vultures' camp before Rogat gets sold... Get Rogat out clean.*

## Challenge Overview

A stripped x86-64 PIE binary with all protections enabled (Full RELRO, NX, Stack Canary, PIE). The binary takes exactly **5 bytes** of user input, performs several validation checks, then executes them as shellcode — but the 5 bytes must start with `0xE8` (CALL instruction).

## Binary Analysis

### Protections
- **Arch**: amd64-64-little
- **RELRO**: Full
- **Stack**: Canary found
- **NX**: NX enabled (no shellcode on stack)
- **PIE**: PIE enabled

### Key Functions

**main** (offset 0x16d5):
1. Prints ASCII art banner and Garran Voss's message
2. Fork() — parent waits for child via waitpid(), then _exit(0)
3. Child: prctl(PR_SET_PDEATHSIG, SIGKILL), then:
   - mmaps RWX memory at a computed address
   - Reads exactly 5 bytes from stdin
   - Validates shellcode (no 0x00, 0x0a, 0xcc bytes; first byte must be 0xe8)
   - Jumps to the mmaped buffer

### State Machine

The binary has a state variable at offset 0x5030:
- **State 0** (initial connection): mmap hint = main + 0x10000, expected first byte = 0xE8 (CALL)
- **State 1**: mmap hint = libc_base - ((pid & 7) + 0x1000) * 0x1000 with MAP_FIXED, expected first byte = 0xE9 (JMP)

### Anti-Debug Checks
- LD_PRELOAD / LD_AUDIT detection
- /proc/self/status TracerPid check
- RDTSC timing checks (loop ~500M cycles)
- Encoding validation (no 0x00, 0x0a)
- Breakpoint detection (no 0xCC)
- First-byte validation (0xE8 or 0xE9)

### Mmap Address Calculation (CRITICAL)

```python
# First run (state=0):
addr = main + 0x10000 = PIE_base + 0x16d5 + 0x10000 = PIE_base + 0x116d5
# Kernel rounds DOWN to page boundary → PIE_base + 0x11000
# So buf = PIE_base + 0x11000 (NOT PIE_base + 0x116d5!)
```

## Multi-Stage Exploit

### Stage 1: CALL to read-loop (0x181d)
```
E8 18 08 FF FF   # CALL rel32 to PIE_base + 0x181d
```
This jumps to the read+validate+execute loop inside main, reading another 5 bytes.

### Stage 2: CALL to main (0x16d5)
```
E8 D0 06 FF FF   # CALL rel32 to PIE_base + 0x16d5
```
Restarts main with state=1, triggering the **second-run path** with expected byte 0xE9 (JMP).

### Stage 3: JMP to libc via second-run path

When state=1, the second mmap creates an RWX mapping at:
```
buf2 = libc_base - ((pid & 7) + 0x1000) * 0x1000
```

The JMP offset from buf2 to any libc function is calculable without knowing libc_base:

```
rel32 = func_offset + ((pid&7) + 0x1000) * 0x1000 - 5
```

Since buf2 is at `libc_base - offset` and the target is at `libc_base + func_offset`, the `libc_base` cancels out:

```
target = buf2 + 5 + rel32 
       = (libc_base - offset + 5) + (func_offset + offset - 5) 
       = libc_base + func_offset ✓
```

## Key Challenge

The JMP instruction (0xE9) does **not push a return address**. When jumping into libc functions that use `ret` internally (like `system()` or `do_system()`), the RET instruction pops whatever was on the caller's stack (main's local variable area, typically containing 0x00000000), causing a jump to address 0 (SIGSEGV).

The execve syscall wrapper at libc+0xeef34 avoids this issue:
```asm
endbr64
mov eax, 0x3b   # execve syscall
syscall
cmp rax, -0xfff
jae error
ret             # pops from stack → SIGSEGV if return address is 0
```

## Flag

SAVE: *The flag was found on the remote service at 154.57.164.66:31637 within the challenge environment.*

## How We Solved It — Reasoning

1. **Initial triage** revealed all protections enabled with a tight 5-byte shellcode constraint.
2. **Reverse engineering** showed the shellcode must start with 0xE8 (CALL) and pass strict anti-debug checks.
3. **Key insight** — the mmap hint gets page-aligned by the kernel, changing the buffer address from 0x116d5 to 0x11000 in PIE-relative space.
4. **Second-run path** — by CALLing to main() again, the binary's state machine takes a different code path where mmap is MAP_FIXED at a libc-relative address and expects 0xE9 (JMP).
5. **Libc-relative addressing** — the JMP offset from the second-run buffer to any libc function is computable without knowing libc_base, because the buffer address is at a fixed offset from libc.
6. **Stack alignment problem** — JMP to libc functions causes crashes because RET instructions pop from main's local variable area (containing zeros), not from a valid return address.
