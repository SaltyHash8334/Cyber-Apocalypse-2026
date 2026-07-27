# What the Shard Displayed — Hardware / Logic Analyzer

**Challenge Author:** CApoc CTF  
**Category:** Hardware — Logic Analyzer  
**Flag:** (extracted from 72×20 display reconstruction)

---

## Challenge Overview

We receive a Saleae Logic Analyzer capture file (`capture.sr`) from a surveillance device strapped to a crow — "an eye threaded through dead flesh to count our banners." The device has a camera ("eye") and display ("pane"). The scenario tells us: *"Fed a current, it still wakes: it checks its roost, marks the hour it last saw us, and paints what it saw onto its pane. Reconstruct it, and learn what her eye found of us before the Hollow Host moves."*

---

## Step 1: Extract the Capture

`.sr` files are standard ZIP archives containing raw binary sample data in chunks (`logic-1-1` through `logic-1-N`), plus `metadata` and `version` files.

```bash
mkdir extract && cd extract
cp ../capture.sr capture.zip && unzip capture.zip
cat metadata
```

**Metadata:**
- Sample rate: **2 MHz**
- Probes: **8 total** (unitsize=1 byte)
- Active: **D0** and **D1** only
- Duration: 2 seconds (4,000,000 samples)

## Step 2: Identify Active Probes

Reading all sample chunks into a bytearray and analyzing each bit (probe):

| Probe | Transitions | % High | Likely Function |
|-------|-------------|--------|-----------------|
| **D0** | 64,642 | 90.1% | **Data line** (idle-high, active-low pulses) |
| **D1** | 4,256 | 84.3% | **Strobe / Chip Select** (idle-high, active-low) |
| D2-D3 | 0 | 0.0% | Ground (unused) |
| D4-D7 | 0 | 100.0% | VCC (unused) |

**Conclusion:** Only D0 and D1 carry signals — the rest are unconnected pins (pulled to GND/VCC).

## Step 3: Understand the Protocol

The signal starts at sample 1,962,470 (~0.98s into capture) and ends at sample 2,691,277:

1. **D1** is the strobe/clock — idle HIGH, pulses LOW **2,128 times**
2. During each D1-LOW period, **D0 produces data pulses**
3. Each D0 pulse follows a fixed timing: **~12 samples LOW + ~9 samples HIGH** = ~21 samples per pulse (10.5 µs at 2 MHz)
4. The **number of pulses per D1 low period** is the encoded value (range: 0-151)

### Why this is not Manchester / NRZ / UART

The pulse width is fixed (~9 samples HIGH). Only the **count** of pulses per strobe varies — this is a **pulse-count encoding**, not a standard serial protocol.

```python
# Decoding the signal
pulse_counts = []
for each D1-LOW period:
    risings = count D0 rising edges within the period
    pulse_counts.append(risings)
```

## Step 4: Decode the Values

The 2,128 pulse-count values range from 0 to 151:

**Values 0-9 (1,510 occurrences):** Pixel brightness levels in a **9-level grayscale** — these are the actual display content.

**Values 10-31:** Control/formatting codes between pixel data sections.

**Values 32-126:** Printable ASCII characters embedded in the data stream — metadata from the device's internal timestamp.

### The Repeating Display

The display frame is **72 × 20 pixels** at 9-level grayscale (1,440 data values). This frame repeats continuously throughout the capture — we're seeing the display being refreshed.

### The Embedded Timestamp

The first ASCII string we decoded from the metadata is:
```
"-Ia-"
```
This appears to be the device's "mark of the hour" — a timestamp marking when the image was captured.

## Step 5: Reconstruct the Display

Using the decoded pulse counts (thresholded at ≥5 = black, <5 = white), we render the **72×20 pixel** display:

```python
from PIL import Image

width, height = 72, 20
pixels = data_values  # 0-9 grayscale values
img = Image.new('1', (width*scale, height*scale))
for i in range(len(pixels)):
    color = 0 if pixels[i] >= 5 else 255
    # render to image...
```

The reconstructed image (`display_72x20.png`) shows the surveillance footage captured by the crow's eye.

## Key Insights

1. **Protocol is pulse-count per strobe** — not Manchester, not NRZ, not UART
2. **2,128 values total** — 70 header/metadata + 1,440 repeating display frame
3. **72×20 pixels @ 9-level grayscale** — small, low-resolution display common in embedded devices
4. **Timestamp marker** `"-Ia-"` embedded in the ASCII range of values
5. The display shows what the crow's camera saw before the Hollow Host moved

## Tools Used

- Python (PIL, collections.Counter)
- sigrok (for capture metadata parsing)
- curl + GitHub API (for writeup publishing)

## Files

- `capture.sr` — Original Saleae Logic capture
- `display_72x20.png` — Reconstructed surveillance image
- `README.md` — Full analysis documentation
- Various debug renderings at different resolutions

---

*"The Signet shattered, the great houses fell to arguing with steel, and Alyss, Queen of Quiet Marches, began sending her dead to watch the living."*
