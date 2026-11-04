# Ambient Occlusion via SDF/Geometry

Also called: SDF-based Ambient Occlusion, Geometry-based Ambient Occlusion, Distance Field Ambient Occlusion

## Overview
This family of techniques estimates local ambient occlusion by querying a geometric representation—typically a Signed Distance Field (SDF) or a structured voxel grid—rather than performing explicit ray tracing against a triangle mesh or screen-space depth buffer. By evaluating the distance to the nearest surface along specific directions, these methods approximate the visibility of the hemisphere around a point. What sets this family apart from [JAOCC Shading](3634-jaocc-shading.md) (which uses precomputed spherical harmonics) and [Local Ambient Occlusion](4977-local-ambient-occlusion.md) (which often relies on screen-space or texture-based approximations) is the direct analytical or semi-analytical evaluation of geometric distance to determine occlusion factors.

## Shared principles
All methods in this family rely on the fundamental property of Signed Distance Fields (SDFs) or structured geometry representations: the value at a point $p$ represents the shortest distance to the surface of the object, with a sign indicating whether the point is inside (negative) or outside (positive) the object.

The core mechanism for estimating ambient occlusion (AO) involves sampling the distance field along a set of directions, typically concentrated in the upper hemisphere relative to the surface normal $\mathbf{n}$. Let $d(\mathbf{p}, \mathbf{v})$ be the distance to the surface in direction $\mathbf{v}$ from point $\mathbf{p}$. The occlusion factor is derived by comparing these distances to the expected distance in an unoccluded environment.

A common formulation for the occlusion contribution of a single sample at distance $d$ is based on the ratio of the actual distance to the maximum possible distance or a reference distance. For a hemisphere of radius $R$, if the distance to the nearest occluder is $d < R$, the point is considered partially occluded. The occlusion weight $w$ is often defined as:
$$ w = \max(0, 1 - \frac{d}{R}) $$
or similar variations that weight closer samples more heavily, as closer occluders block more ambient light.

The final AO value $AO(\mathbf{p})$ is typically an average of these weights over $N$ sampled directions $\mathbf{v}_i$ in the hemisphere:
$$ AO(\mathbf{p}) = 1 - \frac{1}{N} \sum_{i=1}^{N} w(d(\mathbf{p}, \mathbf{v}_i)) $$
where $AO=1$ is fully lit and $AO=0$ is fully occluded.

Some variants, such as Exponential Decay, modify the weighting function to decay exponentially with distance, giving less importance to distant occluders:
$$ w_i = \frac{1}{2^i} \frac{d(\mathbf{p}, \mathbf{v}_i)}{d_{max, i}} $$
where $d_{max, i}$ is the maximum possible distance for that sample step.

## Subtopics

### [SDF-based Ambient Occlusion](1881-sdf-based-ambient-occlusion.md)
Estimates local occlusion by sampling the SDF along the surface normal at increasing distances, weighting closer samples more heavily to simulate ambient light blockage. It is a simplified, single-direction approach compared to full hemisphere sampling. Choose this when computational resources are extremely limited and the primary occlusion source is self-occlusion along the normal (e.g., crevices).

### [GVDB Ambient Occlusion](2193-gvdb-ambient-occlusion.md)
A device function that computes ambient occlusion at a surface point using either point sampling or ray casting against a hemisphere of directions, with tangent-space rotation and distance-based falloff. It is designed for GPU-based voxel rendering (GVDB). Choose this when working with voxel-based geometry on the GPU and requiring hemisphere-aware occlusion.

### [Screen-Space Ambient Occlusion (SSAO)](3980-screen-space-ambient-occlusion-ssao.md)
A post-processing technique that estimates ambient occlusion by sampling points in a hemisphere around each pixel's surface position and checking if those points are occluded by geometry in the depth buffer. It is distinct because it uses screen-space depth rather than a global SDF or voxel grid. Choose this for real-time rasterization pipelines where global geometry data is not available or too expensive to query.

### [Monte Carlo Ambient Occlusion with Distance Mapping](4335-monte-carlo-ambient-occlusion-with-distance-mapping.md)
A method to compute ambient occlusion by casting rays in random directions and numerically evaluating the occlusion factor using a distance mapping function. It is distinct because it uses stochastic sampling rather than deterministic grid or hemisphere sampling. Choose this when high-quality, unbiased AO is required and deterministic sampling is not feasible.

#### Fake Ambient Occlusion
**Definition**
A simplified ambient occlusion estimation technique that approximates occlusion based on the angle between the surface normal and the scene's upward vector, rather than ray-marching for occluders. It does not reflect the influence of nearby occluders [S1, unofficial].

**Step**
1. Calculate occlusion using the dot product of the surface normal $\mathbf{n}$ and the unit upwards vector $\mathbf{u}$:
   $$ \text{occlusion} = 1 - (\mathbf{n} \cdot \mathbf{u}) $$
