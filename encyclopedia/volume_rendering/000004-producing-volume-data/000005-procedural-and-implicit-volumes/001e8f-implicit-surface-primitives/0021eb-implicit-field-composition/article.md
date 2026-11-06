# Implicit Field Composition

Also called: Density Function Composition, Density Function for Scene Generation, Implicit Field Combination

## Overview
Implicit Field Composition constructs complex volumetric or surface geometries by combining primitive scalar fields (such as terrain, cuboids, and spheres) using set-theoretic operations like union and intersection [S1, S2, unofficial]. It differs from single-primitive methods by enabling the procedural generation of hybrid scenes, specifically serving as the signed distance field (SDF) generator for dual contouring mesh extraction [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $\mathbf{x} \in \mathbb{R}^3$ (typically `worldPosition`).
*   **Output:** A scalar value $f(\mathbf{x})$ representing the combined implicit field. For SDFs, the sign indicates inside (negative) or outside (positive) [S2, unofficial].

**Data Structures:**
No persistent data structures are required for the evaluation itself. The method relies on local variables for intermediate field values.

**Parameters:**
*   **Fractal Noise:**
    *   `octaves`: 4 [S1, S2, unofficial]
    *   `frequency`: 0.5343 [S1, S2, unofficial]
    *   `lacunarity`: 2.2324 [S1, S2, unofficial]
    *   `persistence`: 0.68324 [S1, S2, unofficial]
    *   `scale`: $1/128$ [S1, S2, unofficial]
*   **Cuboid:**
    *   `origin`: $(-4, 10, -4)$ [S1, S2, unofficial]
    *   `halfDimensions`: $(12, 12, 12)$ [S2, unofficial] (Note: [S1, unofficial] uses `vec3(-4., 10.f, -4.f)` as origin and `vec3(12.f)` likely as half-dimensions or size, but [S2, unofficial] explicitly defines `halfDimensions` as $(12,12,12)$) [S1, S2, unofficial].
*   **Sphere:**
    *   `origin`: $(15, 2.5, 1)$ [S1, S2, unofficial]
    *   `radius`: 16 [S1, S2, unofficial]

**Steps:**
1.  **Evaluate Terrain Field:**
    *   Compute fractal noise $N(\mathbf{x})$ using Perlin noise.
    *   Calculate terrain height: $h(\mathbf{x}) = \mathbf{x}_y - 20 \cdot N(\mathbf{x})$.
    *   The terrain field value is $f_{terrain}(\mathbf{x}) = \mathbf{x}_y - h(\mathbf{x}) = 20 \cdot N(\mathbf{x})$. *Correction*: The sources define the terrain *surface* as $y = 20 \cdot noise$. The implicit field for a surface $y = g(x,z)$ is typically $f(x,y,z) = y - g(x,z)$.
        *   [S1, unofficial] states: `terrain = worldPosition.y - (20.f * noise)`.
        *   [S2, unofficial] states: `terrain = y - (20.0 * noise)`.
        *   Thus, $f_{terrain}(\mathbf{x}) = \mathbf{x}_y - 20 \cdot N(\mathbf{x})$.
2.  **Evaluate Cuboid Field:**
    *   Compute local position: $\mathbf{p}_{local} = \mathbf{x} - \mathbf{o}_{cuboid}$.
    *   Compute distance to box surface $f_{cuboid}(\mathbf{x})$ using the standard SDF formula:
        $$ \mathbf{d} = |\mathbf{p}_{local}| - \mathbf{h} $$
        $$ f_{cuboid}(\mathbf{x}) = \min(\max(\mathbf{d}_x, \mathbf{d}_y, \mathbf{d}_z), \text{length}(\max(\mathbf{d}, \mathbf{0}))) $$
        where $\mathbf{h}$ is the half-dimensions vector [S2, unofficial].
3.  **Evaluate Sphere Field:**
    *   Compute distance to sphere surface $f_{sphere}(\mathbf{x})$:
        $$ f_{sphere}(\mathbf{x}) = \|\mathbf{x} - \mathbf{o}_{sphere}\| - r_{sphere} $$
        [S1, S2, unofficial].
4.  **Combine Fields:**
    *   The final field is computed as:
        $$ f_{final}(\mathbf{x}) = \max(-f_{cuboid}(\mathbf{x}), \min(f_{sphere}(\mathbf{x}), f_{terrain}(\mathbf{x}))) $$
        [S1, S2, unofficial].
    *   *Note on Logic:* The term $-f_{cuboid}$ inverts the cuboid's SDF (making inside positive, outside negative) before taking the max with the min of the sphere and terrain. This effectively performs a boolean difference or union depending on the sign convention of the underlying SDFs. Since $f_{cuboid}$ is negative inside, $-f_{cuboid}$ is positive inside. The `max` with the other terms suggests the cuboid acts as a "hole" or exclusion zone if the other fields are negative, or a union if positive. Given the standard SDF convention (negative inside), `max(-S, T)` usually implies $S$ is subtracted from $T$ if $S$ is positive (inside S). However, since $S$ is negative inside, $-S$ is positive inside. This specific composition creates a shape where the cuboid volume is treated differently than the sphere/terrain union.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fractal_noise(position_2d, octaves=4, frequency=0.5343, lacunarity=2.2324, persistence=0.68324):
    """
    Computes fractal noise based on Perlin noise.
    Returns value in [0, 1] range typically, but source uses 0.5 + 0.5*noise.
    """
    scale = 1.0 / 128.0
    p = position_2d * scale
    noise_val = 0.0
    amplitude = 1.0
    p *= frequency
    
    for i in range(octaves):
        # Perlin noise returns value in [-1, 1] or [0, 1] depending on implementation.
        # Source [S2, unofficial] implies standard Perlin. 
        # Formula [S1, unofficial] says: N = 0.5 + 0.5 * sum(...). 
        # The loop accumulates sum.
        noise_val += perlin_noise(p.x, p.y) * amplitude
        p *= lacunarity
        amplitude *= persistence
        
    return 0.5 + 0.5 * noise_val

def sphere_sdf(world_position, origin, radius):
    return length(world_position - origin) - radius

def cuboid_sdf(world_position, origin, half_dimensions):
    local_pos = world_position - origin
    # d is the distance from the box boundaries along each axis
    d = abs(local_pos) - half_dimensions
    # m is the maximum distance to the closest face (outside distance)
    m = max(d.x, max(d.y, d.z))
    # If outside, return m. If inside, return the negative distance to the closest face.
    # The source uses: min(m, length(d>0?d:0))
    # This is the standard SDF for an AABB.
    outside_dist = length(max(d, 0.0))
    return min(m, outside_dist)

def terrain_sdf(world_position, noise_value):
    # Terrain surface is y = 20 * noise
    # Implicit field f(x,y,z) = y - surface_y
    return world_position.y - (20.0 * noise_value)

def implicit_field_composition(world_position):
    # 1. Compute Terrain
    # Noise depends on x and z
    noise_val = fractal_noise(vec2(world_position.x, world_position.z))
    f_terrain = terrain_sdf(world_position, noise_val)
    
    # 2. Compute Cuboid
    cuboid_origin = vec3(-4.0, 10.0, -4.0)
    cuboid_half_dims = vec3(12.0, 12.0, 12.0)
    f_cuboid = cuboid_sdf(world_position, cuboid_origin, cuboid_half_dims)
    
    # 3. Compute Sphere
    sphere_origin = vec3(15.0, 2.5, 1.0)
    sphere_radius = 16.0
    f_sphere = sphere_sdf(world_position, sphere_origin, sphere_radius)
    
    # 4. Combine
    # max(-cuboid, min(sphere, terrain))
    return max(-f_cuboid, min(f_sphere, f_terrain))
```

## Operating characteristics
*   **Time Complexity:** $O(1)$ per query point, proportional to the number of octaves in the noise function.
*   **Memory Complexity:** $O(1)$ auxiliary space.
*   **Strengths:** Allows for complex, organic shapes (terrain) combined with hard primitives (cuboids, spheres) in a single analytic field. Suitable for real-time ray marching or dual contouring.
*   **Limitations:** The resulting field is not a true SDF everywhere if the blending operations (max/min) do not preserve the distance property strictly (though the components are SDFs, the combination `max(-S, min(T1, T2))` may not yield a true Euclidean distance field everywhere, potentially affecting marching cubes/dual contouring quality if not handled with QEFs or specific extraction algorithms). The specific combination logic creates non-smooth derivatives at the boundaries of the primitives.

## Background
This method is derived from implementations of Dual Contouring for implicit surfaces, specifically combining procedural noise for terrain with geometric primitives for scene composition [S1, S2, unofficial]. It is used in educational and open-source graphics projects to demonstrate how to generate input data for mesh extraction algorithms like Dual Contouring [S1, S2, unofficial].

## Sources
[S1] nickgildea/DualContouringSample, https://github.com/nickgildea/DualContouringSample, fetched 2026-10-26 (unofficial)
[S2] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
