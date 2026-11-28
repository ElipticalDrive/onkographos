# Ambient Occlusion via SDF/Geometry


## Overview
Ambient Occlusion via SDF/Geometry estimates local occlusion by querying an explicit geometric representation—such as a Signed Distance Field (SDF), a voxel grid, or a set of mesh primitives—to determine how much incoming ambient light is blocked by nearby surfaces. It differs from screen-space methods like [SSAO](3980-screen-space-ambient-occlusion-ssao.md) by leveraging global geometric data, ensuring accurate occlusion for self-occluding or complex structures that are not visible to the camera, while avoiding the depth-buffer artifacts inherent to post-processing techniques.

## Shared principles
This family of methods shares a common computational pattern distinct from screen-space or purely statistical approaches:
1. **Geometric Query**: Instead of relying on a depth buffer or random ray casting against a black-box scene, the method queries a specific data structure that encodes geometry. This structure is typically an SDF (returning distance to nearest surface), a voxel grid (returning density or occupancy), or a list of primitives (returning intersection or proximity) [G].
2. **Hemisphere Sampling**: The core operation involves sampling directions within the upper hemisphere defined by the surface normal $\mathbf{n}$ at a point $\mathbf{p}$ [G].
3. **Occlusion Accumulation**: For each sample direction, the method determines if the path to infinity (or a maximum radius) is blocked. If blocked, a weight is added to an occlusion accumulator. The weight often depends on the distance to the blocking geometry, with closer occluders contributing more to the darkening effect [G].
4. **Normalization**: The accumulated occlusion value is normalized to the range $[0, 1]$, where 1 represents full illumination and 0 represents full occlusion [G].
5. **Biasing**: To prevent self-intersection artifacts (acne) caused by floating-point precision errors when sampling near the surface, all methods in this family apply a small bias offset along the normal $\mathbf{n}$ before querying the geometry [G].

## Subtopics

### [SDF-based Ambient Occlusion](1881-sdf-based-ambient-occlusion.md)
Estimates occlusion by sampling an SDF along the surface normal at increasing distances, weighting closer samples more heavily. It uses the distance field to simulate light blockage based on object proximity, avoiding the unrealistic results of SSAO which relies on depth buffers. Unlike SSAO, this method leverages global geometric information stored in the SDF, providing more accurate occlusion in scenes with complex geometry or self-occlusion.

### [GVDB Ambient Occlusion](2193-gvdb-ambient-occlusion.md)
Estimates visibility by querying a hierarchical voxel grid (GVDB volume) using either point sampling or ray casting. It distinguishes itself by operating directly on a voxel grid, supporting discrete point queries for speed and continuous ray casting for smoother gradients, both utilizing a tangent-space basis for rotation.

### [Screen-Space Ambient Occlusion (SSAO)](3980-screen-space-ambient-occlusion-ssao.md)
A post-processing technique that estimates ambient occlusion by sampling points in a hemisphere around each pixel's surface position and checking if those points are occluded by geometry in the depth buffer. It is the first screen-space method, using only the Z-Buffer, making its cost independent of scene complexity and dependent only on output resolution.

### [Disk-Based Ambient Occlusion](4333-disk-based-ambient-occlusion.md)
Approximates surface geometry as a collection of disk-shaped elements at mesh vertices to compute AO and bent normals analytically, avoiding the cost of ray casting. It distinguishes itself by using explicit geometric primitives (disks) rather than implicit distance fields, allowing for direct analytical occlusion calculation between primitives.

### [Monte Carlo Ambient Occlusion with Distance Mapping](4335-monte-carlo-ambient-occlusion-with-distance-mapping.md)
Estimates AO by casting rays in random directions and numerically integrating the occlusion contribution using a distance mapping function. It differs from deterministic grid-based or screen-space methods by relying on stochastic sampling, allowing for flexible handling of complex geometry at the cost of noise and higher sample counts for convergence.

