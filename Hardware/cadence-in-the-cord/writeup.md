# Cadence in the Cord — Hardware

**Category:** Hardware  
**Challenge:** Cadence in the Cord  
**CTF:** Cyber Apocalypse 2026  
**Flag:** `HTB{th3_f1rst_m4rk_r1ngs_tru3_b3n34th_th3_w0rds}`

---

## Overview

A logic analyzer capture (`capture.sr`) of a wiretap — Lady Seralyne’s forged “dragon’s true note” transmission. The surface-level UART text is a red herring; the real message is hidden in the timing gaps between bytes.

**Tools used:** `unzip`, `python3` (Saleae `.sr` extraction, UART decode, gap analysis)

---

## Step-by-Step Solution

### 1. Extract the Saleae Capture

`.sr` files are ZIP archives (sigrok format). Extracting revealed 782 binary data files (`logic-1-1` through `logic-1-782`) and a `metadata` file:

```
[sigrok version=0.5.2]
[device 1]
capturefile=logic-1
total probes=8
samplerate=2 MHz
total analog=0
probe2=D1
unitsize=1
```

- 8 probes at **2 MHz** sample rate
- Each sample is 1 byte (bit 0 = probe 0, bit 1 = probe 1, etc.)
- Only **Probe 1** had signal activity (3,562 transitions, 95.8% high = idle-high UART)
- Probes 0 was ground; probes 2–7 were disconnected (stuck high)

### 2. Identify the Protocol

Edge-interval analysis showed the dominant interval was **208 samples** (104 µs) — exactly matching **9600 baud** (2,000,000 ÷ 9600 ≈ 208.33):

```
Sample rate bits:
  9600 baud:  208 samples/bit
```

Pulse-width analysis confirmed two classes of high/low pulses centred on 208-sample increments.

### 3. Decode the Surface Message (UART)

Decoding Probe 1 as async serial (9600 baud, 8N1) yielded:

```
To the buyer who paid in secrets: what follows is the pleasant tone, the goods I sell
in daylight and never miss. Lord Varo’s debt, due at the second thaw, yours for a
marriage you already own. The Harlow inheritance, contested by a cousin whose witnesses
I arranged. Take them and thank me. But what is written is worth nothing. The dragon’s
true note does not live in the words; it lives in the rests between them. A long rest
raises the mark to one, a short rest lets it fall to nothing; count eight rests to every
letter before the note will speak. Read the silence, not the song, and pay.
```

The text itself reveals the encoding method.

### 4. Extract Inter-Byte Gaps

For each successive UART byte, compute:

```
gap = start_of_byte_N+1 - start_of_byte_N - byte_transmission_time
```

Where `byte_transmission_time = 10 bits × 208 samples = 2,080 samples`.

Only **two distinct gap values** existed:

| Gap (samples) | Gap (µs)   | Meaning |
|---------------|------------|---------|
| ~4,014        | ~2,007     | Short rest → binary **0** |
| ~20,017       | ~10,009    | Long rest → binary **1** |

### 5. Decode the Hidden Message

Following the surface text’s instructions:
- **Long rest** (≈10 ms) = `1` (“raises the mark to one”)
- **Short rest** (≈2 ms) = `0` (“lets it fall to nothing”)
- **8 rests per letter** (MSB first)
- **“count eight from one who yields the note”** = MSB-first byte construction

```
Raw bits (first 3 chars): 01111001 01101111 01110101 = “you”
```

The full hidden message:

```
you read the silence well HTB{th3_f1rst_m4rk_r1ngs_tru3_b3n34th_th3_w0rds}
```

---

## Flag

```
HTB{th3_f1rst_m4rk_r1ngs_tru3_b3n34th_th3_w0rds}
```

---

## How We Solved It — Reasoning

This was a classic **timing-based side-channel / steganography** challenge in the hardware domain. Key insights:

1. **The title “Cadence in the Cord”** — “cadence” implies rhythm/timing, and “cord” is a wire. The signal was always going to be about timing, not raw data values.

2. **Only two active probes** out of eight told us this wasn’t a complex bus (like SPI or I²C with clock+data). Single-wire = UART or 1-Wire.

3. **The surface text was the decoder ring.** The challenge deliberately gave us the full instructions *in the captured data itself* — a self-referential puzzle. The phrase *“the dragon’s true note does not live in the words; it lives in the rests between them”* was the decisive clue that inter-character gaps carried the payload.

4. **Binary encoding in silence.** The two clean gap clusters (~2 ms and ~10 ms) with zero ambiguity between them made this a straightforward long=1/short=0 mapping, with 8-bit grouping producing readable ASCII directly.

5. **The flag itself confirms the theme:** *“the first mark rings true beneath the words”* — the real signal is hidden in the timing (the “first mark” = the start bit / timing marker) underneath the surface-level text.

---

## Caveats

- The `.sr` file is a standard sigrok format — don’t try to read it raw; extract with `unzip` first.
- The UART decode alignment matters: sample in the *middle* of the start bit, not at the edge.
- The inter-byte gap calculation must subtract exactly 10 bit-times (start + 8 data + stop) to isolate the true gap.
