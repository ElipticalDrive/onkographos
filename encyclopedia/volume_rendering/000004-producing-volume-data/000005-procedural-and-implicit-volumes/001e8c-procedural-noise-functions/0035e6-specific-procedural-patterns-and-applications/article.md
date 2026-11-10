# Specific Procedural Patterns and Applications

Also called: Procedural Volume Patterns, Implicit Volume Generators

## Overview
This family encompasses specific mathematical formulations and algorithmic patterns used to generate volumetric data, density fields, and geometric structures procedurally without explicit mesh storage. It solves the problem of creating complex, organic, or fractal 3D shapes and density distributions using deterministic functions. Unlike [Gradient-based Noise (Perlin/Simplex)](3860-gradient-based-noise-perlin-simplex.md) which provides raw noise values, or [Value Noise](4008-value-noise.md) which interpolates scalar values, this family applies specific transformations (such as polar exponentiation, spherical ramps, or sine-based occupancy) to define the final volume characteristics directly.

## Shared principles
The family shares the following mechanism, terms, and math:

1.  **Gradient Assignment**: Each integer lattice point (vertex) in the domain is assigned a pseudo-random gradient vector $\mathbf{g}$. In 3D, common sets include 12 face diagonals of a cube or 12 axis-aligned vectors [S1, unofficial][S2, unofficial].
2.  **Dot Product Contribution**: The noise value at a query point $\mathbf{P}$ is derived from the dot product between the gradient vector $\mathbf{g}_i$ at a lattice corner $i$ and the distance vector $\mathbf{d}_i = \mathbf{P} - \mathbf{V}_i$ from that corner to $\mathbf{P}$:
    $$ n_i = \mathbf{g}_i \cdot \mathbf{d}_i $$
3.  **Smoothing/Fade Function**: To ensure smooth transitions ($C^1$ continuity), the fractional coordinates of $\mathbf{P}$ within the cell are passed through a fade function $f(t)$ before interpolation. The quintic function $f(t) = 6t^5 - 15t^4 + 10t^3$ is standard for Perlin variants, while Simplex uses a different geometric approach [S1, unofficial][S2, unofficial].
4.  **Interpolation**: The contributions from the cell corners are combined using linear interpolation (`lerp`) along the axes, weighted by the fade values [S1, unofficial][S2, unofficial].
5.  **Hashing**: Integer coordinates are mapped to gradient indices using a permutation table to avoid modulo operations and ensure uniform distribution [S1, unofficial][S2, unofficial].

## Subtopics

### [Sine-based procedural voxel generation](2477-sine-based-procedural-voxel-generation.md)
Creates volumetric data by evaluating trigonometric functions against spatial coordinates to determine voxel occupancy and color. It solves the problem of generating deterministic, periodic, and smooth geometric structures without storing explicit mesh or voxel grid data. Unlike [Simple Heightmap Voxel Generation](4649-simple-heightmap-voxel-generation.md) which focuses on planar terrain, or [Procedural Tree Voxel Generation](4445-procedural-tree-voxel-generation.md) which uses hierarchical branching, this method relies on global periodic wave interference to define shape boundaries and internal density variations.

### [Spherical Density Ramp with Noise](2610-spherical-density-ramp-with-noise.md)
Is a procedural volume density function that confines a gas (such as steam or smoke) to a spherical slab region using a spherical distance ramp and an offset table, simulating steam rising from a source. It differs from its siblings by combining a base turbulence pattern with a specific radial falloff mechanism that uses a precomputed cosine ramp and spatially varying noise offsets to break symmetry, whereas [Sine-based procedural voxel generation](2477-sine-based-procedural-voxel-generation.md) relies on sinusoidal modulation and [Mandelbulb Fractal Generation](3327-mandelbulb-fractal-generation.md) uses iterative complex number transformations.

### [Mandelbulb Fractal Generation](3327-mandelbulb-fractal-generation.md)
Generates a 3D fractal by iteratively applying a polar coordinate power transformation to a complex 3D point. It differs from its siblings by using continuous polar coordinate exponentiation rather than discrete modulo arithmetic (Menger Sponge) or simple polynomial mappings (Bristorbrot).

