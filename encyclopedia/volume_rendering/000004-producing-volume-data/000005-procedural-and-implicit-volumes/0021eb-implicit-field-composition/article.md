# Implicit Field Composition

Also called: Density Function Composition, Density Function for Scene Generation, Implicit Field Combination

## Overview
Implicit Field Composition constructs a complex scalar field by combining multiple primitive fields (such as terrain, spheres, and cuboids) using set-theoretic operations like union, intersection, and difference [S1, S2, unofficial]. It solves the problem of defining complex, multi-material, or hybrid geometries without explicit mesh topology, serving as the primary input generator for dual contouring and ray-marching pipelines [S1, S2, unofficial]. Unlike [Metaball Field Evaluation](1017-metaball-field-evaluation.md), which uses smooth blending functions to create organic morphing shapes, Implicit Field Composition relies on sharp, non-differentiable Boolean operators (min/max) to preserve hard edges and distinct material boundaries [S1, S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   A query point $\vec{x} \in \mathbb{R}^3$ [G].
*   Primitive definitions:
    *   **Terrain:** Parameters for fractal noise generation (octaves, frequency, lacunarity, persistence) and vertical scaling [S2, unofficial].
    *   **Cuboid:** Origin $\vec{o}_c$ and half-dimensions $\vec{h}_c$ [S2, unofficial].
    *   **Sphere:** Origin $\vec{o}_s$ and radius $r_s$ [S2, unofficial].
*   Boolean operator configuration (fixed in the reference implementation as `max(-cube, min(sphere, terrain))` [S1, unofficial].

**Outputs:**
*   A scalar value $f(\vec{x})$ representing the signed distance or implicit value at $\vec{x}$ [G]. The zero-crossing $f(\vec{x})=0$ defines the composite surface [S1, S2, unofficial].

### Data Structures
No persistent data structures are required for the composition logic itself [G]. The primitives are evaluated statelessly [G]. However, the noise generation requires temporary storage for the accumulated noise value and the scaled position vector during the octave loop [G].

### Formulas and Definitions
The composite field $f(\vec{x})$ is defined as:
$$ f(\vec{x}) = \max(-C(\vec{x}), \min(S(\vec{x}), T(\vec{x}))) $$
where $C(\vec{x})$ is the cuboid field, $S(\vec{x})$ is the sphere field, and $T(\vec{x})$ is the terrain field [S1, S2, unofficial].

**1. Terrain Field $T(\vec{x})$**
The terrain is a height field derived from fractal noise [S1, unofficial].
$$ T(\vec{x}) = x_y - K \cdot N(\vec{x}_{xz}) $$
Where:
*   $x_y$ is the y-coordinate of $\vec{x}$ [G].
*   $K = 20.0$ is the vertical scaling factor [S1, S2, unofficial].
*   $N(\vec{u})$ is the fractal noise function evaluated at the horizontal position $\vec{u} = (x, z)$ [S1, S2, unofficial].

**Fractal Noise $N(\vec{u})$**
$$ N(\vec{u}) = 0.5 + 0.5 \sum_{i=0}^{O-1} \text{Perlin}(\vec{p}_i) \cdot A_i $$
Parameters [S1, S2, unofficial]:
*   $O = 4$ (octaves).
*   $\text{SCALE} = 1/128$.
*   $\vec{p}_0 = \vec{u} \cdot \text{SCALE} \cdot F$ [S2, unofficial].
*   $A_0 = 1.0$.
*   For $i > 0$: $\vec{p}_i = \vec{p}_{i-1} \cdot L$ and $A_i = A_{i-1} \cdot P$ [S2, unofficial].
*   $F = 0.5343$ (frequency).
*   $L = 2.2324$ (lacunarity).
*   $P = 0.68324$ (persistence).
*   $\text{Perlin}(\cdot)$ is a standard Perlin noise function returning values in $[-1, 1]$ [S2, unofficial].

**2. Cuboid Field $C(\vec{x})$**
The cuboid is defined as a Signed Distance Function (SDF) [S2, unofficial]:
$$ C(\vec{x}) = \min(m, \|\vec{d}_{pos}\|) $$
Where:
*   $\vec{l} = \vec{x} - \vec{o}_c$ (local position) [G].
*   $\vec{d} = (|\vec{l}_x| - h_{c,x}, |\vec{l}_y| - h_{c,y}, |\vec{l}_z| - h_{c,z})$ [S2, unofficial].
*   $m = \max(d_x, d_y, d_z)$ [S2, unofficial].
*   $\vec{d}_{pos}$ is the vector $\vec{d}$ with negative components clamped to zero: $\vec{d}_{pos} = \max(\vec{d}, \vec{0})$ [G].
*   $\|\cdot\|$ is the Euclidean norm [S2, unofficial].

**3. Sphere Field $S(\vec{x})$**
The sphere is defined as a standard SDF [S2, unofficial]:
$$ S(\vec{x}) = \|\vec{x} - \vec{o}_s\| - r_s $$
Where:
*   $\vec{o}_s = (15, 2.5, 1)$ [S2, unofficial].
*   $r_s = 16$ [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_implicit_field(x, y, z):
    """
    Evaluates the composite implicit field at point (x, y, z).
    Implements: max(-cuboid, min(sphere, terrain))
    """
    terrain = evaluate_terrain(x, y, z)
    cuboid = evaluate_cuboid(x, y, z)
    sphere = evaluate_sphere(x, y, z)
    return max(-cuboid, min(sphere, terrain))

def evaluate_terrain(x, y, z):
    """
    T(x,y,z) = y - 20.0 * FractalNoise(x, z)
    """
    noise_val = fractal_noise(x, z)
    return y - 20.0 * noise_val

def fractal_noise(x, z):
    """
    Generates fractal noise using 4 octaves of Perlin noise.
    """
    SCALE = 1.0 / 128.0
    p_x = x * SCALE
    p_y = z * SCALE
    
    frequency = 0.5343
    lacunarity = 2.2324
    persistence = 0.68324
    octaves = 4
    
    noise_sum = 0.0
    amplitude = 1.0
    
    # Initial scaling by frequency
    p_x *= frequency
    p_y *= frequency
    
    for i in range(octaves):
        # [G] Standard 2D Perlin noise implementation returning values in [-1, 1].
        perlin_val = perlin_noise(p_x, p_y)
        noise_sum += perlin_val * amplitude
        
        # Prepare next octave
        p_x *= lacunarity
        p_y *= lacunarity
        amplitude *= persistence
        
    # Normalize to [0, 1] as per source definition: 0.5 + 0.5 * noise
    return 0.5 + 0.5 * noise_sum

def evaluate_cuboid(x, y, z):
    """
    SDF for a cuboid.
    Source: tuckbone/DualContouringCSharp
    """
    origin = (-4.0, 10.0, -4.0)
    half_dims = (12.0, 12.0, 12.0)
    
    local_x = x - origin[0]
    local_y = y - origin[1]
    local_z = z - origin[2]
    
    # Absolute distance from center minus half-dimensions
    d_x = abs(local_x) - half_dims[0]
    d_y = abs(local_y) - half_dims[1]
    d_z = abs(local_z) - half_dims[2]
    
    # Max component
    m = max(d_x, max(d_y, d_z))
    
    # Vector of positive distances only
    d_pos_x = max(d_x, 0.0)
    d_pos_y = max(d_y, 0.0)
    d_pos_z = max(d_z, 0.0)
    
    # Euclidean norm of positive distances
    dist_pos = (d_pos_x**2 + d_pos_y**2 + d_pos_z**2) ** 0.5
    
    return min(m, dist_pos)

def evaluate_sphere(x, y, z):
    """
    SDF for a sphere.
    Source: tuckbone/DualContouringCSharp
    """
    origin = (15.0, 2.5, 1.0)
    radius = 16.0
    
    dist = ((x - origin[0])**2 + (y - origin[1])**2 + (z - origin[2])**2) ** 0.5
    return dist - radius

def perlin_noise(x, y):
    # [G] Standard 2D Perlin noise implementation returning values in [-1, 1].
    pass
```

### Parameters
*   **Terrain Octaves:** 4 [S1, S2, unofficial].
*   **Terrain Frequency:** 0.5343 [S1, S2, unofficial].
*   **Terrain Lacunarity:** 2.2324 [S1, S2, unofficial].
*   **Terrain Persistence:** 0.68324 [S1, S2, unofficial].
*   **Terrain Scale:** 1/128 [S1, S2, unofficial].
*   **Terrain Vertical Scale:** 20.0 [S1, S2, unofficial].
*   **Cuboid Origin:** $(-4, 10, -4)$ [S2, unofficial].
*   **Cuboid Half-Dimensions:** $(12, 12, 12)$ [S2, unofficial].
*   **Sphere Origin:** $(15, 2.5, 1)$ [S2, unofficial].
*   **Sphere Radius:** 16 [S2, unofficial].

### Edge Cases and Pitfalls
*   **Precision:** The use of `max` and `min` creates non-differentiable discontinuities in the gradient [G].
*   **Sign Convention:** The cuboid SDF returns positive values outside and negative inside [G]. The term `-cuboid` inverts this, making the interior positive [G]. This specific combination `max(-C, min(S, T))` implies a specific Boolean tree: The terrain and sphere are intersected (`min`), and the result is unioned with the inverted cuboid (`max`) [G].
*   **Noise Domain:** The fractal noise is 2D (x, z), while the primitives are 3D [S1, S2, unofficial]. This restricts the terrain to a height-field-like structure [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(O \cdot N)$ where $O$ is the number of octaves (4) and $N$ is the cost of the underlying noise function [G]. The primitive evaluations are $O(1)$ [G]. The composition is $O(1)$ [G].
*   **Memory Complexity:** $O(1)$ auxiliary space [G]. No geometry is stored during evaluation [G].
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
Implicit Field Composition is a technique rooted in constructive solid geometry (CSG) applied to implicit surfaces [G]. The specific implementation combining fractal terrain, cuboids, and spheres for dual contouring input originates from open-source educational examples in C# and C++ [S1, S2, unofficial]. The use of SDFs for primitives (cuboid, sphere) and Boolean composition via min/max is a standard approach in modern implicit modeling [G].

## Sources
[S1] DualContouringSample, https://github.com/nickgildea/DualContouringSample, fetched 2026-10-26 (unofficial)
[S2] DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
