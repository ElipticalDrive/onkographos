# Specific Procedural Patterns and Applications

Also called: Procedural Volume Patterns, Noise Applications

## Overview
This family encompasses specific mathematical constructions and algorithmic patterns used to generate volumetric data, density fields, and structural geometry from procedural inputs. It solves the problem of creating complex, organic, or structured 3D content without explicit mesh storage by defining deterministic functions over spatial coordinates. It differs from its siblings by focusing on distinct functional forms—such as trigonometric occupancy, spherical confinement, fractal iteration, and noise remapping—rather than the underlying noise generation primitives themselves.

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
Creates volumetric structures by evaluating trigonometric functions against voxel coordinates to determine occupancy and material properties. It solves the problem of generating deterministic, periodic 3D patterns without storing explicit geometry or heightmaps, relying instead on real-time evaluation of sine functions. It differs from siblings like [Spherical Density Ramp with Noise](2610-spherical-density-ramp-with-noise.md) by using direct coordinate mapping for structure rather than turbulence confinement. See [Sine-based procedural voxel generation](2477-sine-based-procedural-voxel-generation.md).

### [Spherical Density Ramp with Noise](2610-spherical-density-ramp-with-noise.md)
Is a procedural volume density function that confines a gas to a slab region using a spherical distance ramp and an offset table, simulating steam rising from a source. It differs from siblings like [Sine-based procedural voxel generation](2477-sine-based-procedural-voxel-generation.md) and [Mandelbulb Fractal Generation](3327-mandelbulb-fractal-generation.md) by applying a deterministic spherical falloff combined with a discrete random offset table to modulate base turbulence density, avoiding complex flow advection. See [Spherical Density Ramp with Noise](2610-spherical-density-ramp-with-noise.md).

### [Mandelbulb Fractal Generation](3327-mandelbulb-fractal-generation.md)
Generates a 3D fractal by iteratively applying a polar coordinate power transformation to a 3D point. It differs from its siblings, such as [Sine-based procedural voxel generation](2477-sine-based-procedural-voxel-generation.md) and [Spherical Density Ramp with Noise](2610-spherical-density-ramp-with-noise.md), by mapping coordinates through continuous spherical power functions rather than trigonometric sampling or radial ramps. See [Mandelbulb Fractal Generation](3327-mandelbulb-fractal-generation.md).

### [Volumetric Density Generation](4894-volumetric-density-generation.md)
Generates a patchy volumetric density field from procedural noise by applying a power function and a threshold to create sparse, organic structures. It differs from its parent [Specific Procedural Patterns and Applications](7615-specific-procedural-patterns-and-applications.md) by explicitly combining Fractal Brownian Motion (FBM) with non-linear remapping and hard cutoffs, rather than relying on raw noise values or simple linear blending. See [Volumetric Density Generation](4894-volumetric-density-generation.md).

### [Aperiodic Tiling with Edge Matching](10394-aperiodic-tiling-with-edge-matching.md)
Is a method to map pre-shaded texture cells onto a terrain or volume surface without visible repetition, using aperiodic boundary conditions and edge-matching constraints. It differs from standard periodic tiling by enforcing specific edge correlations to prevent seams, and from simple noise-based tiling by relying on pre-computed geometric patterns rather than procedural generation at render time. See [Aperiodic Tiling with Edge Matching](10394-aperiodic-tiling-with-edge-matching.md).

### Cloud Noise Generation
#### definition
- [S1, unofficial] A noise function designed for 3D volumetric clouds that samples a 2D texture slice based on 3D coordinates to generate organic randomness.
#### step
- [S1, unofficial] Compute integer part `p` and fractional part `f` of the input 3D coordinate `x`.
- [S1, unofficial] Compute smooth interpolation factor `u` using the formula `f * f * (3. - 2. * f)`.
- [S1, unofficial] Calculate 2D UV coordinates by combining `p.xy` with `p.z` scaled by `vec2(37.0, 239.0)` and adding `u.xy`.
- [S1, unofficial] Sample the `uNoise` texture at `(uv + 0.5) / 256.0` with LOD 0.0, returning the `.yx` components.
- [S1, unofficial] Mix the two texture components `tex.x` and `tex.y` using `u.z` as the weight, then scale the result to the range [-1, 1] by multiplying by 2.0 and subtracting 1.0.
#### code
- [S1, unofficial] float noise(vec3 x ) {
  vec3 p = floor(x);
  vec3 f = fract(x);
  vec2 u = f * f * (3. - 2. * f);
  vec2 uv = (p.xy + vec2(37.0, 239.0) * p.z) + u.xy;
  vec2 tex = textureLod(uNoise,(uv + 0.5) / 256.0, 0.0).yx;
  return mix( tex.x, tex.y, u.z ) * 2.0 - 1.0;
}

### Environment Constraints Generation
#### definition
- [S2, unofficial] Determines initial constraints that define the overall look of the world by projecting a smooth noise function onto the horizontal plane.
#### parameter
- [S2, unofficial] Noise function: Simplex noise.
- [S2, unofficial] Inlandness: 1.0 (inland), 0.5 (coastal), 0.0 (submerged).
- [S2, unofficial] Temperature: 1.0 (hot), 0.5 (temperate), 0.0 (cold).
- [S2, unofficial] Lushness: 1.0 (densely packed with flora), 0.0 (barren).

## Background
Originating from the need for efficient, deterministic procedural content generation in real-time rendering and voxel engines, these patterns evolved alongside noise functions. Cloud noise generation was popularized in shader communities (e.g., Shadertoy) for volumetric raymarching [S1, unofficial]. Environment constraints generation is documented in the context of the DigBuild voxel engine for terrain biomes [S2, unofficial]. Mandelbulb generation stems from 3D fractal mathematics applied to raymarching [S1, unofficial].

## Sources
[S1] Real-time dreamy Cloudscapes with Volumetric Raymarching, https://blog.maximeheckel.com/posts/real-time-cloudscapes-with-volumetric-raymarching/, fetched 2026-10-28 (unofficial)
[S2] DigBuild: Sandbox Voxel Game and Engine Using Vulkan, https://gredos.usal.es/handle/10366/149914, fetched 2026-10-31 (unofficial)
