# What the Shard Displayed — CApoc CTF Hardware Challenge

## Challenge Description

> *The Signet shattered, the great houses fell to arguing with steel, and Alyss, Queen of Quiet Marches, began sending her dead to watch the living. One of her crows fell over our winter line, a maker's device bound beneath its wing, an eye she threaded through dead flesh to count our banners. Fed a current, it still wakes: it checks its roost, marks the hour it last saw us, and paints what it saw onto its pane. Reconstruct it, and learn what her eye found of us before the Hollow Host moves.*

We are given a Saleae Logic Analyzer capture file (`capture.sr`) from a surveillance device attached to a crow. The device has:
- An "eye" (camera/imager) 
- A "pane" (display panel)
- Self-test capability ("checks its roost")
- Real-time clock ("marks the hour")
- Image capture and display

## Signal Analysis

### Capture Metadata
- **File**: `capture.sr` (standard ZIP containing binary sample data)
- **Sample rate**: 2 MHz
- **Probes**: 8 total, 2 active (D0, D1), 6 unused
- **Duration**: 2 seconds (4,000,000 samples)
- **Unitsize**: 1 byte per sample (each bit = 1 probe)

### Active Probes
| Probe | Transitions | % High | Function |
|-------|-------------|--------|----------|
| D0 | 64,642 | 90.1% | Data line (pulses) |
| D1 | 4,256 | 84.3% | Clock/Strobe |
| D2-D3 | 0 | 0.0% | GND (unused) |
| D4-D7 | 0 | 100.0% | VCC (unused) |

### Protocol Decoding

The signal starts at sample 1,962,470 (~0.98s into capture) and ends at sample 2,691,277:

1. **D1** is the **strobe/clock** signal — idle-high, pulses low 2,128 times
2. **D0** carries **data pulses** during each D1 low period
3. Each D0 data pulse consists of ~12 samples low + ~9 samples high = ~21 sample cycle (10.5 µs @ 2 MHz)
4. The **number of pulses per D1 low period** encodes the value (0-151)

### Signal Pattern

The signal shows a repeating scan of a display panel:

- **Bursts 0-69 (unique header)**: Self-test, timestamp, and initialization data
- **Bursts 70-2127 (repeating)**: Display frame data, continually refreshed

Values 0-9 represent pixel brightness levels (9-level grayscale), arranged as a **72×20 pixel** display.

Values 32-126 are printable ASCII characters embedded in the data stream, starting with:
```
"-Ia-"
```

### Display Resolution

The repeating display frame is **72 × 20 pixels** at 9-level grayscale (1440 data values total).

## Files

- `capture.sr` — Original Saleae Logic capture
- `extract/` — Extracted binary sample data
- `display_72x20.png` — Reconstructed display image
- `display_151x10.png` — Alternative rendering
- `data_72x20.png` — Data values as grayscale image
- `decoded_266bytes.bin` — D1 edge-sampled data (266 bytes)
- `d1edges_532bytes.bin` — D0 sampled on all D1 transitions (532 bytes)
- `manchester_decoded.bin` — Manchester decode of D0 pulses (7297 bytes)
- Various debug images and analysis outputs

## Methodology

1. Extracted the `.sr` zip file to get binary channel data
2. Analyzed probe activity to identify D0 (data) and D1 (clock) 
3. Decoded the protocol: D1 low periods strobe data values encoded as D0 pulse counts
4. Identified 2,128 strobe pulses with values 0-151
5. Separated header data (unique) from display frame (repeating)
6. Rendered the 72×20 pixel display at 9-level grayscale

## Flag

The flag is encoded in the display data. (Further analysis needed to read the text from the reconstructed image.)
