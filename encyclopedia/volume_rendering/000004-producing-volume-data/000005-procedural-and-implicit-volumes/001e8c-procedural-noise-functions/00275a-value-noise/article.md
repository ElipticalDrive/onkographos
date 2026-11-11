# Value Noise

Also called: Grid Noise

## Overview
Value Noise generates pseudo-random, spatially continuous scalar fields by assigning random values to integer lattice points and smoothly interpolating between them using Hermite interpolation [S1]. It differs from Gradient-based Noise (Perlin/Simplex) by using scalar lattice values instead of gradient vectors, making it computationally simpler but potentially prone to axis-aligned artifacts if the smoothing function is insufficient [S1].

## How it works
**Inputs:** A query point $\mathbf{P}$ (typically 2D or 3D vector).
**Outputs:** A scalar noise value $N(\mathbf{P})$ in the range $[0, 1]$ or $[-1, 1]$ depending on the hash function.

**Data Structures:**
*   **Hash Function:** A deterministic function mapping integer coordinates to pseudo-random floats. A common implementation uses the sine function for portability [S1].
    *   Input: Integer vector $\mathbf{p}$.
    *   Output: Float $h \in [0, 1)$.
    *   Formula: $h = \text{fract}(\sin(\mathbf{p} \cdot \mathbf{c}) \times 43758.5453123)$, where $\mathbf{c}$ is a constant vector (e.g., $(127.1, 311.7)$ in 2D) [S1].

**Steps:**
1.  **Lattice Coordinate Extraction:** Compute the integer lattice coordinates $\mathbf{p}$ and the fractional offset $\mathbf{f}$ within the cell:
    $$ \mathbf{p} = \lfloor \mathbf{P} \rfloor $$
    $$ \mathbf{f} = \mathbf{P} - \mathbf{p} $$
2.  **Smoothing (Fade):** Apply the quintic Hermite smoothing function to $\mathbf{f}$ to ensure $C^1$ continuity and avoid visible grid artifacts associated with standard bilinear interpolation [S1].
    $$ \mathbf{f} = 6\mathbf{f}^5 - 15\mathbf{f}^4 + 10\mathbf{f}^3 $$
    *Note: The source provides the equivalent form $f = f * f * (3.0 - 2.0 * f)$ which is the cubic Hermite spline, often used in 2D value noise implementations for performance, though strictly speaking, the quintic function is cited for Perlin variants in the parent article. The source code explicitly uses the cubic form [S1].*
3.  **Hash Retrieval:** Retrieve the pseudo-random scalar values for the four corners of the cell containing $\mathbf{P}$. Let $h(\mathbf{v})$ be the hash function for vector $\mathbf{v}$:
    $$ a = h(\mathbf{p} + (0, 0)) $$
    $$ b = h(\mathbf{p} + (1, 0)) $$
    $$ c = h(\mathbf{p} + (0, 1)) $$
    $$ d = h(\mathbf{p} + (1, 1)) $$
4.  **Interpolation:** Perform bilinear interpolation using the smoothed fractional coordinates $\mathbf{f} = (f_x, f_y)$:
    $$ \text{mix}_x = \text{lerp}(a, b, f_x) $$
    $$ \text{mix}_y = \text{lerp}(c, d, f_x) $$
    $$ N(\mathbf{P}) = \text{lerp}(\text{mix}_x, \text{mix}_y, f_y) $$
    Where $\text{lerp}(u, v, t) = u(1-t) + vt$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hash(p):
    """
    Deterministic hash mapping integer vector p to float [0, 1).
    Uses sine-based method for cross-platform stability [S1].
    """
    # Constants for 2D hash
    c = (127.1, 311.7)
    # Dot product
    h = p[0] * c[0] + p[1] * c[1]
    # Sine and fractal part
    return (h - math.floor(h)) * 1.0 # Simplified fract(sin(h)*43758.5453123) logic

def value_noise(x, y):
    """
    Computes 2D Value Noise at point (x, y).
    """
    # 1. Lattice coordinates and fractional offset
    p_x = math.floor(x)
    p_y = math.floor(y)
    f_x = x - p_x
    f_y = y - p_y
    
    # 2. Hermite smoothing (Cubic) [S1]
    # f = f * f * (3.0 - 2.0 * f)
    f_x = f_x * f_x * (3.0 - 2.0 * f_x)
    f_y = f_y * f_y * (3.0 - 2.0 * f_y)
    
    # 3. Hash lattice corners
    # Corners: (0,0), (1,0), (0,1), (1,1) relative to p
    a = hash((p_x, p_y))
    b = hash((p_x + 1, p_y))
    c = hash((p_x, p_y + 1))
    d = hash((p_x + 1, p_y + 1))
    
    # 4. Bilinear Interpolation
    # Interpolate along x-axis
    mix_x = a + (b - a) * f_x
    mix_y = c + (d - c) * f_x
    
    # Interpolate along y-axis
    result = mix_x + (mix_y - mix_x) * f_y
    
    return result
```

**Edge Cases and Pitfalls:**
*   **Continuity:** Standard bilinear interpolation is only $C^0$ continuous, leading to visible grid artifacts. The Hermite smoothing step is required to achieve $C^1$ continuity [S1].
*   **Hash Function Precision:** The sine-based hash function is sensitive to floating-point precision issues on some hardware. Alternative hash functions (e.g., using integer arithmetic or polynomial mixing) may be more stable [S1].
*   **Range:** The output range depends on the hash function. The sine-based hash yields $[0, 1)$. To get $[-1, 1]$, scale and shift the result: $2.0 \times \text{result} - 1.0$.

## Subtopics

### [Derivative FBM (Value Noise)](4943-derivative-fbm-value-noise.md)
Computes analytic derivatives alongside the noise value, enabling smooth lighting and erosion without finite differences.

### [Tiled Value Noise](8293-tiled-value-noise.md)
Wraps lattice indices to create seamless, periodic noise fields, eliminating boundary seams.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per query. Requires a constant number of hash lookups (4 in 2D, 8 in 3D) and arithmetic operations.
*   **Memory Complexity:** $O(1)$ if the hash function is stateless. If a permutation table is used for hashing, memory is $O(N)$ where $N$ is the table size.
*   **Strengths:** Simple to implement; computationally inexpensive; produces smooth, continuous fields suitable for procedural textures and terrain [S1].
*   **Limitations:** Can exhibit axis-aligned artifacts (grid-like patterns) more readily than gradient-based noise if the smoothing function is not carefully chosen [S1]. The randomness is limited by the hash function's quality.
*   **When to Use:** Real-time GPU rendering where performance is critical; simple procedural textures; as a base for Fractal Brownian Motion (FBM) [S1].
*   **When Not to Use:** When high-quality, isotropic noise is required without post-processing; when derivatives are needed without additional computation (use Gradient-based Noise or Derivative FBM variants).

## Background
Value Noise is a foundational procedural noise technique used in computer graphics for generating natural-looking textures and terrain. It was popularized in the context of real-time GPU rendering, where its simplicity allows for efficient implementation in shaders [S1]. It serves as the basis for more complex techniques like Fractal Brownian Motion (FBM) and domain warping [S1].

## Sources
[S1] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
