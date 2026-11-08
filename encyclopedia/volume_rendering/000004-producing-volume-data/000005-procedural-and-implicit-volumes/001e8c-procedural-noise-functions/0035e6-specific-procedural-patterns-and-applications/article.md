# Specific Procedural Patterns and Applications


## Overview
This family defines specific procedural algorithms for generating volumetric density, occupancy, and structural patterns by applying mathematical transformations, geometric constraints, or fractal iterations to spatial coordinates. It solves the problem of creating diverse, organic, or structured 3D volumes without relying on explicit geometry or large texture atlases, distinguishing itself from general noise generators by focusing on the final density field's shape and physical properties [S1, S2, unofficial].

## Shared principles
The methods in this family share a common architectural pattern: they take a spatial coordinate (and potentially auxiliary parameters) as input and produce a scalar density or boolean occupancy value as output. Unlike the base noise functions which generate raw stochastic values, these patterns impose specific structural logic:
1.  **Geometric Confinement**: Many methods restrict the volume to specific shapes (spheres, slabs) using distance functions or ramps [S1, unofficial].
2.  **Non-Linear Remapping**: Raw noise or turbulence values are often remapped via power functions, thresholds, or lookup tables to control sparsity and density falloff [S1, unofficial].
3.  **Iterative Transformation**: Some methods, like fractals, apply iterative coordinate transformations to determine inclusion in the volume [S1, unofficial].
4.  **Stateless Evaluation**: All methods are stateless and compute values on-the-fly based on input coordinates, avoiding persistent data structures for the core logic [G].

## Subtopics

### [Sine-based procedural voxel generation](2477-sine-based-procedural-voxel-generation.md)
Generates periodic 3D structures using trigonometric functions to determine occupancy and color. It differs from siblings by relying on deterministic sine waves and modulo arithmetic for periodicity rather than stochastic noise or radial ramps. Choose it for regular, repeating volumetric patterns.

### [Spherical Density Ramp with Noise](2610-spherical-density-ramp-with-noise.md)
Confines gas density to a spherical region using a distance-based ramp and noise offsets. It differs by combining a deterministic spherical falloff with precomputed lookup tables for turbulence and randomness, avoiding complex flow advection. Choose it for simulating steam or gas clouds rising from a source.

### [Mandelbulb Fractal Generation](3327-mandelbulb-fractal-generation.md)
Generates complex 3D fractal structures by iteratively applying polar coordinate power transformations. It differs by mapping coordinates through continuous spherical power functions rather than trigonometric sampling or radial ramps. Choose it for organic, self-similar fractal shapes.

### [Volumetric Density Generation](4894-volumetric-density-generation.md)
Creates sparse, organic density fields by applying power functions and thresholds to FBM noise. It differs by relying on non-linear remapping and hard cutoffs to create patchy structures like clouds or smoke. Choose it for general-purpose organic volumetric textures.

### Cloud Noise Generation
*   **Definition**: A noise function designed for 3D volumetric clouds that samples a 2D texture slice based on 3D coordinates to generate organic randomness [S1, unofficial].
*   **Step**: Compute integer part `p` and fractional part `f` of the input 3D coordinate `x`. Compute smooth interpolation factor `u` using the formula `f * f * (3. - 2. * f)`. Calculate 2D UV coordinates by combining `p.xy` with `p.z` scaled by `vec2(37.0, 239.0)` and adding `u.xy`. Sample the `uNoise` texture at `(uv + 0.5) / 256.0` with LOD 0.0, returning the `.yx` components. Mix the two texture components `tex.x` and `tex.y` using `u.z` as the weight, then scale the result to the range [-1, 1] by multiplying by 2.0 and subtracting 1.0 [S1, unofficial].
*   **Code**:
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

### Environment Constraints Generation
*   **Definition**: Determines initial constraints that define the overall look of the world by projecting a smooth noise function onto the horizontal plane [S2, unofficial].
*   **Parameter**: Uses Simplex noise as the base function. Defines three factors: Inlandness (1.0 inland, 0.5 coastal, 0.0 submerged), Temperature (1.0 hot, 0.5 temperate, 0.0 cold), and Lushness (1.0 densely packed with flora, 0.0 barren) [S2, unofficial].

## Background
The techniques in this family originate from real-time graphics and procedural generation research. Sine-based voxel generation and Mandelbulb fractals are rooted in raymarching and fractal geometry, popularized by demos and community resources like Shadertoy [S1, unofficial]. Spherical density ramps and volumetric density generation are standard techniques in volumetric raymarching for simulating atmospheric effects like clouds and smoke [S1, unofficial]. Environment constraints generation stems from procedural world-building engines, specifically the DigBuild engine, which uses noise projections to define biomes and terrain attributes [S2, unofficial].

## Sources
[S1] Real-time dreamy Cloudscapes with Volumetric Raymarching, https://blog.maximeheckel.com/posts/real-time-cloudscapes-with-volumetric-raymarching/, fetched 2026-10-28 (unofficial)
[S2] DigBuild: Sandbox Voxel Game and Engine Using Vulkan, https://gredos.usal.es/handle/10366/149914, fetched 2026-10-31 (unofficial)
