# Value Noise

Also called: Grid Noise

## Overview
Value Noise generates pseudo-random, spatially continuous scalar fields by placing independent random values at integer lattice points and smoothly interpolating between them using Hermite interpolation [S1]. It differs from Gradient Noise (such as Perlin or Simplex noise) by interpolating scalar values directly rather than dot products of gradients, which makes it simpler to implement but prone to axis-aligned artifacts if interpolation is not smoothed [S1, S5]. It serves as a foundational building block for procedural textures and volume density fields, often combined with Fractal Brownian Motion (FBM) to create multi-scale detail [S1].

## How it works
**Inputs:** A query point $P = (x, y)$ in 2D (or $(x, y, z)$ in 3D).
**Outputs:** A scalar value $V \in [0, 1]$ representing the noise intensity at $P$.

**Data Structures:**
*   **Hash Function:** A deterministic function mapping integer coordinates to a pseudo-random float in $[0, 1]$.
*   **Lattice Points:** The four (2D) or eight (3D) integer corners surrounding the query point.

**Steps:**
1.  **Coordinate Decomposition:** Compute the integer lattice coordinates $I = \lfloor P \rfloor$ and the fractional offset $F = P - I$. The fractional part $F$ lies in $[0, 1)$ and determines the interpolation weights [S1].
2.  **Smoothing:** Apply Hermite smoothing to the fractional offset to ensure $C^1$ continuity (smooth first derivatives) and eliminate visible grid artifacts. The smoothing function is:
    $$ F_{smooth} = 3F^2 - 2F^3 $$
    This is equivalent to $F \cdot F \cdot (3.0 - 2.0 \cdot F)$ [S1].
3.  **Hash Retrieval:** Retrieve the pseudo-random scalar values stored at the surrounding lattice points. For 2D, these are the four corners:
    *   $h_{00} = \text{hash}(I + (0, 0))$
    *   $h_{10} = \text{hash}(I + (1, 0))$
    *   $h_{01} = \text{hash}(I + (0, 1))$
    *   $h_{11} = \text{hash}(I + (1, 1))$
    The hash function typically uses a sine-based approach for compactness:
    $$ \text{hash}(p) = \text{fract}(\sin(\text{dot}(p, \vec{c})) \cdot 43758.5453123) $$
    where $\vec{c}$ is a constant vector like $(127.1, 311.7)$ [S1].
4.  **Interpolation:** Perform bilinear interpolation using the smoothed fractional coordinates $F_{smooth} = (u, v)$:
    *   Interpolate along the x-axis:
        $$ A = \text{mix}(h_{00}, h_{10}, u) $$
        $$ B = \text{mix}(h_{01}, h_{11}, u) $$
    *   Interpolate along the y-axis:
        $$ V = \text{mix}(A, B, v) $$
    Where $\text{mix}(a, b, t) = a + t(b - a)$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def hash_2d(p_x, p_y):
    """
    Deterministic hash mapping integer coordinates to [0, 1].
    Uses sine-based hash as per source [S1].
    """
    # Constants from source
    c_x = 127.1
    c_y = 311.7
    multiplier = 43758.5453123
    
    dot_val = p_x * c_x + p_y * c_y
    sin_val = math.sin(dot_val)
    # fract(x) = x - floor(x)
    return sin_val * multiplier - math.floor(sin_val * multiplier)

def value_noise_2d(x, y):
    """
    Computes 2D Value Noise at point (x, y).
    """
    # 1. Coordinate Decomposition
    # floor() returns integer part, fract() returns fractional part
    p_x = math.floor(x)
    p_y = math.floor(y)
    f_x = x - p_x
    f_y = y - p_y
    
    # 2. Smoothing (Hermite)
    # f = f * f * (3.0 - 2.0 * f)
    f_x = f_x * f_x * (3.0 - 2.0 * f_x)
    f_y = f_y * f_y * (3.0 - 2.0 * f_y)
    
    # 3. Hash Retrieval
    # Retrieve values at the four corners of the cell
    h00 = hash_2d(p_x, p_y)
    h10 = hash_2d(p_x + 1, p_y)
    h01 = hash_2d(p_x, p_y + 1)
    h11 = hash_2d(p_x + 1, p_y + 1)
    
    # 4. Interpolation
    # mix(a, b, t) = a + t * (b - a)
    # Interpolate along X
    mix_x_0 = h00 + f_x * (h10 - h00)
    mix_x_1 = h01 + f_x * (h11 - h01)
    
    # Interpolate along Y
    result = mix_x_0 + f_y * (mix_x_1 - mix_x_0)
    
    return result
```

**Parameters:**
*   **Lattice Spacing:** Implicitly 1.0 (integer grid).
*   **Hash Constants:** $(127.1, 311.7)$ and $43758.5453123$ are standard values provided in the source [S1].

**Edge Cases and Pitfalls:**
*   **Continuity:** Standard bilinear interpolation without smoothing results in $C^0$ continuity (visible creases). The Hermite smoothing step is mandatory for visual quality [S1].
*   **Precision:** The sine-based hash is sensitive to floating-point precision at very large coordinate values. For large-scale noise, coordinate wrapping or a more robust hash (e.g., integer arithmetic based) may be required [S1].
*   **Dimensionality:** The 2D implementation above extends to 3D by adding a third axis interpolation step and using 8 lattice points.

## Subtopics

### [Derivative FBM (Value Noise)](4943-derivative-fbm-value-noise.md)
Value noise implementation that returns both the noise value and its analytic derivatives (dFdx, dFdy) for use in lighting and erosion calculations.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per query. The number of operations is constant regardless of resolution, involving a fixed number of arithmetic operations and hash lookups.
*   **Memory Complexity:** $O(1)$ if using a procedural hash function. If pre-computed lattice values are stored, memory scales with the grid size, but procedural hashing avoids this [S1].
*   **Strengths:** Extremely fast to compute; simple to implement; produces smooth, organic-looking fields when combined with FBM.
*   **Limitations:** Lacks the directional coherence of Gradient Noise, which can lead to "blobby" artifacts. The hash function's quality directly impacts the visual randomness; poor hashes introduce patterns [S1].
*   **Scaling:** Scales well with Fractal Brownian Motion (FBM) by summing multiple octaves. Each octave adds a constant overhead [S1].

## Background
Value Noise is a fundamental technique in procedural generation, widely used in computer graphics for texture synthesis and volume rendering. It was popularized in real-time shader programming for its simplicity and efficiency on GPUs [S1]. The method relies on the principle of placing random scalars at lattice points and interpolating between them, a concept distinct from the gradient-based approaches introduced by Ken Perlin [S1].

## Sources
[S1] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
