# Incremental Random Number Generation

Also called: Golden Ratio Random, Quasi-Monte Carlo Random

## Overview
Incremental Random Number Generation is a method for producing pseudo-random numbers with low discrepancy properties, suitable for sampling in ray tracing and volume rendering [S1, unofficial]. It differs from its parent [Ray Origin Offset Noise](2118-ray-origin-offset-noise.md) by using a deterministic additive sequence based on irrational constants rather than a hash function, and from its sibling [Hash-based Pseudo-Random Number Generator](2896-hash-based-pseudo-random-number-generator.md) by avoiding bitwise operations in favor of floating-point arithmetic [S1, unofficial]. This approach ensures uniform distribution over iterations while maintaining statelessness across samples if the initial seed is managed externally.

## How it works
**Inputs:** A current random value $r_i$ (scalar for 1D, vector for 2D).
**Outputs:** The next random value $r_{i+1}$.
**Constants:**
- $\phi$ (Golden Ratio): $0.6180339887$ [S1, unofficial].
- $\pi$ (Pi): $3.1415926538$ [S1, unofficial].
- Multiplier: $4$ [S1, unofficial].

**Data Structures:** None required; state is held in the input variable.

**Steps:**
1. For 1D generation, add the product of the golden ratio and 4 to the current value, then take the fractional part.
2. For 2D generation, add a vector containing the golden ratio and pi to the current vector, then take the fractional part of each component.

**Formulas:**
The 1D sequence is defined as:
$$ r_{i+1} = \{ r_i + 4 \phi \} $$
where $\{ \cdot \}$ denotes the fractional part operation ($x - \lfloor x \rfloor$) [S1, unofficial].

The 2D sequence is defined as:
$$ r_{i+1} = \{ r_i + (\phi, \pi) \} $$
where the addition and fractional part are applied component-wise [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def incremental_rand_1d(rand: float) -> float:
    """
    Generates the next 1D incremental random number.
    :param rand: Current random value.
    :return: Next random value in the sequence.
    """
    GOLDEN_RATIO = 0.6180339887
    return (rand + GOLDEN_RATIO * 4) % 1.0

def incremental_rand_2d(rand: tuple) -> tuple:
    """
    Generates the next 2D incremental random number.
    :param rand: Current random value as (x, y).
    :return: Next random value as (x, y).
    """
    GOLDEN_RATIO = 0.6180339887
    PI = 3.1415926538
    new_x = (rand[0] + GOLDEN_RATIO) % 1.0
    new_y = (rand[1] + PI) % 1.0
    return (new_x, new_y)
```

**Edge Cases and Pitfalls:**
- Precision: Floating-point precision limits may cause periodicity in very long sequences. For standard ray tracing depths, this is negligible [S1, unofficial].
- Initialization: The initial value $r_0$ should be chosen to avoid degenerate patterns; a random seed or a fixed offset like 0.5 is common [G].

## Subtopics

### Blue Noise Jitter Generator
A variant that uses the golden ratio low-discrepancy sequence to initialize a lookup table for blue noise jitter [S2, unofficial]. It differs from standard incremental generation by precomputing a grid of values and adding temporal shifts for animation [S2, unofficial].
- **Initialization:** Create a table of size $N \times N$ (default 64). For each index $i$, compute `table[i] = (i * phi) % 1.0`, where $\phi = 0.618033988749895$ [S2, unofficial].
- **Pixel Jitter:** For pixel $(x, y)$ and frame index, calculate `px = abs(floor(x)) % size`, `py = abs(floor(y)) % size`. Get `baseOffset = table[py * size + px]`. Calculate `temporalShift = ((frameIndex % 16) * phi) % 1.0`. Return `(baseOffset + temporalShift) % 1.0` [S2, unofficial].
- **Subpixel Jitter:** Use a Halton sequence with bases 2 and 3. Compute `h2 = halton(frameIndex + 1, 2) - 0.5` and `h3 = halton(frameIndex + 1, 3) - 0.5` [S2, unofficial].
- **Halton Implementation:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def halton(index: int, base: int) -> float:
    result = 0.0
    f = 1.0 / base
    i = index
    while i > 0:
        result += f * (i % base)
        i = i // base
        f = f / base
    return result
```

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample. No loops or complex math beyond basic arithmetic [S1, unofficial].
- **Memory Complexity:** $O(1)$ for standard incremental generation. $O(N^2)$ for the Blue Noise Jitter variant due to the lookup table [S2, unofficial].
- **Strengths:** Extremely fast; low discrepancy ensures good coverage with few samples; easy to parallelize [S1, unofficial].
- **Limitations:** Not truly random; patterns may emerge in high-dimensional spaces if not carefully constructed [S1, unofficial].
- **When to use:** Ray origin offsetting, simple stratified sampling, or when performance is critical and true randomness is not required [S1, unofficial].

## Background
The method originates from the use of the golden ratio in quasi-Monte Carlo methods for low-discrepancy sequence generation [S1, unofficial]. It is implemented in various open-source voxel ray tracing demos and volumetric rendering shaders [S1, unofficial][S2, unofficial].

## Sources
[S1] newDINO/voxel_ray_demo, https://github.com/newDINO/voxel_ray_demo, fetched 2026-10-24 (unofficial)
[S2] nff747/volumetric-clouds-wgsl, https://github.com/nff747/volumetric-clouds-wgsl, fetched 2026-10-27 (unofficial)
