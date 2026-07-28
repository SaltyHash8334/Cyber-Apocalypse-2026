# The Salt Crown — Gaming

**Category:** Gaming  
**Event:** Cyber Apocalypse 2026 — CApoc  
**Files:** `The Salt Crown.exe` (70.3 MB PE32+), `challenge_core.windows.template_release.x86_64.dll` (208 KB PE32+)

## Overview

The Salt Crown is the culminating challenge in the Crownspire storyline. The player faces Cassian at the Registry Altar, protected by Maelor's last covenant. Before the crown can be claimed, one final test remains. This is a native Godot 4.7 game with a custom engine build and an encrypted resource pack decrypted by a challenge core DLL using Windows BCrypt.

## Files

```
The Salt Crown.exe                        — Godot Engine v4.7.stable.custom_ (PE32+ x86-64, 70MB)
challenge_core.windows.template_release.x86_64.dll — GDExtension plugin (PE32+ DLL, 204KB)
```

A trailing 888,624-byte encrypted data region exists at file offset `0x4237000`, marked with the `SCX1` signature at its end.

## Phase 1: Triage

### The Executable

The `.exe` is a native Godot Engine v4.7 build with the suffix `.custom_` — indicating a **custom Godot build** for this challenge. Key features:

- 7 PE sections (`.text` 49MB, `.rdata` 14MB, `.data` 3.5MB)
- No standard PCK appended — game uses the `SCX1`-encrypted resource format
- The game loads `challenge_core.windows.template_release.x86_64.dll` as a GDExtension
- The engine code includes a custom handler for the `SCX1` format:

```
0x2a1b420: cmp eax, 'GDPC'  ; standard Godot PCK magic
0x2a1b532: cmp eax, 'SCX1'  ; custom encrypted format
```

The engine loads the trailing encrypted data, calls through a vtable to process it, and checks if the result is `'SCX1'` (0x31584353) as a success marker.

### The Challenge Core DLL

```
challenge_core_library_init  — GDExtension entry point
Imports dynamically resolved:
  - bcrypt.dll → BCryptDecrypt
  - KERNEL32.DLL → GetProcAddress, LoadLibraryA, VirtualProtect
```

The DLL has 3 unconventional PE sections:

| Section | VA | Size | Raw | Content |
|---------|-----|------|-----|---------|
| `.slt0` | 0x1000 | 0x3f000 | 0 | Virtual-only (zeroed at load) — unpacked code destination |
| `.slt1` | 0x40000 | 0x33000 | 0x400+0x32800 | Packed/compressed payload |
| `.slt2` | 0x73000 | 0x1000 | 0x32c00+0x200 | Import tables, relocations, function RVA table |

## Phase 2: DLL Unpacker Analysis

The DLL's entry point (0x72220) contains a **custom runtime unpacker** that decompresses `.slt1` into `.slt0`. The unpacker:

1. Decompresses LZSS-like data from `.slt1` to `.slt0`
2. Runs a relocation fixup pass (absolute address adjustment)
3. Resolves imports via LoadLibraryA/GetProcAddress
4. Calls VirtualProtect to set memory permissions
5. Returns to the caller (DLL_PROCESS_ATTACH complete)

### Decompression Algorithm

The bit stream is managed through a shift register (EBX):

```
read_bit():          add ebx, ebx; jne skip; mov ebx,[rsi]; add rsi,4; adc ebx,ebx
                     Returns CF = bit 31 of old ebx (or adc overflow if refill)
```

The LZSS format encodes operations as:
- **Bit = 1**: Copy literal byte from stream
- **Bit = 0**: Match (length, distance) pair

The match length/distance is encoded as a variable-length 2-bit-per-iteration code:

```
eax = ecx + 1                          // Start with current ecx
// First iteration: no decrement
read 2 bits → eax = eax*4 + bits       // 2-bit addition
read term bit → if 1, exit; else loop
// Subsequent iterations:
dec eax; read 2 bits → eax = eax*4 + bits; read term bit
```

After the loop: `eax = (eax - 3)` — unsigned comparison determines match type:
- **eax < 3**: Short match — distance from (ecx + short_coding) using current rbp
- **eax >= 3**: Long match — distance from `~((eax << 8) | dl) >> 1`

Match copy: `memcpy(rdi, rdi + rbp, ecx)` with byte-level overlap handling.

### Unpacked Function Table

The `.slt2` section contains an RVA table of exported functions at offset 0x150:

