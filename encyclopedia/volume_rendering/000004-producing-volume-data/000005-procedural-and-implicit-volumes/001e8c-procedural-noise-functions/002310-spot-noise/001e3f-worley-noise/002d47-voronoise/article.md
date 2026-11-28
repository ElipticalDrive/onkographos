# Voronoise

Also called: Voronoi-Noise Hybrid

## Overview
Voronoise is a procedural noise function that unifies value noise and Voronoi cellular patterns through a single interpolation parameter [S1]. It solves the problem of needing distinct algorithms for continuous noise and cellular textures by providing a smooth transition between the two, controlled by a smoothness parameter [S1]. Unlike standard Worley noise which returns only distance metrics, Voronoise returns a scalar value derived from weighted hash contributions, allowing it to mimic value noise at low smoothness and cellular noise at high smoothness [S1].

## How it works
**Inputs and Outputs:**
- Input `p`: 2D or 3D coordinate vector.
- Input `u`: Interpolation parameter. `u=0` yields value noise; `u=1` yields Voronoi patterns [S1].
- Input `v`: Smoothness parameter. `v=0` yields sharp cells; `v=1` yields smooth noise [S1].
- Output: Scalar float value in the range [0, 1] (or similar, depending on hash range) [S1].

**Data Structures:**
- `hash32(p)`: A hash function mapping integer coordinates to a pseudo-random 3D vector. The specific implementation cited uses a sin-free, cross-platform stable method involving fractal mixing [S1].
- `smoothstep(edge0, edge1, x)`: A standard Hermite interpolation function clamped to [0, 1] [S1].

**Steps:**
1.  Calculate the exponent `k` based on the smoothness parameter `v`:
    $$k = 1.0 + 63.0 \cdot (1.0 - v)^6$$
    This formula maps `v=0` to `k=64` (sharp falloff) and `v=1` to `k=1` (linear falloff) [S1].
2.  Determine the integer lattice coordinates `i` and fractional part `f` of the input `p`:
    $$i = \lfloor p \rfloor$$
    $$f = p - i$$
3.  Initialize accumulator `a` to zero vector [S1].
4.  Iterate over a 5x5 grid of lattice points `g` centered on `i`. The loop ranges from -2 to +2 for both x and y dimensions (for 2D input; analogous for 3D) [S1]:
    a.  Compute the offset vector `o` by hashing the integer coordinate `i + g` and scaling it:
        $$o = \text{hash32}(i + g) \cdot (u, u, 1.0)$$
        Note: The hash output is typically in [0, 1]. The scaling applies `u` to the spatial offset components and `1.0` to the value component [S1].
    b.  Compute the distance vector `d` from the fractional coordinate `f` to the perturbed lattice point:
        $$d = g - f + o_{xy}$$
        Here, `g` is the integer offset, `f` is the fractional part, and `o_{xy}` is the x and y components of `o` [S1].
    c.  Calculate the weight `w` using a power falloff based on the length of `d`:
        $$w = (1.0 - \text{smoothstep}(0.0, 1.414, \|d\|))^k$$
        The threshold `1.414` is approximately $\sqrt{2}$, representing the diagonal distance in the unit cell [S1].
    d.  Accumulate the weighted hash value and weight into `a`:
        $$a += (o_z \cdot w, w)$$
        Where `o_z` is the z-component of the hash offset, representing the random value associated with that lattice point [S1].
5.  Return the weighted average:
    $$\text{result} = \frac{a_x}{a_y}$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voronoise(p: vec2, u: float, v: float) -> float:
    # Calculate exponent k for smoothness control
    # v=0 -> k=64 (sharp), v=1 -> k=1 (smooth)
    k = 1.0 + 63.0 * pow(1.0 - v, 6.0)
    
    # Integer and fractional parts of input
    i = floor(p)
    f = fract(p)
    
    # Accumulator for weighted sum
    a = vec2(0.0)
    
    # Iterate over 5x5 grid around the cell
    for y in range(-2, 3):
        for x in range(-2, 3):
            g = vec2(float(x), float(y))
            
            # Hash integer coordinate to get random offset
            # hash32 returns vec3 in [0,1]
            o = hash32(i + g) * vec3(u, u, 1.0)
            
            # Distance from fractional part to perturbed lattice point
            d = g - f + o.xy
            
            # Weight calculation
            # smoothstep(0.0, 1.414, length(d)) clamps distance influence
            # 1.414 is approx sqrt(2)
            dist = length(d)
            s = smoothstep(0.0, 1.414, dist)
            w = pow(1.0 - s, k)
            
            # Accumulate: o.z is the random value, w is the weight
            a += vec2(o.z * w, w)
            
    # Return weighted average
    return a.x / a.y

