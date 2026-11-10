# Ambient Occlusion via SDF/Geometry

Also called: SDF-based Ambient Occlusion, GVDB Ambient Occlusion, Screen-Space Ambient Occlusion (SSAO), Disk-Based Ambient Occlusion, Monte Carlo Ambient Occlusion with Distance Mapping

## Overview
This family evaluates ambient occlusion by querying geometric representations—such as Signed Distance Fields (SDFs), volumetric density grids, screen-space depth buffers, or mesh primitives—to determine how much the local environment blocks indirect light [S1, S2, S3, S4, S5]. It differs from parent methods that rely on precomputed vertex colors or simple gradient approximations by explicitly sampling the geometry's proximity or density along specific directions to compute a visibility factor [S5].

## Shared principles
All methods in this family share the core mechanism of evaluating visibility or illumination at a specific point $\mathbf{p}$ within or on the boundary of a volume, based on a light source direction $\mathbf{l}$ or environment [G].

1.  **Visibility/Transmittance Evaluation:** The fundamental operation is determining how much light reaches $\mathbf{p}$ from a source [G]. This involves integrating the extinction coefficient $\sigma_t(\mathbf{x})$ along a path from the source to $\mathbf{p}$ (or vice versa) [G]. The transmittance $T$ is calculated as:
    $$ T = \exp\left(-\int_{\mathbf{p}_{source}}^{\mathbf{p}} \sigma_t(\mathbf{x}) \, d\mathbf{x}\right) $$
    In discrete implementations, this integral is approximated by summing $\sigma_t \cdot \Delta s$ over samples along the ray [G].

2.  **Local Shading Models:** For surface-like appearance in volumes, normals are approximated from the volume gradient $\mathbf{g} = \nabla V$ [G]. Usually, a simple absorption and emission model is used because it is relatively cheap to evaluate and gives good enough results [S1, unofficial].

3.  **Shadowing Approximation:** Shadows are computed by testing occlusion along the direction to the light source [G].

4.  **Ambient Occlusion:** Indirect illumination is approximated by integrating over the hemisphere or using simplified models (e.g., upward ray for sky light) [G]. The ambient contribution $L_{ambient}$ is often modulated by a shadowing factor based on local density:
    $$ L_{ambient} = L_{sky} \cdot \exp(-D_{up}) \cdot AO_{factor} $$
    where $D_{up}$ is the integrated density along an upward ray [G].

## Subtopics

### [SDF-based Ambient Occlusion](1881-sdf-based-ambient-occlusion.md)
Estimates local occlusion by sampling a Signed Distance Field (SDF) along the surface normal at increasing distances, weighting closer samples more heavily to simulate ambient light blockage. It approximates ambient occlusion by using the distance field itself to simulate light occlusion based on object proximity, avoiding the unrealistic results of Screen Space Ambient Occlusion (SSAO) which relies on depth buffers and surface normals in screen space. This method is capable of producing visually pleasing results by leveraging the geometric accuracy of the SDF representation.

### [GVDB Ambient Occlusion](2193-gvdb-ambient-occlusion.md)
Computes local occlusion at a surface point by sampling a hemisphere of directions against a volume density field, using either point sampling or ray casting with distance-based falloff. It differs from SDF-based AO by querying a volumetric density field directly rather than a signed distance function, and from SSAO by operating in world-space volume coordinates rather than screen-space depth buffers.

### [Screen-Space Ambient Occlusion (SSAO)](3980-screen-space-ambient-occlusion-ssao.md)
Screen-Space Ambient Occlusion (SSAO) is a post-processing technique that estimates ambient occlusion by sampling points in a hemisphere around each pixel's surface position and checking if those points are occluded by geometry in the depth buffer. It is the first screen-space method for ambient occlusion, using only the Z-Buffer to compute AO at full resolution. Unlike SDF-based methods which rely on global geometric representations, SSAO relies exclusively on screen-space data (depth and normals), meaning its performance cost is independent of scene complexity and depends only on output resolution.

### [Disk-Based Ambient Occlusion](4333-disk-based-ambient-occlusion.md)
Approximates surface geometry as a collection of disk-shaped elements at mesh vertices to compute ambient occlusion (AO) and bent normals analytically, avoiding the cost of ray casting. It distinguishes itself from SDF-based methods by operating directly on mesh primitives without voxelization or distance field construction, though it requires a two-pass approach to correct for double shadowing artifacts inherent in naive disk summation.

### [Monte Carlo Ambient Occlusion with Distance Mapping](4335-monte-carlo-ambient-occlusion-with-distance-mapping.md)
Computes ambient occlusion by casting rays in random directions and numerically evaluating the occlusion factor using a distance mapping function.

#### Fake Ambient Occlusion
**Definition**
A simplified ambient occlusion estimation technique that approximates occlusion based on the angle between the surface normal and the scene's upward vector, rather than ray-marching for occluders. It does not reflect the influence of nearby occluders [S1, unofficial].