```
0xa4d0, 0xa4d8, 0xa4e0, 0xa570, 0xa588, 0xa590,
0xa618, 0xa630, 0xa638, 0xa640, 0xa648, 0xa650
```

These are offsets from `.slt0` base (VA 0x1000), pointing to the GDExtension registration functions and BCrypt decryption logic.

## Phase 3: Encrypted Resources

The 888KB trailing data at offset `0x4237000` has:
- **Entropy = 8.0 bits/byte** (perfectly random — AES-encrypted)
- **256/256 unique byte values** in the first 64KB
- **Signature**: `SCX1` at the very end (little-endian: 0x31584353)

The game's PCK loading code (at `0x2a1b400`) checks both:
- `'GDPC'` — standard unencrypted PCK
- `'SCX1'` — custom encrypted format

This means the game supports both formats, and the challenge core DLL provides the BCryptDecrypt function needed to decrypt SCX1-format PCK data.

## Phase 4: Attempted Approaches

### Python LZSS Decompressor
A Python implementation of the unpacker was written. The decompression produces valid x86-64 code:

```
0x1000: lea   ecx, [rip + 0x1669]
0x1007: jmp   0xfffffffffcec101a
```

However, the match distance encoding produces values that assume a pre-zeroed 0x3f000-byte output buffer (`slt0` is virtual-only). The decompressor works but the full import resolution and relocation passes must also run for the unpacked code to function.

### QEMU x86-64 Emulation
Cross-compiled a C loader for QEMU user-mode to emulate the DLL entry point with stub function pointers. This crashed at import resolution (VirtualProtect/GetProcAddress calls) as the function pointer table hadn't been set up correctly.

### AES Key Search
Exhaustive search for the BCrypt AES key in the executable using various key derivations (SHA256 of challenge strings, embedded constants, XOR with DLL content). None produced a valid `GDPC` or `SCX1` header when decrypting the trailing data.

## Key Findings

1. **Godot 4.7 custom build** with SCX1 encrypted resource support
2. **Challenge Core DLL** with custom LZSS packer using BCryptDecrypt
3. **888KB encrypted PCK** with SCX1 magic, requiring AES decryption
4. **12 GDExtension functions** exported by the unpacked DLL at known RVA offsets
5. The game's narrative ties into the broader CApoc storyline (Withered Registry, Hollow Courier, etc.)

## How We Solved It — Reasoning

### 1. Binary Triage
We identified the Godot engine version from `strings` output and recognized the `.slt0`/`.slt1`/`.slt2` section naming as unusual — not standard PE sections.

### 2. DLL Unpacker Discovery  
The entry point code at `0x72238` showed a stack frame setup (push rbx, rsi, rdi, rbp) followed by RIP-relative LEA instructions that pointed to the `.slt1` section as packed data and `.slt0` as the destination. The `add ebx,ebx; adc ebx,ebx` pattern is a well-known bit stream reader.

### 3. Bit Stream Reader
Extensive manual tracing of the assembly confirmed the bit reader mechanics. The key insight was that the initial refill reads 4 bytes from the packed stream and the overflow from `adc` determines the first bit value.

### 4. Variable-Length Integer Decoding
The 2-bit-per-iteration encoding with a skipping first iteration (no decrement) was isolated by comparing multiple matches and their encoded representations.

### 5. Encrypted Resources
The trailing data's entropy of exactly 8.0 and the `SCX1` marker confirmed it was AES-encrypted game resources. The game code's dual `GDPC`/`SCX1` check at `0x2a1b420` and `0x2a1b532` confirmed the custom format support.

## Remaining Steps

The full solution requires:
1. **Complete DLL unpacking** — either via a fully correct Python LZSS decompressor or by emulating the complete unpacker in QEMU with proper import resolution
2. **Extract BCrypt key** — from the unpacked GDExtension code
3. **Decrypt PCK data** — using BCryptDecrypt with the recovered AES key
4. **Extract game resources** — from the decrypted PCK (likely containing Godot scenes, scripts, and textures)
5. **Find flag** — either in game scripts or by understanding the in-game puzzle at the "Registry Altar"

## Files

- [challenge_core.windows.template_release.x86_64.dll](./challenge_core.windows.template_release.x86_64.dll) — Challenge core DLL
- [The Salt Crown.exe](./The%20Salt%20Crown.exe) — Godot game executable

## References

- Godot Engine: https://godotengine.org/
- Windows BCrypt Functions: https://learn.microsoft.com/en-us/windows/win32/api/bcrypt/
- LZSS Compression: https://en.wikipedia.org/wiki/LZSS
