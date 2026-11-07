# Implicit Field Composition

Also called: Density Function Composition, Density Function for Scene Generation, Implicit Field Combination

## Overview
Implicit Field Composition constructs a complex signed distance field (SDF) by combining multiple primitive fields (such as terrain, cuboids, and spheres) using set-theoretic operations [S1, S2, unofficial]. It solves the problem of defining intricate, multi-component geometry for dual contouring or ray marching without explicit mesh storage [S1, S2, unofficial]. It differs from [Metaball Field Evaluation](1017-metaball-field-evaluation.md) by using hard boolean operations (max/min) rather than smooth blending, and from [Superelliptic Blends](2051-superelliptic-blends.md) by relying on exact SDF primitives rather than approximate smooth unions [S1, S2, unofficial].

## How it works
**Inputs:**
- Query point $\mathbf{x} \in \mathbb{R}^3$.
- Primitive parameters:
  - Terrain: Octaves $O=4$, Frequency $f=0.5343$, Lacunarity $L=2.2324$, Persistence $P=0.68324$ [S2, unofficial].
  - Cuboid: Origin $\mathbf{o}_c = (-4, 10, -4)$, Half-dimensions $\mathbf{h}_c = (12, 12, 12)$ [S2, unofficial].
  - Sphere: Origin $\mathbf{o}_s = (15, 2.5, 1)$, Radius $r_s = 16$ [S2, unofficial].

**Outputs:**
- Scalar field value $f(\mathbf{x})$. Negative values indicate inside the volume, positive outside [Implicit Surface Definition](2798-implicit-surface-definition.md).

**Data Structures:**
- No persistent data structures are required for evaluation; all state is local to the query [S1, S2, unofficial].

**Steps:**
1. **Evaluate Terrain Field:**
   Compute a fractal noise height field and subtract it from the world Y coordinate.
   $$ \text{noise} = 0.5 + 0.5 \sum_{i=0}^{O-1} \text{Perlin}(\mathbf{p}_i) \cdot A_i $$
   Where:
   - $\mathbf{p}_0 = \mathbf{x}_{xz} \cdot \text{SCALE}$, with $\text{SCALE} = 1/128$ [S2, unofficial].
   - $\mathbf{p}_i = \mathbf{p}_{i-1} \cdot L$ for $i > 0$ [S2, unofficial].
   - $A_0 = 1.0$, $A_i = A_{i-1} \cdot P$ for $i > 0$ [S2, unofficial].
   - $\mathbf{x}_{xz} = (x, z)$ [S1, unofficial].
   - $\text{Perlin}(u, v)$ is a standard 2D Perlin noise function [S2, unofficial].
   $$ \text{terrain}(\mathbf{x}) = y - 20.0 \cdot \text{noise} $$
   [S1, S2, unofficial]

2. **Evaluate Cuboid SDF:**
   Compute the signed distance to an axis-aligned cuboid.
   $$ \mathbf{d}_{\text{local}} = \mathbf{x} - \mathbf{o}_c $$
   $$ \mathbf{d} = (|\mathbf{d}_{\text{local}}.x| - h_c.x, |\mathbf{d}_{\text{local}}.y| - h_c.y, |\mathbf{d}_{\text{local}}.z| - h_c.z) $$
   $$ m = \max(\mathbf{d}.x, \mathbf{d}.y, \mathbf{d}.z) $$
   $$ \mathbf{d}_{\text{pos}} = \begin{cases} \mathbf{d} & \text{if } m > 0 \\ \mathbf{0} & \text{otherwise} \end{cases} $$
   $$ \text{cuboid}(\mathbf{x}) = \min(m, \|\mathbf{d}_{\text{pos}}\|) $$
   [S2, unofficial]

3. **Evaluate Sphere SDF:**
   Compute the signed distance to a sphere.
   $$ \text{sphere}(\mathbf{x}) = \|\mathbf{x} - \mathbf{o}_s\| - r_s $$
   [S2, unofficial]