**Step**
1. Calculate occlusion using the dot product of the surface normal $\mathbf{n}$ and the unit upwards vector $\mathbf{u}$: $\text{occlusion} = 1 - (\mathbf{n} \cdot \mathbf{u})$ [S1, unofficial].
2. Scale the occlusion value by a scale factor $s$, clamp the result to the range $[0, 1]$, and return $1 - \text{clamp}(s \cdot \text{occlusion}, 0, 1)$ to produce the final brightness [S1, unofficial].

**Parameter**
*   $s$: A scale value used to adjust the strength of the fake ambient occlusion effect [S1, unofficial].

**Pitfall**
The method does not account for occlusion caused by nearby objects (occluders); it only considers self-occlusion relative to the upward direction [S1, unofficial].

#### Hemispheric Ambient Occlusion
**Definition**
Calculates hemispheric ambient occlusion (0..1) from nearby terrain geometry by sampling along the normal direction, measuring how much the terrain rises relative to the sample distance [S2, unofficial].

**Step**
1. If AO is disabled or samples $\le 0$, return 1. Initialize $\text{occlusion} = 0$ [S2, unofficial].
2. Loop $i$ from 1 to $\text{samples}$:
   a. $d = (\text{aoRadius} \cdot i) / \text{samples}$ [S2, unofficial].
   b. Sample point $\mathbf{s}_x = \mathbf{p}_x + \mathbf{n}_x \cdot d$, $\mathbf{s}_y = \mathbf{p}_y + \mathbf{n}_y \cdot d$, $\mathbf{s}_z = \mathbf{p}_z + \mathbf{n}_z \cdot d$ [S2, unofficial].
   c. Calculate $\text{rise} = \text{heightAtExtended}(\text{field}, \mathbf{s}_x, \mathbf{s}_z) - \mathbf{s}_y$ [S2, unofficial].
   d. If $\text{rise} > 0$, add $\text{clamp01}(\text{rise} / (0.9 \cdot d)) / \text{samples}$ to $\text{occlusion}$ [S2, unofficial].
3. Return $\text{clamp01}(1 - \text{occlusion} \cdot \text{aoStrength})$ [S2, unofficial].

**Formula**
$$ \text{occlusion} = \sum_{i=1}^{\text{samples}} \frac{\text{clamp01}(\text{rise}_i / (0.9 \cdot d_i))}{\text{samples}} $$
where $d_i = (\text{aoRadius} \cdot i) / \text{samples}$ and $\text{rise}_i = \max(0, \text{height}(\mathbf{s}_{x_i}, \mathbf{s}_{z_i}) - \mathbf{s}_{y_i})$. Final AO = $\text{clamp01}(1 - \text{occlusion} \cdot \text{aoStrength})$ [S2, unofficial].

**Parameter**
*   $\text{aoSamples}$: number of samples to take for AO [S2, unofficial].
*   $\text{aoRadius}$: maximum radius for AO sampling [S2, unofficial].
*   $\text{aoStrength}$: multiplier for the final occlusion value [S2, unofficial].

#### Exponential Decay Ambient Occlusion
**Definition**
An improvement over Real Ambient Occlusion that weighs samples exponentially based on their distance from the surface, giving less importance to further samples [S3, unofficial].

**Step**
1. Initialize $\text{sum}$ and $\text{maxSum}$ to 0 [S3, unofficial].
2. Calculate the sample position $\mathbf{p}$ by moving from the surface position ($\text{pos}$) along the normal by $(i+1) \cdot \text{_AOStepSize}$ [S3, unofficial].
3. Add the weighted distance field value at $\mathbf{p}$ to $\text{sum}$. The weight is $1 / 2^i$ [S3, unofficial].
4. Add the weighted maximum possible distance $(i+1) \cdot \text{_AOStepSize}$ to $\text{maxSum}$. The weight is $1 / 2^i$ [S3, unofficial].
5. Return the ratio of $\text{sum}$ to $\text{maxSum}$. If no occlusion, $\text{sum}$ equals $\text{maxSum}$, resulting in 1 [S3, unofficial].

## Background
Ambient occlusion via geometric queries originated as a way to approximate indirect lighting in real-time rendering and volumetric visualization. Early implementations relied on screen-space techniques like SSAO to avoid the cost of ray tracing [S2, unofficial]. Later, as Signed Distance Fields (SDFs) became popular for implicit surfaces, methods like SDF-based AO emerged to provide more geometrically accurate occlusion estimates [S1, unofficial]. Volumetric approaches, such as GVDB AO, adapted these concepts to 3D texture data, allowing for occlusion in non-surface-based media [S1, unofficial]. The family also includes simplified heuristics like "Fake Ambient Occlusion" for performance-critical applications where geometric accuracy is secondary to speed [S1, unofficial].

## Sources
[S1] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
[S2] mingderwang/theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
[S3] Volumetric Rendering: Ambient Occlusion, https://www.alanzucconi.com/2016/07/01/ambient-occlusion/, fetched 2026-10-28 (unofficial)