### Fake Ambient Occlusion
**Definition**: A simplified ambient occlusion estimation technique that approximates occlusion based on the angle between the surface normal and the scene's upward vector, rather than ray-marching for occluders. It does not reflect the influence of nearby occluders [S1, unofficial].
**Step**: Calculate occlusion using the dot product of the surface normal $\mathbf{n}$ and the unit upwards vector $\mathbf{u}$: $\text{occlusion} = 1 - (\mathbf{n} \cdot \mathbf{u})$. Scale the occlusion value by a scale factor $s$, clamp the result to the range $[0, 1]$, and return $1 - \text{clamp}(s \cdot \text{occlusion}, 0, 1)$ to produce the final brightness [S1, unofficial].
**Parameter**: $s$: A scale value used to adjust the strength of the fake ambient occlusion effect [S1, unofficial].
**Pitfall**: The method does not account for occlusion caused by nearby objects (occluders); it only considers self-occlusion relative to the upward direction [S1, unofficial].

### Hemispheric Ambient Occlusion
**Definition**: Calculates hemispheric ambient occlusion (0..1) from nearby terrain geometry by sampling along the normal direction, measuring how much the terrain rises relative to the sample distance [S2, unofficial].
**Step**: If AO is disabled or samples $\le 0$, return 1. Initialize $\text{occlusion} = 0$. Loop $i$ from 1 to samples: $d = (\text{aoRadius} \cdot i) / \text{samples}$. Sample point $\mathbf{s} = \mathbf{p} + \mathbf{n} \cdot d$. Calculate $\text{rise} = \text{heightAtExtended}(\text{field}, s_x, s_z) - s_y$. If $\text{rise} > 0$, add $\text{clamp01}(\text{rise} / (0.9 \cdot d)) / \text{samples}$ to occlusion. Return $\text{clamp01}(1 - \text{occlusion} \cdot \text{aoStrength})$ [S2, unofficial].
**Formula**: $\text{occlusion} = \sum_{i=1}^{\text{samples}} \frac{\text{clamp01}(\text{rise}_i / (0.9 \cdot d_i))}{\text{samples}}$, where $d_i = (\text{aoRadius} \cdot i) / \text{samples}$ and $\text{rise}_i = \max(0, \text{height}(s_{x,i}, s_{z,i}) - s_{y,i})$. Final AO = $\text{clamp01}(1 - \text{occlusion} \cdot \text{aoStrength})$ [S2, unofficial].
**Parameter**: $\text{aoSamples}$: number of samples; $\text{aoRadius}$: maximum radius; $\text{aoStrength}$: multiplier for the final occlusion value [S2, unofficial].

### Exponential Decay Ambient Occlusion
**Definition**: An improvement over Real Ambient Occlusion that weighs samples exponentially based on their distance from the surface, giving less importance to further samples [S3, unofficial].
**Step**: Initialize $\text{sum}$ and $\text{maxSum}$ to 0. Calculate the sample position $\mathbf{p}$ by moving from the surface position $\text{pos}$ along the normal by $(i+1) \cdot \text{_AOStepSize}$. Add the weighted distance field value at $\mathbf{p}$ to $\text{sum}$. The weight is $1 / 2^i$. Add the weighted maximum possible distance $(i+1) \cdot \text{_AOStepSize}$ to $\text{maxSum}$. The weight is $1 / 2^i$. Return the ratio $\text{sum} / \text{maxSum}$ [S3, unofficial].

## Background
The origins and specific historical development of these techniques are not detailed in the provided sources. SDF-based methods are commonly used in ray marching and implicit surface rendering [S3, unofficial]. GVDB techniques utilize hierarchical voxel processing on the GPU for dynamic geometry queries [S1, unofficial]. Disk-based methods provide analytical AO for static meshes [S1, unofficial]. Monte Carlo approaches with distance mapping integrate geometric backends into a stochastic solver [S1, unofficial]. Fake AO is a heuristic for scenarios where geometric queries are expensive [S1, unofficial].

## Sources
[S1] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
[S2] mingderwang/theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
[S3] Volumetric Rendering: Ambient Occlusion, https://www.alanzucconi.com/2016/07/01/ambient-occlusion/, fetched 2026-10-28 (unofficial)
