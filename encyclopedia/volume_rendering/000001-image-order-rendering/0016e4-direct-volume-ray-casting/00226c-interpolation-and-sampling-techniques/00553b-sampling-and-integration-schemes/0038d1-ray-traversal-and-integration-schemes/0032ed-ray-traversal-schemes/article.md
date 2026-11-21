# Ray Traversal Schemes

Also called: Accumulate Projection, Depth Projection, Max Intensity Projection, Projection Methods, Ray Traversal -Accumulate

## Overview
Ray Traversal Schemes define the aggregation logic applied to voxel samples collected along a ray to produce a final pixel value [S1, S2, S3, unofficial]. While [Ray Marching and Discretization](2788-ray-marching-and-discretization.md) determines where to sample, these schemes determine how to combine those samples [S2, unofficial]. They are distinguished from standard front-to-back alpha compositing by their ability to perform non-compositional aggregations (such as Maximum or Average) or to simplify the compositing process for specific visualization goals like X-ray simulation [S1, S2, S3, unofficial].

## How it works
**Inputs**: Ray origin $\mathbf{o}$, direction $\mathbf{d}$, step size $\Delta t$, volume data grid with voxel colors $C_i$ and opacities $\alpha_i$ (or scalar intensities), and transfer functions mapping scalar values to $C_i, \alpha_i$.
**Outputs**: Final pixel color $C$ and accumulated opacity $\alpha$.
**Data structures**: A 3D texture or grid storing voxel attributes; per-ray accumulators for color $C$ and opacity $\alpha$.
**Steps**:
1. Initialize accumulators: $C \leftarrow (0, 0, 0)$, $\alpha \leftarrow 0$.
2. Iterate along the ray from the near clipping plane to the far clipping plane with step $\Delta t$.
3. At each step, query the volume at position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$ to obtain sample color $C_s$ and opacity $\alpha_s$ using interpolation (e.g., trilinear) [S4, unofficial].
4. Apply the selected aggregation scheme to update $C$ and $\alpha$.
5. Terminate early if $\alpha \geq 1.0$ (for Accumulate) or after traversing the full volume extent.

**Formulas** (Accumulate/Front-to-Back):
$$C \leftarrow C_s \alpha_s (1 - \alpha) + C$$
$$\alpha \leftarrow \alpha_s (1 - \alpha) + \alpha$$
[S2, S3, S4, unofficial]

Other schemes modify step 4:
- **First**: Stop at the first voxel where $\alpha_s > \tau$ (threshold) [S1, S4, unofficial].
- **Max**: $C \leftarrow \max(C, C_s)$, $\alpha \leftarrow 1.0$ [S2, S4, unofficial].
- **Average**: Sum $C_s$ and count steps, return $\frac{1}{N}\sum C_s$ [S1, S2, S4, unofficial].
- **Depth**: Accumulate $t$ values weighted by $\alpha_s$ [S1, S4, unofficial].

**Pseudocode**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_traversal(ray_origin, ray_dir, volume, step_size, scheme):
    C = [0.0, 0.0, 0.0]
    alpha = 0.0
    t = 0.0
    intensity_sum = [0.0, 0.0, 0.0]
    step_count = 0
    while t < volume.max_depth:
        p = ray_origin + ray_dir * t
        Cs, alpha_s = volume.sample(p)
        if scheme == "accumulate":
            C = [Cs[i] * alpha_s * (1.0 - alpha) + C[i] for i in range(3)]
            alpha = alpha_s * (1.0 - alpha) + alpha
            if alpha >= 1.0:
                break
        elif scheme == "first":
            if alpha_s > 0.5:
                return Cs
        elif scheme == "max":
            if Cs > C:
                C = Cs
            alpha = 1.0
        elif scheme == "average":
            intensity_sum = [intensity_sum[i] + Cs[i] for i in range(3)]
            step_count += 1
        elif scheme == "depth":
            C = [t * alpha_s + C[i] for i in range(3)]
            alpha = alpha_s + alpha
        t += step_size
    if scheme == "average":
        return [intensity_sum[i] / step_count for i in range(3)] if step_count > 0 else C
    return C
