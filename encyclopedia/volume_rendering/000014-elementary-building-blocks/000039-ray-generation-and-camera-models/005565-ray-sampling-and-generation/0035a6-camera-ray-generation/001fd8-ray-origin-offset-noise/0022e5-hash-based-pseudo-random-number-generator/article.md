# Hash-based Pseudo-Random Number Generator

Also called: Hash Functions, Hash-based PRNG, Integer to Float Mapping, Pseudo-Random Number Generator (Hash), Random Number Generation for Noise, hash12, hash13

## Overview
A hash-based pseudo-random number generator maps integer or floating-point coordinates to a deterministic pseudo-random value in the range $[0, 1)$ or $[-1, 1]$ using arithmetic operations. It solves the problem of generating unique, seemingly random offsets for ray origins or lattice indices without storing large lookup tables. It is distinguished from [Incremental Random Number Generation](4898-incremental-random-number-generation.md) by its coordinate-based determinism rather than state-based iteration, and from [Deterministic Integer Hashing](4149-deterministic-integer-hashing.md) by its reliance on floating-point sine functions rather than bitwise integer operations.

## How it works
**Inputs:** A coordinate vector $p$ (typically `vec2` or `vec3`).
**Outputs:** A single floating-point value $r \in [0, 1)$.

**Data Structures:** None. Pure arithmetic functions.

**Parameters:**
*   Constants for the sine-based hash: $C_1 = 127.1, C_2 = 311.7$ (for 2D); $C_3 = 74.7$ (for 3D); Multiplier $M = 43758.5453123$.
*   Constants for the sin-free hash: $K = 0.1031$, $B = 33.33$.

**Steps (Sin-based variant):**
1.  Compute the dot product of the input coordinate $p$ and a constant vector $c$: $h = p \cdot c$.
2.  Apply the sine function: $s = \sin(h)$.
3.  Scale by the multiplier: $v = s \times M$.
4.  Extract the fractional part: $r = v - \lfloor v \rfloor$.

**Steps (Sin-free variant):**
1.  Scale the input $p$ by $K$: $p' = p \times K$.
2.  Compute a dot product of $p'$ with itself plus a bias: $d = p' \cdot (p' + B)$.
3.  Combine components and extract the fractional part.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def hash_sin_2d(p_x: float, p_y: float) -> float:
    """
    Sin-based hash for 2D coordinates.
    Constants from [S1, unofficial], [S4].
    """
    # Step 1: Dot product with constants
    h = p_x * 127.1 + p_y * 311.7
    
    # Step 2: Sine
    s = math.sin(h)
    
    # Step 3: Scale
    v = s * 43758.5453123
    
    # Step 4: Fract (fractional part)
    return v - math.floor(v)

def hash_sin_3d(p_x: float, p_y: float, p_z: float) -> float:
    """
    Sin-based hash for 3D coordinates.
    Constants from [S1, unofficial].
    """
    h = p_x * 127.1 + p_y * 311.7 + p_z * 74.7
    s = math.sin(h)
    v = s * 43758.5453123
    return v - math.floor(v)

def hash_sin_free_2d(p_x: float, p_y: float) -> float:
    """
    Sin-free hash for 2D coordinates.
    Constants from [S4].
    """
    # Step 1: Scale by 0.1031
    p3_x = (p_x * 0.1031) - math.floor(p_x * 0.1031)
    p3_y = (p_y * 0.1031) - math.floor(p_y * 0.1031)
    
    # Reconstruct vec3(p.xyx) logic: p3 = fract(vec3(p.xyx) * .1031)
    # In 2D, we treat p as (p.x, p.y, p.x) effectively for the dot product structure
    # However, the source code uses vec3(p.xyx). 
    # Let's implement the exact GLSL logic:
    # vec3 p3 = fract(vec3(p.xyx) * .1031);
    # p3 += dot(p3, p3.yzx + 33.33);
    # return fract((p3.x + p3.y) * p3.z);
    
    # Component 0: p.x * 0.1031
    c0 = p_x * 0.1031
    # Component 1: p.y * 0.1031
    c1 = p_y * 0.1031
    # Component 2: p.x * 0.1031
    c2 = p_x * 0.1031
    
    # Apply fract to each
    f0 = c0 - math.floor(c0)
    f1 = c1 - math.floor(c1)
    f2 = c2 - math.floor(c2)
    
    # p3.yzx means (f1, f2, f0)
    # p3.yzx + 33.33 means (f1+33.33, f2+33.33, f0+33.33)
    # dot(p3, p3.yzx + 33.33)
    # p3 is (f0, f1, f2)
    dot_val = f0 * (f1 + 33.33) + f1 * (f2 + 33.33) + f2 * (f0 + 33.33)
    
    # p3 += dot_val
    # This adds the scalar dot_val to each component
    f0 += dot_val
    f1 += dot_val
    f2 += dot_val
    
    # return fract((p3.x + p3.y) * p3.z)
    res = (f0 + f1) * f2
    return res - math.floor(res)
```

**Edge Cases and Pitfalls:**
*   **Precision Sensitivity:** The sin-based hash relies on the precision of the sine function and floating-point multiplication. On some GPUs, this may produce visible artifacts or statistical shortcomings [S1, unofficial], [S4].
*   **Statistical Quality:** The distribution of values may not be perfectly uniform or independent, which is acceptable for visual noise but insufficient for high-quality Monte Carlo rendering where integer/bitwise hashes (like Murmur) are recommended [S1, unofficial].
*   **Negative Coordinates:** The sin-based hash handles negative coordinates via the sine function's periodicity, but results may differ from integer-based hashes due to floating-point representation differences [S2, unofficial].

## Subtopics

### [Deterministic Integer Hashing](4149-deterministic-integer-hashing.md)
Uses bitwise operations for platform-independent reproducibility; choose when bit-exact CPU/GPU consistency is required.

### [Permutation Table Generation](4174-permutation-table-generation.md)
Generates a shuffled lookup table; choose when avoiding sine calculations is necessary for performance or when using value noise.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per coordinate. Very fast, consisting of a few multiplications, a dot product, a sine, and a fract operation.
*   **Memory Complexity:** $O(1)$. No storage required.
*   **Strengths:** Extremely lightweight, no memory access, easy to implement in shaders.
*   **Limitations:** Statistical quality is lower than integer-based hashes; precision-dependent on hardware [S1, unofficial].
*   **When to use:** Visual effects, procedural texture generation, ray origin offsetting where perfect statistical independence is not critical.
*   **When not to use:** High-quality Monte Carlo integration, cryptographic applications, or when bit-exact reproducibility across different hardware architectures is required.

## Background
The sin-based hash function `fract(sin(dot(co.xy, vec2(12.9898, 78.233))) * 43758.5453)` is widely attributed to Rey (1998) and cited by TestU01 [S3, unofficial]. It became a standard one-liner in shader programming due to its brevity and speed. The sin-free variant by Dave Hoskins was developed to address cross-platform precision issues [S4].

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] Minecraft Voxel DDA Raycast - Shadertoy, https://www.shadertoy.com/view/WccyWl, fetched 2026-10-26 (unofficial)
[S3] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S4] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
