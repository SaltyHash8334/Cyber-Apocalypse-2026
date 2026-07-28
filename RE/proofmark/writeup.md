# Proofmark — Reverse Engineering

**Category:** Reverse Engineering  
**Challenge:** Cyber Apocalypse 2026 — Proofmark  
**Flag:** `HTB{p3rf3ct_f4c3_tru3_sp1n3}`

---

## Overview

Proofmark is an Android Godot export. The visible game presents a forge and an anvil, but the important validation logic is in the native GDExtension:

```text
lib/arm64-v8a/libproofmark.arm64.so
lib/x86_64/libproofmark.x86_64.so
```

The APK used for this solve was:

```text
SHA-256: e326627607b98fea7f4939496e710430a6da0c4a6ee522b9090f17b78db8546f
```

The ARM64 native library used for verification was:

```text
SHA-256: 05dab75469798c49ccc9def9c2842b4bd614dc700c11458b1b6042f52a607153
```

---

## Scenario

Vaultrune's assay decides which seals are inherited and which were cut from a Signet shard, and every house sends its stamps down to it. Elric Ashspar walks in with a ring he filed himself. If the anvil calls his work inherited, it cannot tell the difference, and neither can anyone else. One strike, no second.

---

## Analysis

### APK triage

```bash
file proofmark.apk
sha256sum proofmark.apk
unzip -Z1 proofmark.apk | grep -E 'assets/scripts|lib/.*/libproofmark|proofmark.gdextension'
```

The APK contains compiled GDScript resources beginning with `GDSCe`, an Android-only `.gdextension`, and native ARM64/x86_64 libraries. The extension declares Godot 4.7 compatibility, while the available host Godot was 4.6.3, so running the APK directly in the Linux editor was not a reliable path. The native ARM64 library was analysed and called from a small ARM64 runner instead.

The library exports `proofmark_library_init`. Relevant native routines identified with `radare2` were:

| Routine | Offset | Role |
|---|---:|---|
| Certificate | `0x1af4` | Derives a 32-bit value from the supplied mark |
| Submit/reseal | `0x1ff4` | Checks the mark, generates output, checks `HTB{`, and returns status |

### Accepted mark

At `0x1ff4`, the verifier requires a 16-byte input and compares two 64-bit loads against four little-endian 32-bit constants:

```text
53 00 00 00 43 00 00 00 37 00 00 00 ce 01 00 00
```

As integers:

```text
[0x53, 0x43, 0x37, 0x1ce]
[83, 67, 55, 462]
```

The certificate routine also checks the weighted relation:

```text
83 + 2*67 + 3*55 = 462
```

Calling the native certificate routine with this mark returns:

```text
certificate = 0xdc457bf0
```

### Reversible state generator

The submit routine uses 32-bit arithmetic and the constants:

```text
ADD  = 0xc2b2ae35
MUL1 = 0x85ebca6b
MUL2 = 0xc2b2ae35
```

The core mixer is:

```c
uint32_t mix_no_final(uint32_t x) {
    x ^= x >> 16;
    x *= 0x85ebca6b;
    x ^= x >> 13;
    x *= 0xc2b2ae35;
    return x;
}
```

Each output byte uses a 28-byte mask embedded at native offset `0x580`:

```text
2a53db7ba35d34f55f59745e0043881ca1136fb7f8d73f79c1b0af1a
```

The generator runs `0x124f80` state transitions. Since both multipliers are odd and right-xorshift operations are reversible, the state transition is a permutation over 32-bit words.

---

## Solution

### State search

The first four desired output bytes are `HTB{`. XORing with the first four mask bytes gives four constrained mixer-output high bytes. The solver enumerates the remaining 24 bits of the first mixer output, checks the next three bytes, reverses the state transition, and finally calls the original native routine for verification.

The complete solver is:

```text
/home/davey/CTF/CApoc/Proofmark/arm_work/solve_seed.c
```

The native runner and patched working library are:

```text
/home/davey/CTF/CApoc/Proofmark/arm_work/runner.c
/home/davey/CTF/CApoc/Proofmark/arm_work/runner
/home/davey/CTF/CApoc/Proofmark/arm_work/libproofmark.arm64.host.so
```

Run it with:

```bash
cd /home/davey/CTF/CApoc/Proofmark/arm_work
./solve_seed
```

Verified output:

```text
candidate=0 state0=0x000df25a seed=0xd1f87045 native_status=2
output_hex=4854427b0363f89979dc5a77e68aa4d396e739d063138994adc5bf3f00

candidate=1 state0=0x00b22130 seed=0x71d3a101 native_status=2
output_hex=4854427b703372663363745f663463335f747275335f7370316e337d00
output=HTB{p3rf3ct_f4c3_tru3_sp1n3}
format_ok=PASS
verification=PASS
candidates_tested=2
```

The first candidate passes the native `HTB{` prefix test but is not a printable, brace-terminated flag. The second candidate is the complete 28-byte result.

---

## How We Solved It — Reasoning

### 1. Follow the verifier, not the scenery

The scenario suggests a 3D gameplay sequence, but the APK exposes a native `Proofmark` class and the compiled GDScript strings reference `certificate`, `submit`, `mark`, `hallmark`, and `strike`. This made the native verifier the authoritative target.

### 2. Do not treat the certificate as the final seed

The accepted mark produces `0xdc457bf0`, but passing that directly to the reseal routine returns a non-flag output. This rejects the hypothesis that the certificate is the final generator state.

The routine instead performs a long reversible state transition. The first four output bytes narrow the possible internal states, after which the native function is used as the final oracle.

### 3. Prefix checks are not enough

Two candidates pass the native prefix check. Only one produces a printable 28-byte string ending in `}`. This is why the solve validates the complete output and native return status rather than stopping at `HTB{`.

### 4. The key insight

The “perfect face, wrong spine” idea is reflected directly in the implementation: the first visible bytes can look correct while the remaining state path is wrong. The state mixer is reversible, so recovering the hidden spine is feasible without brute-forcing all 32-bit values.

---

## Key Takeaways

- Godot APK logic may live in a native GDExtension rather than readable source.
- Android-only extensions are not automatically loadable by a Linux Godot editor.
- Odd 32-bit multipliers and XOR-right-shifts form reversible mixers.
- The accepted mark is a binary little-endian structure, not text.
- A prefix-only match is insufficient; verify the complete flag against the original native routine.

---

## Final Flag

```text
HTB{p3rf3ct_f4c3_tru3_sp1n3}
```

---

## Appendix: exact proof and winning state

```text
proof=530000004300000037000000ce010000
certificate=0xdc457bf0
winning_seed=0x71d3a101
native_status=2
output_hex=4854427b703372663363745f663463335f747275335f7370316e337d00
format_ok=PASS
verification=PASS
```

The original APK was not modified; only a working copy of the ARM64 library had its loader metadata adapted so the native routines could be called on the host.

---

## End

Flag verified: `HTB{p3rf3ct_f4c3_tru3_sp1n3}`
