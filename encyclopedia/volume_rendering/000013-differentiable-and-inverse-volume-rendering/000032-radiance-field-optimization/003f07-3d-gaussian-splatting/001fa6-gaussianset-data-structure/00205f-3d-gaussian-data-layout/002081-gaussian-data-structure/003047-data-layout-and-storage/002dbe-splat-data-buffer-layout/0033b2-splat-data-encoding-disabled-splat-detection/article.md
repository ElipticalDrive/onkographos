# Splat Data Encoding & Disabled Splat Detection

Also called: Disabled Splat Detection, Splat Data Encoding

## Overview
This method defines the binary encoding scheme for 3D Gaussian splats and provides the logic to detect "disabled" splats during spatial queries and sorting [S1, unofficial]. It solves the problem of efficiently filtering out inactive or merged Gaussians without allocating separate boolean flags or requiring complex metadata lookups [S1, unofficial]. It is distinct from standard active/inactive flags by embedding the status directly into the data payload's bit pattern, allowing the renderer to treat disabled splats as invalid data (`NaN`) during coordinate extraction [S1, unofficial]. This builds on [Splat Data Buffer Layout](5568-splat-data-buffer-layout.md) [S1, unofficial].

## How it works
**Inputs:** A stream of splat data stored in two `Uint32Array` buffers [S1, unofficial].
**Outputs:** A boolean indicating if a splat is disabled; modified coordinates (`NaN`) if disabled [S1, unofficial].

**Data Structure:**
Each splat occupies 32 bytes (8 words), split across two arrays [S1, unofficial]:
- `splatA`: Words 0–3 (16 bytes).
- `splatB`: Words 4–7 (16 bytes).
- For splat index $i$, the starting index in `splatB` is $i_4 = i \times 4$ [S1, unofficial].

**Disabled Splat Signature:**
A splat is considered disabled if the following bit pattern is detected in `splatB` [S1, unofficial]:
1. The upper 16 bits of `splatB[i_4 + 1]` equal `0xfc00`.
2. The entire value of `splatB[i_4 + 2]` equals `0xfc00fc00`.

**Algorithm:**
1. For each splat index $i$:
   a. Calculate $i_4 = i \times 4$.
   b. Read `word1 = splatB[i_4 + 1]`.
   c. Read `word2 = splatB[i_4 + 2]`.
   d. Check if `(word1 >> 16) == 0xfc00` AND `word2 == 0xfc00fc00` [S1, unofficial].
   e. If true, the splat is disabled.

**Coordinate Extraction with Detection:**
When extracting center coordinates $(x, y, z)$ for sorting or spatial queries:
1. If the splat is disabled, set $x = \text{NaN}$, $y = \text{NaN}$, $z = \text{NaN}$ [S1, unofficial].
2. If the splat is enabled, extract coordinates normally from the encoding [G].
3. Pass these coordinates to the sorting/spatial query function. Standard NaN-aware sorting or spatial partitioning will exclude these entries [G].

**Pitfalls:**
- **False Positives:** If valid Gaussian data accidentally matches the `0xfc00` pattern, the splat will be incorrectly disabled. This pattern must be reserved exclusively for disabled states [S1, unofficial].
- **Processing Artifacts:** If disabled splats are not detected and processed as valid, they may cause rendering artifacts or incorrect sorting behavior because their coordinates are undefined or meaningless [S1, unofficial].
- **Boundary Checks:** Ensure array bounds are respected when accessing `splatB[i_4 + 1]` and `splatB[i_4 + 2]` [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def is_splat_disabled(splatB, i):
    i4 = i * 4
    word1 = splatB[i4 + 1]
    word2 = splatB[i4 + 2]
    return (word1 >> 16) == 0xfc00 and word2 == 0xfc00fc00

def extract_center_with_disabled_check(splatA, splatB, i):
    if is_splat_disabled(splatB, i):
        return (float('nan'), float('nan'), float('nan'))
    # Extract x, y, z from splatA and splatB according to the standard encoding [G]
    x = decode_float(splatA[i * 4 + 0])
    y = decode_float(splatA[i * 4 + 1])
    z = decode_float(splatA[i * 4 + 2])
    return (x, y, z)

def decode_float(word):
    import struct
    return struct.unpack('f', struct.pack('I', word))[0]
```

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(1)$ per splat for detection [S1, unofficial].
- **Memory Overhead:** Zero additional memory; the status is encoded within the existing 32-byte splat structure [S1, unofficial].
- **Performance:** Extremely fast detection via bitwise operations [S1, unofficial].
- **Strengths:** Efficient, compact, and integrates seamlessly with NaN-based filtering in spatial data structures [S1, unofficial].
- **Limitations:** Requires strict adherence to the reserved bit pattern; any corruption of the data could lead to false positives or negatives [S1, unofficial].

## Background
This method is part of the `Gaussian-Splat-Lite` renderer, an unofficial implementation of 3D Gaussian Splatting optimized for WebGPU [S1, unofficial]. It was developed to handle large-scale Gaussian datasets efficiently in a browser environment, where memory and performance constraints are significant [S1, unofficial]. The specific encoding and detection mechanism were designed to minimize overhead while ensuring robust handling of merged or inactive Gaussians [S1, unofficial].

## Sources
[S1] WilliamLiu-1997/Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
