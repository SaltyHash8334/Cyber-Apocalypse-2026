# Cinderbound — MicroPython Bytecode Reverse Engineering

**Category:** Reverse Engineering  
**Challenge:** Cyber Apocalypse 2026 — Cinderbound  
**File:** `cinderbound.mpy` (221 bytes)

---

## Overview

A `.mpy` (MicroPython compiled bytecode) file containing a single function `judge(syllable)` — a "ward" that listens for a specific "syllable" (password) and returns `True` only when the correct input is provided. The challenge required reverse-engineering the MicroPython bytecode format, decompiling the algorithm, and reversing it character-by-character to recover the input.

---

## How We Solved It — Reasoning

### Phase 1: Recognizing the Artifact

The file was `.mpy` — a MicroPython compiled bytecode format (header magic `0x064d`, version 6). The challenge lore described a "ward" listening for a "syllable" inside a "foreign engine" — immediately pointing to the `judge` function as the validation gate.

### Phase 2: Decompiling the Bytecode

**Tools/approach:** We used the official `mpy-tool.py` from the MicroPython source (`tools/mpy-tool.py`) with `-d` (disassemble) flag to decompile the `.mpy` file.

The `.mpy` structure:
1. **Header** — magic, flags, small-int bits
2. **Qstr table** — 8 entries including `judge`, `syllable`, `len`, `ord`, `list`, `append`
3. **Object table** — a constant tuple of 16 integers: `(57, 129, 154, 31, 199, 192, 73, 243, 43, 176, 255, 173, 54, 203, 67, 15)`
4. **Module-level raw code** — defines the `judge` function and stores it
5. **Child raw code** — the actual `judge` function bytecode

**Decompiled `judge(syllable)` algorithm:**

```python
def judge(syllable):
    temp = (57, 129, 154, 31, 199, 192, 73, 243, 43, 176, 255, 173, 54, 203, 67, 15)
    state = 90
    result = []
    
    for i in range(len(syllable)):
        c = ord(syllable[i])
        new_val = (state ^ c) ^ ((i * 13) & 255)
        state = (state + c) & 255
        result.append(new_val)
    
    return result == list(temp)
```

Key observations:
- `state` initializes to 90 and updates with `(state + c) & 255` after each character
- Each output byte depends on the current `state`, the character, and its position
- The expected output is a 16-element tuple of integers

### Phase 3: Reversing the Algorithm

Since we know the expected output `new_val[i]` for each position `i`, we reverse:

```
c = (new_val[i] ^ ((i * 13) & 255)) ^ state
state = (state + c) & 255
```

**Python solver:**

```python
expected = (57, 129, 154, 31, 199, 192, 73, 243, 43, 176, 255, 173, 54, 203, 67, 15)
state = 90
syllable = []

for i, ev in enumerate(expected):
    c = (ev ^ ((i * 13) & 255)) ^ state
    syllable.append(c)
    state = (state + c) & 255

print(bytes(syllable).decode())  # c1nd3rbound_v0w5
```

### Phase 4: Verification

Running the forward algorithm on the recovered string `c1nd3rbound_v0w5` exactly reproduces the expected tuple — confirming the solution.

---

## Flag

```
HTB{c1nd3rbound_v0w5}
```

---

## Key Insights

1. **MicroPython `.mpy` is a real compiled format** — not obfuscated or encrypted, just compiled bytecode for the MicroPython VM. Standard CPython decompilation tools won't help — you need `mpy-tool.py`.

2. **Static qstr references** — the `.mpy` file references built-in functions (`len`, `ord`, `list`, `append`) via indices into MicroPython's static qstr table rather than storing the names inline.

3. **The constant tuple is the golden state** — the 16 integers in the object table are the expected computed values. Reversing a linear-congruential-style hash function from known outputs is straightforward arithmetic.

4. **Algorithm structure** — The function uses a rolling hash where each step mixes the current state with the character and position, making it simple to reverse character-by-character.

---

## Files

| File | Description |
|------|-------------|
| `cinderbound.mpy` | Challenge file — MicroPython bytecode |
| `solve.py` | Solver script — reverses the algorithm |
