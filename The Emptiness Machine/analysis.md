# Summary: The Emptiness Machine CTF Pwn Challenge

## Challenge Type
PWN - FILE struct corruption / FSOP exploitation
Binary: the_emptiness_machine (x86-64, PIE, NX, Full RELRO, no canary)

## Binary Behavior
Simple main():
1. setbuf(stdin, NULL), setbuf(stdout, NULL)
2. puts(banner_ascii_art)
3. printf("[Subdued Voice]: Let you cut me open... Just to watch me bleed..🩸🩸\n\nRin's interaction: ")
4. scanf("%40s", stdout) — writes 40 chars to stdout's FILE struct (libc+0x2045c0)
5. printf("[Subdued Voice]: Gave up who I am for who you wanted me to be..\n\nRin's interaction: ")
6. scanf("%224s", stderr) — writes 224 chars to stderr's FILE struct (libc+0x2044e0)
7. main returns → _IO_cleanup flushes all FILE streams

## Critical Libc Details
- glibc 2.39 (Ubuntu) with **vtable validation REMOVED** — no _IO_validate_vtable
- _IO_2_1_stderr_ = libc + 0x2044e0
- _IO_2_1_stdout_ = libc + 0x2045c0  
- Distance between stderr and stdout = 0xe0 = 224 bytes (exactly the second input size!)
- Second input's null terminator at byte 224 = overwrites first byte of stdout._flags

## Exploit Strategy (two phases)

### Phase 1: Leak (via stdout partial overwrite)
- Send 32 bytes as first input (flags = 0xfbad2086, no _IO_NO_WRITES, no _IO_NO_WRITES)
- Null at byte 32 zeroes LSB of _IO_write_base
- _IO_write_ptr becomes > _IO_write_base, triggering flush
- Flushed data contains libc pointers from stdout FILE struct

### Phase 2: Shell (via stderr fake FILE struct + vtable redirect)
- No vtable check → set vtable to any address
- Vtable at offset 0xD8 in FILE struct. __overflow at vtable+0x18
- Set vtable = stderr + 0x20 → __overflow reads from stderr + 0x38 = _IO_buf_base
- Set _IO_buf_base = system() address → __overflow = system
- Start stderr with "/bin/sh\x00" → system(stderr) = system("/bin/sh")

### Key First Input Flags
0xfbad2086 = _IO_MAGIC | _IO_IS_FILEBUF | _IO_LINKED | _IO_UNBUFFERED | _IO_NO_READS

CRITICAL: Must NOT have _IO_NO_WRITES (0x08) or _IO_CURRENTLY_PUTTING (0x0800)
- _IO_NO_WRITES blocks all output
- _IO_CURRENTLY_PUTTING redirects characters to memory instead of socket

## Libc Offsets
- system = 0x58750
- /bin/sh = 0x1cb42f
- execve = 0xeef30
