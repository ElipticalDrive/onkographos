# Projection and MIP Techniques
Also called: MIP (Maximum Intensity Projection)

## Overview
This family of techniques projects 3D volumetric data onto a 2D image plane by aggregating scalar values along viewing rays, discarding the depth-dependent opacity accumulation used in standard alpha blending [S1, S2, S3, S11, S12, S14, S17, S23, S24, S25, S26, S27, unofficial][S28][S33, S37, unofficial]. Unlike [Direct volume ray casting](19-direct-volume-ray-casting.md), which composites color and opacity front-to-back to simulate light transport, these methods are order-independent and produce X-ray-like images that lack explicit depth cues [S4, S6, S9, S27, unofficial][S28][S38, S39, unofficial]. They are chosen when the goal is to highlight extremal structures (e.g., high-density bones or contrast-enhanced vessels) or total density, rather than visualizing the internal volume structure [S5, S11, S23, unofficial][S29, book][S30, S35, unofficial].

## Shared principles
All methods in this family share the fundamental ray traversal loop defined in [Direct volume ray casting](19-direct-volume-ray-casting.md) [G]. For each pixel, a ray $r(t) = \mathbf{o} + t\mathbf{d}$ is intersected with the volume bounding box, discretized into $N$ samples, and the scalar value $s_i$ is retrieved via trilinear interpolation [S5, S10, S17, unofficial]. The core distinction lies in the **aggregation function** applied to the sequence of sampled scalars $\{s_i\}$ along the ray path:

1.  **Identity Element**: Each method initializes an accumulator with the identity element for its aggregation operator:
    *   Maximum: $-\infty$ (or minimum representable value) [S2, S12, S16, S17, S19, S20, S23, S25, S32, S33, S34, unofficial].
    *   Minimum: $+\infty$ (or maximum representable value) [S2].
    *   Summation: $0.0$ [G, parent].
2.  **Update Rule**: The accumulator is updated at each sample step $t_i$:
    *   MIP: $A \leftarrow \max(A, s_i)$ [S2, S12, S16, S26, S33, S34, S40, unofficial].
    *   MinIP: $A \leftarrow \min(A, s_i)$ [S2].
    *   Projection/Forward: $A \leftarrow A + s_i \cdot \Delta t$ [S1, unofficial][S2].
3.  **Classification**: The final aggregated scalar $A$ is mapped to a color $C$ using a transfer function $T(A)$ [S2, S12, S16, S25, S26, S27, S32, S34, unofficial].
4.  **Order Independence**: The result is invariant to the order of sample processing, allowing for flexible traversal strategies and early termination [S2][S29, book][S37, unofficial].

## Subtopics

### [Maximum Intensity Projection](902-maximum-intensity-projection.md)
Projects the maximum scalar value encountered along each ray. It emphasizes high-intensity structures like blood vessels in angiography. It differs from siblings by using the `max` operator, whereas [Projected Slabs Approximation](1449-projected-slabs-approximation.md) approximates perspective geometry, and [Forward Projection (aTx)](1492-forward-projection-atx.md) sums values.

### [Projected Slabs Approximation](1449-projected-slabs-approximation.md)
Accelerates perspective rendering by partitioning the view frustum into parallel slabs, approximating perspective projection with orthographic-like processing. It differs from standard ray casting by bounding the analytical error introduced by this geometric approximation.

### [Forward Projection (aTx)](1492-forward-projection-atx.md)
Sums interpolated sample values along the ray to mimic radiographic imaging physics. It represents total attenuation/density, losing depth information but preserving quantitative intensity relationships better than MIP.

### [Minimum Intensity Projection](5299-minimum-intensity-projection.md)
Projects the minimum scalar value along each ray, used to visualize low-density structures like air-filled lungs. It discards depth cues like MIP but selects the lowest rather than highest values.

### [Projection](7205-projection.md)
Sums voxel values along rays parallel to a chosen axis to visualize total accumulated intensity. It preserves quantitative information unlike MIP and discards depth via uniform opacity.

### Perspective Approximation via Orthogonal Slices
An approximation for perspective volume rendering that uses view-orthogonal slices instead of concentric spherical shells, which are expensive to generate [S2]. The only difference from exact ray casting is that the distances of sample positions along a ray depend on the ray's orientation [S2]. Exact perspective ray casting produces associated sample positions forming concentric spherical shells, not planes [S2].

## Background
Maximum Intensity Projection (MIP) was introduced as a standard technique for visualizing high-contrast structures in medical imaging, particularly CT angiography [S5, S11, S23, unofficial][S29, book][S30, S35, unofficial]. The term "MIP_Mode" appears in early visualization libraries like Visualization Library to denote this rendering state [S1, unofficial]. The approximation of perspective projection using orthogonal slices was discussed in the context of interactive texture-mapping volume rendering to leverage hardware speed [S2].

## Sources
[S1] GPU Raycast Volume Rendering with Transfer ... - Visualization Library, https://visualizationlibrary.org/documentation/pag_guide_raycast_volume.html, fetched 2026-10-26 (unofficial)
[S2] Interactive Volume Rendering Using 3D Texture-Mapping ..., https://web.cs.ucdavis.edu/~okreylos/PhDStudies/Winter2000/TextureMapping.html, fetched 2026-10-27
[S3] niklasgruener/volume_vis, https://github.com/niklasgruener/volume_vis, fetched 2026-10-26 (unofficial)
