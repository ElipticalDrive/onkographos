# Deterministic Integer Hashing

Also called: hashCell, hashToFloat, lowbias32

## Overview
Deterministic integer hashing provides a platform-independent method for generating pseudo-random integers from integer coordinates, ensuring identical bit-exact results between CPU (TypeScript) and GPU (GLSL) environments [S2, unofficial]. It solves the reproducibility problem in procedural content generation where floating-point sine-based hashing drifts due to library implementation differences [S2, unofficial]. Unlike floating-point sine hashing, this method uses only 32-bit bitwise operations and multiplication, guaranteeing consistent wrapping behavior for negative coordinates [S2, unofficial].

## How it works
**Inputs and Outputs:** `uhash` takes a 32-bit unsigned integer `x` and returns a 32-bit unsigned integer `h` [S2, unofficial]. `hashCell` takes three 32-bit signed integers `cx, cy, cz` and returns a 32-bit unsigned integer `h` [S2, unofficial]. `hashToFloat` takes a 32-bit unsigned integer `h` and returns a 32-bit float `f` in $[0, 1)$ [S2, unofficial].

**Data Structures:** All intermediate values are 32-bit unsigned integers (`uint32`) [S2, unofficial]. Multiplication uses 32-bit wrapping semantics (modulo $2^{32}$) [S2, unofficial].

**Parameters:** `uhash` multipliers are $M_{u1} = \text{0x7feb352d}$ and $M_{u2} = \text{0x846ca68b}$ [S2, unofficial]. `hashCell` multipliers are $M_{cx} = \text{0x9e3779b9}$, $M_{cy} = \text{0x85ebca6b}$, and $M_{cz} = \text{0xc2b2ae35}$ [S2, unofficial]. `compute_hash` constants are $M_0 = 2860486313\text{U}$ and $M_1 = 3267000013\text{U}$ [S1, unofficial].

**Steps:**

1. **Base Mixer (`uhash`):**
   Mix the input integer `x` using a sequence of XOR shifts and multiplications:
   $$
   \begin{aligned}
   x &\leftarrow x \oplus (x \gg 16) \\
   x &\leftarrow x \times M_{u1} \\
   x &\leftarrow x \oplus (x \gg 15) \\
   x &\leftarrow x \times M_{u2} \\
   x &\leftarrow x \oplus (x \gg 16) \\
   \text{return } x \gg 0
   \end{aligned}
   $$
   The final `>>> 0` or `x >> 0` ensures the result is treated as an unsigned 32-bit integer, masking out any sign extension if the language treats shifts differently on signed types [S2, unofficial].

2. **3D Coordinate Hashing (`hashCell`):**
   Combine three integer coordinates into a single hash:
   $$
   \begin{aligned}
   h &\leftarrow \text{uhash}(cx \times M_{cx}) \\
   h &\leftarrow \text{uhash}(h \oplus (cy \times M_{cy})) \\
   h &\leftarrow \text{uhash}(h \oplus (cz \times M_{cz})) \\
   \text{return } h \gg 0
   \end{aligned}
   $$
   Negative coordinates are handled naturally because the multiplication and XOR operations on 32-bit integers wrap identically to GLSL's `uint(int)` cast [S2, unofficial].

3. **Float Conversion (`hashToFloat`):**
   Map the integer hash to a float in $[0, 1)$:
   $$
   f = \frac{h \gg 8}{16777216.0}
   $$
   Shifting right by 8 bits discards the lower 8 bits, leaving the top 24 bits [S2, unofficial]. This ensures the value fits within the 24-bit mantissa of a 32-bit IEEE 754 float without precision loss during conversion [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def uhash(x: int) -> int:
    x = x & 0xFFFFFFFF
    x ^= (x >> 16)
    x = (x * 0x7feb352d) & 0xFFFFFFFF
    x ^= (x >> 15)
    x = (x * 0x846ca68b) & 0xFFFFFFFF
    x ^= (x >> 16)
    return x & 0xFFFFFFFF

def hashCell(cx: int, cy: int, cz: int) -> int:
    M_CX = 0x9e3779b9
    M_CY = 0x85ebca6b
    M_CZ = 0xc2b2ae35
    h = uhash(cx * M_CX)
    h = uhash(h ^ (cy * M_CY))
    h = uhash(h ^ (cz * M_CZ))
    return h

def hashToFloat(h: int) -> float:
    return (h >> 8) / 16777216.0

def compute_hash_byte_slice(data: list) -> int:
    M0 = 2860486313
    M1 = 3267000013
    h = M0
    for b in data:
        h = ((h ^ b) * M1) & 0xFFFFFFFF
    return h
```

**Edge Cases and Pitfalls:**
- **Negative Coordinates:** Integer hashing handles negative inputs correctly because 32-bit two's complement arithmetic wraps consistently across platforms [S2, unofficial].
- **Precision Loss:** Using the full 32-bit integer directly in a float conversion would lose precision because the mantissa only holds 24 bits [S2, unofficial]. The `>> 8` shift is critical [S2, unofficial].
- **Platform Drift:** Avoid `sin()` or `cos()` for hashing if CPU and GPU code share the same hash function, as math library implementations may differ slightly [S2, unofficial].

## Subtopics

### PCG Random Number Generator
Wraps the PCG32 fast random number generator, providing seeded streams per job [S3, unofficial]. It seeds the engine for a specific job `jid` using a hash of the global seed and job ID: `m_engine.seed(hash(m_seed, jid))` [S3, unofficial]. It generates uniform floats in $[0, 1-\epsilon]$ via `val = engine()` and `return min(0x1.fffffep-1, val * 0x1p-32f)` [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Extremely fast due to simple bitwise operations and multiplication [G].
- **Strengths:** Bit-exact reproducibility across CPU/GPU boundaries; handles negative coordinates naturally; no precision loss in float conversion [S2, unofficial].
- **Limitations:** Not a cryptographic hash; statistical properties depend on the specific mixer constants [G].

## Background
This method originates from the `neon-raymarcher` project, which required identical procedural generation on both a TypeScript CPU renderer and a GLSL GPU renderer [S2, unofficial]. The `compute_hash` variant is sourced from the `mc` repository [S1, unofficial]. The use of specific constants like `0x9e3779b9` is common in hash functions to ensure good bit dispersion [G].

## Sources
[S1] nsf/mc, https://github.com/nsf/mc, fetched 2026-10-26 (unofficial)
[S2] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
[S3] Shibodd/volume_path_tracer, https://github.com/Shibodd/volume_path_tracer, fetched 2026-10-26 (unofficial)