def hash32(p: vec2) -> vec3:
    # Sin-free hash implementation from source [S1]
    p3 = fract(vec3(p.xyx) * vec3(0.1031, 0.1030, 0.0973))
    p3 += dot(p3, p3.yxz + 33.33)
    return fract((p3.xxy + p3.yzz) * p3.zyx)
```

**Parameters:**
- `u`: Interpolation factor. `0` produces value noise (interpolating hash values directly), `1` produces Voronoi noise (using hash values as cell offsets) [S1].
- `v`: Smoothness. `0` produces sharp, distinct cells (high `k`), `1` produces smooth, blended noise (low `k`) [S1].

**Edge Cases and Pitfalls:**
- Division by zero: If `a.y` (sum of weights) is zero, the result is undefined. This occurs if no lattice points contribute to the weight, which is unlikely given the 5x5 grid and `smoothstep` range, but should be guarded against in implementation [G].
- Precision: The hash function `hash32` uses `fract` and dot products which can suffer from floating-point precision issues at very large coordinate values [S1].
- Boundary: The 5x5 grid covers the current cell and neighbors. For `u=1`, the influence radius is effectively limited by the `smoothstep` threshold of `1.414`, which fits within the 5x5 grid (max distance to corner is $\sqrt{2^2+2^2} \approx 2.82$) [S1].

## Subtopics

### Metallic Flake Procedural Layer
A procedural technique for generating metallic paint appearances using 3D Voronoi cells to distribute metallic flakes [S2, unofficial]. It differs from standard Voronoise by applying the Voronoi structure to define flake positions, sizes, orientations, and colors, then replacing the base BSDF lobe with a narrow GGX flake lobe under a clear coat layer [S2, unofficial]. Choose this when rendering realistic car paints or metallic coatings with sparkle effects [S2, unofficial].

### Eroded Voronoi Weather Maps
A cloud generation technique that uses Voronoi cells whose boundaries are organically eroded by Simplex FBM coordinate warping, while maintaining cell identity [S3, unofficial]. It differs from standard Voronoise by integrating coordinate warping to erode cell edges and decoding cell IDs to determine local cloud properties like thickness, altitude, and density [S3, unofficial]. Choose this when generating realistic, puffy cloud structures with organic erosion patterns [S3, unofficial]. The weather map stores distance to edge, distance to center, cell ID, and density per cell [S3, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(1)$ per sample, proportional to the number of lattice points in the search grid (25 for 2D, 125 for 3D) [S1].
- Memory: $O(1)$ auxiliary memory for accumulators and hash lookups [S1].

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- Performance scales linearly with the number of octaves if used within an FBM structure, but the base Voronoise function is a single evaluation [S1].

**Strengths and Limitations:**
- Strengths: Unified control over noise and cellular patterns; smooth interpolation; no need for separate algorithms [S1].
- Limitations: Requires a 5x5 (or larger) grid lookup, which is more expensive than simple value noise [S1]. The hash function must be robust to avoid artifacts [S1].

**When to Use:**
- Use when a single noise function needs to transition between value noise and Voronoi patterns [S1].
- Use for procedural textures requiring both continuous and cellular features [S1].

**When Not to Use:**
- Use standard Worley noise if only distance metrics are needed [Parent].
- Use standard value noise if cellular patterns are not required [S1].

## Background
Voronoise was introduced as part of a unified noise function framework, specifically described in a shader development skills document [S1]. It builds on the principles of value noise and Voronoi noise, providing a hybrid approach [S1]. The technique is used in procedural texture generation and shader development [S1].

## Sources
[S1] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
[S2] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
[S3] Optimize cloud rendering with eroded Voronoi weather maps and stable linear raymarching, https://github.com/ricecake/boidsish/pull/679, fetched 2026-10-28 (unofficial)