4. **Compose Fields:**
   Combine the primitives using boolean operations. The union is represented by `min` and intersection by `max` for SDFs [S1, S2, unofficial].
   $$ f(\mathbf{x}) = \max(-\text{cuboid}(\mathbf{x}), \min(\text{sphere}(\mathbf{x}), \text{terrain}(\mathbf{x}))) $$
   [S1, S2, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_implicit_field(x, y, z):
    # 1. Terrain
    scale = 1.0 / 128.0
    p_x = x * scale
    p_y = z * scale
    noise = 0.0
    amplitude = 1.0
    freq = 0.5343
    lacunarity = 2.2324
    persistence = 0.68324
    
    # Apply frequency to initial position
    p_x *= freq
    p_y *= freq
    
    for i in range(4): # 4 octaves
        noise += perlin_2d(p_x, p_y) * amplitude
        p_x *= lacunarity
        p_y *= lacunarity
        amplitude *= persistence
        
    noise = 0.5 + 0.5 * noise
    terrain = y - 20.0 * noise
    
    # 2. Cuboid
    origin_c = (-4.0, 10.0, -4.0)
    half_dims = (12.0, 12.0, 12.0)
    local_x = x - origin_c[0]
    local_y = y - origin_c[1]
    local_z = z - origin_c[2]
    
    d_x = abs(local_x) - half_dims[0]
    d_y = abs(local_y) - half_dims[1]
    d_z = abs(local_z) - half_dims[2]
    
    m = max(d_x, d_y, d_z)
    
    if m > 0:
        d_pos_x = d_x
        d_pos_y = d_y
        d_pos_z = d_z
    else:
        d_pos_x = 0.0
        d_pos_y = 0.0
        d_pos_z = 0.0
        
    dist_pos = sqrt(d_pos_x**2 + d_pos_y**2 + d_pos_z**2)
    cuboid = min(m, dist_pos)
    
    # 3. Sphere
    origin_s = (15.0, 2.5, 1.0)
    radius_s = 16.0
    sphere = sqrt((x - origin_s[0])**2 + (y - origin_s[1])**2 + (z - origin_s[2])**2) - radius_s
    
    # 4. Composition
    # max(-cuboid, min(sphere, terrain))
    term1 = -cuboid
    term2 = min(sphere, terrain)
    result = max(term1, term2)
    
    return result
```

**Edge Cases:**
- Precision: The `max(-cuboid, ...)` term inverts the cuboid's sign. Ensure the cuboid SDF is correctly signed (negative inside) before negation [S1, S2, unofficial].
- Boundaries: The terrain is infinite in X/Z but defined by Y. The cuboid and sphere are bounded. The composition creates a bounded volume where the sphere and terrain intersect, clipped by the cuboid's exterior [S1, S2, unofficial].

## Subtopics

### [Metaball Field Evaluation](1017-metaball-field-evaluation.md)
Uses smooth blending (sum of fields) instead of boolean max/min. Choose when organic, merging shapes are needed.

### [Superelliptic Blends](2051-superelliptic-blends.md)
Uses smooth approximations of boolean operations. Choose when GPU performance is critical and exact SDF continuity is less important than speed.

### [Soft Metablobby SDF](2753-soft-metablobby-sdf.md)
Combines metaballs with SDF properties. Choose for fluid-like interactions.

### [Implicit Surface Definition](2798-implicit-surface-definition.md)
Defines the mathematical basis for $f(\mathbf{x})$.

## Operating characteristics
- **Time Complexity:** $O(1)$ per query point, dominated by the fixed number of noise evaluations (4 octaves) and primitive distance calculations [S1, S2, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary memory. No grid or octree is stored during evaluation [S1, S2, unofficial].
- **Performance:** Highly parallelizable on GPU. The terrain noise evaluation is the most expensive step due to multiple Perlin calls [S2, unofficial].
- **Strengths:** Exact SDFs for boolean operations; no discretization artifacts; easy to modify geometry by changing parameters [S1, S2, unofficial].
- **Limitations:** Cannot represent non-manifold geometry easily; noise evaluation is slower than simple primitive checks [S2, unofficial].
- **Use Case:** Dual contouring input generation [S1, S2, unofficial].

## Background
Originated in the context of dual contouring for implicit surfaces, specifically in sample implementations for generating complex terrain and object combinations [S1, S2, unofficial]. The technique leverages the property that SDFs support exact boolean operations via min/max [S1, S2, unofficial].

## Sources
[S1] nickgildea/DualContouringSample, https://github.com/nickgildea/DualContouringSample, fetched 2026-10-26 (unofficial)
[S2] tuckbone/DualContouringCSharp, https://github.com/tuckbone/DualContouringCSharp, fetched 2026-10-26 (unofficial)
