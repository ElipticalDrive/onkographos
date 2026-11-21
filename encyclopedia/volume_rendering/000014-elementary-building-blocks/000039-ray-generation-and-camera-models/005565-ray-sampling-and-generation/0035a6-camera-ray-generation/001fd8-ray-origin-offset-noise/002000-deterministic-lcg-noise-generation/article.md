# Deterministic LCG Noise Generation

Also called: Linear Congruential Generator, Rng

## Overview
Deterministic LCG Noise Generation is a pseudo-random number generator (PRNG) method that produces normalized floating-point values in the range $[-1, 1]$ using linear congruential recurrence and bit-shifting normalization [S1, S2, unofficial]. It is used in volume rendering to provide deterministic noise for ray origin offsetting, thereby avoiding banding artifacts during ray marching [Parent]. It differs from hash-based PRNGs by relying on sequential state updates rather than spatial hashing, and from incremental methods by its fixed modular arithmetic structure [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A seed value (initial state).
*   **Output:** A normalized float $v \in [-1, 1)$ (or $[-1, 1]$ depending on implementation details, see [S1, S2, unofficial]).

**Data Structures:**
The generator maintains a single state variable. Two variants are documented in the sources:
1.  **32-bit variant:** A single 32-bit unsigned integer (`u32`) state [S1, unofficial].
2.  **64-bit variant:** A single 64-bit unsigned integer (`u64`) state [S2, unofficial].

**Parameters:**
*   **32-bit constants:** Multiplier $a = 1664525$, Increment $c = 1013904223$ [S1, unofficial].
*   **64-bit constants:** Multiplier $a = 6364136223846793005$, Increment $c = 1442695040888963407$ [S2, unofficial]. These are the Numerical Recipes constants [S2, unofficial].
*   **Modulus:** Implicitly $2^{32}$ for the 32-bit variant and $2^{64}$ for the 64-bit variant, implemented via wrapping arithmetic [S1, S2, unofficial].

**Steps:**
1.  **Update State:** Compute the next state using the linear congruential formula:
    $$x_{n+1} = (x_n \cdot a + c) \pmod M$$
    where $M = 2^{32}$ or $2^{64}$, and arithmetic is performed with wrapping (modulo $2^k$) [S1, S2, unofficial].
2.  **Extract Bits:**
    *   For the 32-bit variant: Shift the state right by 8 bits (`state >> 8`) to extract the top 24 bits [S1, unofficial].
    *   For the 64-bit variant: Shift the state right by 40 bits (`state >> 40`) to extract the top 24 bits [S2, unofficial].
3.  **Normalize:** Convert the extracted integer bits to a float in $[-1, 1)$.
    *   For the 32-bit variant: Scale the 24-bit integer to $[0, 1)$ and then to $[-1, 1]$ via $v = (\text{bits} / 2^{24}) \cdot 2.0 - 1.0$ [S1, unofficial].
    *   For the 64-bit variant: Cast the shifted value to `f32` and divide by $8388608.0$ ($2^{23}$), then subtract 1.0: $v = (\text{bits} / 8388608.0) - 1.0$ [S2, unofficial]. Note that $8388608.0 = 2^{23}$, implying a 23-bit precision float conversion or specific bit-width handling [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lcg_noise_32(seed: u32) -> f32:
    """
    32-bit LCG noise generation as described in [S1, unofficial].
    Produces values in [-1, 1].
    """
    state = seed
    # Step 1: Update state
    # Constants from [S1, unofficial]: a=1664525, c=1013904223
    # Wrapping arithmetic is implicit in u32 multiplication/addition in many languages,
    # or explicit via modulo 2^32.
    state = (state * 1664525 + 1013904223) & 0xFFFFFFFF
    
    # Step 2: Extract top 24 bits
    # Shift right by 8 bits
    bits = state >> 8
    
    # Step 3: Normalize to [-1, 1]
    # Convert to f32, scale to [0, 1), then to [-1, 1]
    # 2^24 = 16777216.0
    val = (bits / 16777216.0) * 2.0 - 1.0
    return val

def lcg_noise_64(seed: u64) -> f32:
    """
    64-bit LCG noise generation as described in [S2, unofficial].
    Produces values in [-1, 1).
    Uses Numerical Recipes constants.
    """
    state = seed
    # Constants from [S2, unofficial]: a=6364136223846793005, c=1442695040888963407
    # Modulo 2^64 is implicit in u64 wrapping arithmetic
    state = (state * 6364136223846793005 + 1442695040888963407)
    
    # Step 2: Extract top 24 bits
    # Shift right by 40 bits
    bits = state >> 40
    
    # Step 3: Normalize to [-1, 1)
    # Cast to f32 and divide by 8388608.0 (2^23)
    # Note: 8388608.0 is 2^23. The source [S2, unofficial] specifies this divisor.
    val = (bits / 8388608.0) - 1.0
    return val
```

**Edge Cases and Pitfalls:**
*   **Precision:** The 64-bit variant casts to `f32` after shifting, which may lose precision if the upper bits are not sufficiently distinct, though the top 24 bits are preserved in the shift [S2, unofficial].
*   **Periodicity:** The 32-bit LCG has a maximum period of $2^{32}$, which may be insufficient for large scenes or long animations without reseeding [S1, unofficial]. The 64-bit variant has a period of $2^{64}$ [S2, unofficial].
*   **Correlation:** LCGs can exhibit lattice structures in higher dimensions, which may cause visible artifacts if used for multi-dimensional sampling without care [G].
*   **Range:** The 32-bit variant is described as producing values in $[-1, 1]$ [S1, unofficial], while the 64-bit variant produces $[-1, 1)$ [S2, unofficial]. Implementers should verify the exact range behavior at the boundaries.

## Subtopics

### Hash-based Pseudo-Random Number Generator
Uses spatial hashing to generate deterministic noise based on coordinates, avoiding state management. Choose when spatial coherence is required or when parallel access to independent noise values is needed. [Hash-based Pseudo-Random Number Generator](2896-hash-based-pseudo-random-number-generator.md)

### Incremental Random Number Generation
Updates the random state incrementally, often used for streaming or progressive rendering. Choose when memory bandwidth is limited or when noise needs to be generated sequentially along a ray. [Incremental Random Number Generation](4898-incremental-random-number-generation.md)

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample. The operations are simple integer arithmetic and bit shifts, making it very fast [S1, S2, unofficial].
*   **Memory Complexity:** $O(1)$ state storage (32 or 64 bits) [S1, S2, unofficial].
*   **Performance:** Extremely efficient on modern CPUs due to the lack of branching and complex memory access [S1, S2, unofficial].
*   **Strengths:** Deterministic, low memory footprint, fast execution [S1, S2, unofficial].
*   **Limitations:** Short period for 32-bit variant, potential lattice artifacts in high dimensions [G].
*   **When to use:** Suitable for simple noise generation where high statistical quality is not critical, or for ray origin offsetting where speed is paramount [Parent, S1, S2].
*   **When not to use:** When high-quality statistical randomness is required, or when long periods are needed without reseeding [G].

## Background
The 32-bit LCG variant is used in the Wildforge voxel game engine for procedural generation tasks [S1, unofficial]. The 64-bit variant uses constants from Numerical Recipes and is used in the roxlap portable reimplementation of Ken Silverman's engine for procedural sound synthesis [S2, unofficial]. The method is a standard technique in computer graphics for generating pseudo-random numbers [G].

## Sources
[S1] A Minecraft-alpha-style voxel game written in Rust with a custom engine, https://github.com/dollspace-gay/Wildforge, fetched 2026-10-26 (unofficial)
[S2] GitHub - NCrashed/roxlap: Rust portable reimplementation of Ken ..., https://github.com/NCrashed/roxlap, fetched 2026-10-29 (unofficial)