```

**Edge cases and pitfalls**: Zero direction components require axis-aligned traversal logic to avoid division by zero; precision loss in deep volumes requires normalized depth or early termination; degenerate rays (origin inside volume) need near-plane clamping [G].

## Subtopics

### [Pixel Intensity Evaluation with Scene Aggregates](10890-pixel-intensity-evaluation-with-scene-aggregates.md)
Integrates radiance over a pixel footprint using aggregated visibility functions to handle inter-voxel correlation and background blending without order-dependent ray marching. It sets itself apart from sibling methods by precomputing visibility terms to approximate complex integration, allowing efficient level-of-detail rendering. Choose it when rendering scenes with complex background interactions or when inter-voxel correlation matters.

### [X-Ray Volume Rendering](11336-x-ray-volume-rendering.md)
Simulates a transmission view (such as a CT scan) by linearly accumulating or averaging voxel intensity values along a ray, ignoring alpha compositing for the final accumulation logic. It differs from its sibling [Pixel Intensity Evaluation with Scene Aggregates](10890-pixel-intensity-evaluation-with-scene-aggregates.md) by operating directly on volumetric data via ray traversal rather than scene aggregates. Choose it when a transparent/transmissive appearance representing mean or sum of densities is required.

### Occupancy Probability (Yan et al.)
Casts a ray from the pixel, samples all associated voxels, and chooses the one with the highest occupancy probability. Choose it for binary occupancy visualization [S5, unofficial].

### Stopping Probability (Tulsiani et al.)
Computes the probability that a ray stops at a certain distance, allowing rendering of depth maps, color images, and foreground masks. Choose it when differentiable depth/color outputs are needed [S5, unofficial].

### Bidirectional Emission-Absorption (Henzler et al.)
Introduces a bidirectional emission-absorption (EA) light radiation model for emitting materials, in addition to visual hull (VH) and absorption-only (AO) models. Choose it for scenes with self-illuminating volumetric materials [S5, unofficial].

### Iterative Accumulation (Lombardi et al.)
Casts a ray from each pixel and iteratively accumulates color and opacity probabilities. Remaining voxels are discarded when cumulative opacity reaches a maximum value. Choose it for differentiable rendering with early termination [S5, unofficial].

### Surface-Weighted Sum (Jiang et al.)
Casts a ray from the pixel and locates the surface voxel closest to the camera. The final pixel color is computed as a weighted sum of the surface voxel and its neighboring voxels, with shading based on normals computed from voxel values. Choose it for differentiable surface rendering from signed distance fields [S5, unofficial].

## Operating characteristics
Time and memory complexity per ray are not given in the sources. Measured performance on specific hardware and resolution scaling are not given in the sources. Strengths include hardware-friendly early termination for the Accumulate scheme [S2, unofficial], specialized visualization modes for MIP in angiography [S2, unofficial], and X-ray simulation for Average [S2, unofficial]. Limitations include order-dependence in standard accumulation and potential aliasing without proper sampling rates [G].

## Background
The classification of ray traversal schemes into Intensity, Max, Average, Accumulate, First, and Depth was established in early direct volume rendering literature [S1, S2, S3, unofficial]. The "Accumulate" scheme, which performs front-to-back compositing, is attributed to Levoy (1988) and is designed to make transparent layers visible [S1, S3, unofficial]. The "Max" scheme, or Maximum Intensity Projection (MIP), is widely used in Magnetic Resonance Angiography [S1, S2, S3, unofficial]. The "Average" scheme produces an X-ray-like picture by averaging values along the ray [S1, S2, S3, unofficial]. The "First" scheme extracts iso-surfaces [S1, S3, unofficial]. These schemes form the basis of the compositing stage in the volume rendering pipeline, following ray-tracing/resampling and preceding image pixel output [S2, S3, unofficial].

## Sources
[S1] PPT - Direct Volume Rendering (DVR): Ray-casting PowerPoint Presentation - ID:465503, https://slideserve.com/jariath/direct-volume-rendering-dvr-ray-casting, fetched 2026-10-24 (unofficial)
[S2] Direct Volume Rendering Techniques, https://www.scribd.com/presentation/380929057/10-raycasting, fetched 2026-10-25 (unofficial)
[S3] Direct Volume Rendering (DVR): Ray-casting | PPT - SlideShare, https://www.slideshare.net/slideshow/direct-volume-rendering-dvr-raycasting/265104120, fetched 2026-10-26 (unofficial)
[S4] Direct volume rendering ray casting, https://slidetodoc.com/direct-volume-rendering-dvr-raycasting-jian-huang-this/, fetched 2026-10-29 (unofficial)
[S5] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