### [Volumetric Density Generation](4894-volumetric-density-generation.md)
Generates a patchy volumetric density field from procedural noise by applying a power function and a threshold to create sparse, organic structures. It differs from its parent [Specific Procedural Patterns and Applications](7615-specific-procedural-patterns-and-applications.md) by explicitly combining Fractal Brownian Motion (FBM) with non-linear remapping and hard cutoffs, rather than relying on raw noise values or simple linear blending.

### [Aperiodic Tiling with Edge Matching](10394-aperiodic-tiling-with-edge-matching.md)
A method to map pre-shaded texture cells onto a terrain or volume surface without visible repetition, using aperiodic boundary conditions and edge-matching constraints. It differs from standard periodic tiling by enforcing specific edge correlations to prevent seams, and from simple noise-based tiling by relying on pre-computed geometric patterns rather than procedural generation at render time.

#### Cloud Noise Generation
- **Definition**: A noise function designed for 3D volumetric clouds that samples a 2D texture slice based on 3D coordinates to generate organic randomness [S1, unofficial].
- **Steps**:
    1. Compute integer part `p` and fractional part `f` of the input 3D coordinate `x` [S1, unofficial].
    2. Compute smooth interpolation factor `u` using the formula `u = f * f * (3. - 2. * f)` [S1, unofficial].
    3. Calculate 2D UV coordinates by combining `p.xy` with `p.z` scaled by `vec2(37.0, 239.0)` and adding `u.xy` [S1, unofficial].
    4. Sample the `uNoise` texture at `(uv + 0.5) / 256.0` with LOD 0.0, returning the `.yx` components [S1, unofficial].
    5. Mix the two texture components `tex.x` and `tex.y` using `u.z` as the weight, then scale the result to the range [-1, 1] by multiplying by 2.0 and subtracting 1.0 [S1, unofficial].
- **Code**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```glsl
float noise(vec3 x ) {
  vec3 p = floor(x);
  vec3 f = fract(x);
  vec2 u = f * f * (3. - 2. * f);
  vec2 uv = (p.xy + vec2(37.0, 239.0) * p.z) + u.xy;
  vec2 tex = textureLod(uNoise,(uv + 0.5) / 256.0, 0.0).yx;
  return mix( tex.x, tex.y, u.z ) * 2.0 - 1.0;
}
```

#### Environment Constraints Generation
- **Definition**: Determines initial constraints that define the overall look of the world by projecting a smooth noise function onto the horizontal plane [S2, unofficial].
- **Parameters**:
    - **Noise function**: Simplex noise [S2, unofficial].
    - **Inlandness**: 1.0 (inland), 0.5 (coastal), 0.0 (submerged) [S2, unofficial].
    - **Temperature**: 1.0 (hot), 0.5 (temperate), 0.0 (cold) [S2, unofficial].
    - **Lushness**: 1.0 (densely packed with flora), 0.0 (barren) [S2, unofficial].

## Background
Procedural patterns for volume generation have evolved from simple mathematical shapes to complex fractal algorithms and noise-based density fields. The Mandelbulb, introduced in the early 2000s, popularized 3D fractal generation via polar coordinate transformations [S1, unofficial]. Cloud noise techniques, such as the texture-slice method, emerged from real-time raymarching demonstrations, allowing for organic volumetric effects without heavy computational overhead [S1, unofficial]. Environment constraint generation, as seen in engines like DigBuild, utilizes projected noise to define biomes and terrain properties, linking procedural generation to semantic world-building [S2, unofficial].

## Sources
[S1] Real-time dreamy Cloudscapes with Volumetric Raymarching, https://blog.maximeheckel.com/posts/real-time-cloudscapes-with-volumetric-raymarching/, fetched 2026-10-28 (unofficial)
[S2] DigBuild: Sandbox Voxel Game and Engine Using Vulkan, https://gredos.usal.es/handle/10366/149914, fetched 2026-10-31 (unofficial)