2. Scale the occlusion value by a scale factor $s$, clamp the result to the range $[0, 1]$, and return $1$ minus the clamped value to produce the final brightness [S1, unofficial].

**Parameter**
*   $s$: A scale value used to adjust the strength of the fake ambient occlusion effect [S1, unofficial].

**Pitfall**
The method does not account for occlusion caused by nearby objects (occluders); it only considers self-occlusion relative to the upward direction [S1, unofficial].

#### Disk-Based Ambient Occlusion
**Definition**
An approximation modeling surfaces as collections of disk-shaped elements at mesh vertices to compute AO and bent normals analytically [S2, unofficial].

**Step**
1. Pass 1: Perform a simple sum of all occlusion factors including double shadowing [S2, unofficial].
2. Pass 2: Reduce each disk's occlusion contribution by its occlusion factor from the previous pass [S2, unofficial].
3. Optimization: Use a hierarchical tree to optimize computations against distant disks, reducing complexity to $O(n \log n)$ [S2, unofficial].

**Pitfall**
Double shadowing: merely summing occlusion from all disks leads to an overly dark image because disks behind other disks still count towards occlusion [S2, unofficial].

**Complexity**
Naive implementation has $O(n^2)$ runtime complexity computing all disk pairs [S2, unofficial].

#### Hemispheric Ambient Occlusion
**Definition**
Calculates hemispheric ambient occlusion (0..1) from nearby terrain geometry by sampling along the normal direction, measuring how much the terrain rises relative to the sample distance [S3, unofficial].

**Step**
1. If AO disabled or samples $\le 0$, return 1. Initialize occlusion = 0 [S3, unofficial].
2. Loop $i$ from 1 to samples:
   - $d = (\text{aoRadius} \times i) / \text{samples}$
   - Sample point $sx = px + nx \times d$, $sy = py + ny \times d$, $sz = pz + nz \times d$
   - Calculate rise = $\text{heightAtExtended}(\text{field}, sx, sz) - sy$
   - If rise > 0, add $\text{clamp01}(\text{rise} / (0.9 \times d)) / \text{samples}$ to occlusion [S3, unofficial].
3. Return $\text{clamp01}(1 - \text{occlusion} \times \text{aoStrength})$ [S3, unofficial].

**Formula**
$$ \text{occlusion} = \sum_{i=1}^{\text{samples}} \frac{\text{clamp01}(\text{rise}_i / (0.9 \times d_i))}{\text{samples}} $$
where $d_i = (\text{aoRadius} \times i) / \text{samples}$, $\text{rise}_i = \max(0, \text{height}(sx_i, sz_i) - sy_i)$. Final AO = $\text{clamp01}(1 - \text{occlusion} \times \text{aoStrength})$ [S3, unofficial].

**Parameter**
*   $\text{aoSamples}$: number of samples to take for AO [S3, unofficial].
*   $\text{aoRadius}$: maximum radius for AO sampling [S3, unofficial].
*   $\text{aoStrength}$: multiplier for the final occlusion value [S3, unofficial].

#### Exponential Decay Ambient Occlusion
**Definition**
An improvement over Real Ambient Occlusion that weighs samples exponentially based on their distance from the surface, giving less importance to further samples [S4, unofficial].

**Step**
1. Initialize sum and maxSum to 0 [S4, unofficial].
2. Calculate the sample position $p$ by moving from the surface position (pos) along the normal by $(i+1) \times \text{\_AOStepSize}$ [S4, unofficial].
3. Add the weighted distance field value at $p$ to sum. The weight is $1 / 2^i$ [S4, unofficial].
4. Add the weighted maximum possible distance $(i+1) \times \text{\_AOStepSize}$ to maxSum. The weight is $1 / 2^i$ [S4, unofficial].
5. Return the ratio of sum to maxSum. If no occlusion, sum equals maxSum, resulting in 1 [S4, unofficial].

## Background
Ambient Occlusion via SDF/Geometry emerged as a practical solution for real-time rendering where traditional ray-traced AO was too expensive. The use of Signed Distance Fields for AO was popularized in the context of sphere tracing and voxel rendering. Bunnel's disk-based approximation [S2, unofficial] provided an early analytical approach for mesh-based geometry, while SDF-based methods leveraged the efficiency of distance field queries. The exponential decay variant [S4, unofficial] was introduced to improve the realism of SDF-based AO by better modeling the falloff of ambient light. These techniques are widely used in volumetric rendering, voxel games, and real-time graphics engines where geometry is represented implicitly or in structured grids.

## Sources
[S1] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
[S2] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S3] mingderwang/theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
[S4] Volumetric Rendering: Ambient Occlusion, https://www.alanzucconi.com/2016/07/01/ambient-occlusion/, fetched 2026-10-28 (unofficial)
