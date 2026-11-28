# Ray-marching on 3D Gaussians


## Overview
Ray-marching on 3D Gaussians is a volume rendering method that evaluates the continuous Gaussian mixture field along camera rays to compute accurate volumetric integrals, explicitly handling primitive overlap and self-attenuation [S1]. It serves as a ground-truth baseline to analyze the approximations inherent in standard rasterization-based 3D Gaussian Splatting (3DGS), such as ignoring spatial overlap and using approximate projected shapes [S1]. Unlike 3DGS, which relies on screen-space sorting and alpha blending, this method performs adaptive sampling along rays to resolve the exact order and contribution of overlapping Gaussians, ensuring precision governed by sample count rather than rasterization artifacts [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of $N$ 3D Gaussians, each defined by a mean position, a covariance matrix (or scale and rotation), an opacity, and a color value [S1].
*   **Input:** Camera ray $r(t) = \mathbf{o} + t\mathbf{d}$ for each pixel [G].
*   **Output:** Final pixel color and accumulated opacity [S1].

**Data Structures**
The method reuses the standard 3DGS data structures [S1]:
*   **Tensors:** Arrays storing positions, scales, rotations (quaternions), opacities, and colors [S1].

**Algorithm Steps**
1.  **Adaptive Sampling:** Determine sample points along the ray. Unlike fixed-step marching, adaptive sampling increases density in regions where the Gaussian mixture field changes rapidly to accurately resolve overlap [S1].
2.  **Field Evaluation:** At each sample point $t_k$, evaluate the Gaussian mixture. The density $\sigma(t_k)$ and color $c(t_k)$ are computed by aggregating contributions from all active Gaussians $i$:
    $$ \sigma(t_k) = \sum_{i} \alpha_i \exp\left(-\frac{1}{2} d_i^2(t_k)\right) $$
    $$ c(t_k) = \frac{\sum_{i} \alpha_i \exp\left(-\frac{1}{2} d_i^2(t_k)\right) c_i}{\sigma(t_k) + \epsilon} $$
    where $d_i^2(t_k)$ is the squared Mahalanobis distance from the ray point to the Gaussian center, $\alpha_i$ is the Gaussian opacity, $c_i$ is the color, and $\epsilon$ prevents division by zero [G].
3.  **Integration:** Perform volumetric integration using the evaluated density and color. The standard volume rendering integral is discretized:
    $$ C = \sum_{k} T_k \cdot (1 - e^{-\sigma(t_k) \Delta t_k}) \cdot c(t_k) $$
    $$ T_{k+1} = T_k \cdot e^{-\sigma(t_k) \Delta t_k} $$
    where $\Delta t_k$ is the step size between samples, $T_k$ is the transmittance at step $k$, and $T_0 = 1$ [G].
4.  **Compositing:** Accumulate the color and opacity for the final pixel [S1].

**Parameters**
*   **Sample Count:** The number of samples per ray, governing integration precision [S1].

**Edge Cases and Pitfalls**
*   **Precision:** Floating-point precision errors can accumulate during long ray marches or when evaluating very small Gaussian contributions [G].
*   **Boundary Handling:** Gaussians truncated at their support radius may cause artifacts if the ray passes near the boundary without sufficient sampling [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def ray_march_gaussians(ray_origin, ray_direction, gaussians, max_samples=1000):
    """
    Ray-marches through a set of 3D Gaussians.
    ray_origin: (x, y, z)
    ray_direction: (x, y, z), normalized
    gaussians: List of dicts with keys: 'center' (3,), 'cov_inv' (3x3 list of lists), 'opacity' (float), 'color' (3,)
    """
    # 1. Adaptive Sampling: Collect ray bounds from Gaussian extents and subdivide
    bounds = []
    for g in gaussians:
        # Project Gaussian center to ray
        diff = (g['center'][0] - ray_origin[0], g['center'][1] - ray_origin[1], g['center'][2] - ray_origin[2])
        t_center = diff[0]*ray_direction[0] + diff[1]*ray_direction[1] + diff[2]*ray_direction[2]
        # Approximate radius from scale (simplified 3-sigma bound)
        r = math.sqrt(g['scale'][0]**2 + g['scale'][1]**2 + g['scale'][2]**2)
        bounds.append(max(0.0, t_center - r))
        bounds.append(t_center + r)
    
    bounds = sorted(set(bounds))
    if not bounds:
        return (0.0, 0.0, 0.0), 0.0

    # Subdivide intervals for adaptive refinement
    t_points = []
    for i in range(len(bounds) - 1):
        t_start, t_end = bounds[i], bounds[i+1]
        t_points.append(t_start)
        t_points.append((t_start + t_end) / 2.0)
    t_points.append(bounds[-1])
    t_points = sorted(set(t_points))

    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_opacity = 0.0
    transmittance = 1.0

    for i in range(len(t_points) - 1):
        t_start = t_points[i]
        t_end = t_points[i+1]
        if t_end <= t_start:
            continue
        t_mid = (t_start + t_end) / 2.0
        delta_t = t_end - t_start

        # 2. Field Evaluation at midpoint
        density = 0.0
        weighted_color = [0.0, 0.0, 0.0]
        point = (ray_origin[0] + ray_direction[0]*t_mid, 
                 ray_origin[1] + ray_direction[1]*t_mid, 
                 ray_origin[2] + ray_direction[2]*t_mid)

        for g in gaussians:
            diff = (point[0] - g['center'][0], point[1] - g['center'][1], point[2] - g['center'][2])
            # Mahalanobis distance squared: d^2 = diff^T Sigma^-1 diff
            dist_sq = 0.0
            for r_idx in range(3):
                row_sum = 0.0
                for c_idx in range(3):
                    row_sum += g['cov_inv'][r_idx][c_idx] * diff[c_idx]
                dist_sq += diff[r_idx] * row_sum
                
            # Truncate if outside support radius (e.g., 3 sigma -> dist_sq > 9)
            if dist_sq > 9.0:
                continue
                
            weight = g['opacity'] * math.exp(-0.5 * dist_sq)
            density += weight
            weighted_color[0] += weight * g['color'][0]
            weighted_color[1] += weight * g['color'][1]
            weighted_color[2] += weight * g['color'][2]

        if density < 1e-6:
            continue

        # Normalize color
        color = [c / density for c in weighted_color]

        # 3. Integration
        alpha = 1.0 - math.exp(-density * delta_t)
        accumulated_color[0] += transmittance * alpha * color[0]
        accumulated_color[1] += transmittance * alpha * color[1]
        accumulated_color[2] += transmittance * alpha * color[2]
        accumulated_opacity += transmittance * alpha
        transmittance *= (1.0 - alpha)

        if transmittance < 1e-4:
            break

    return accumulated_color, accumulated_opacity
```

## Subtopics

### [OTS Marching Ray Marching Algorithm](1073-ots-marching-ray-marching-algorithm.md)
The OTS Marching Ray Marching Algorithm is a volumetric rendering method for 3D Gaussians that evaluates the full Gaussian mixture to accurately handle primitive overlap and compute the volumetric integral using an extinction-based model. It distinguishes itself from splatting-based approaches by performing adaptive binning and analytical integration along rays, ensuring that overlapping Gaussians are resolved with precision governed by sample count rather than relying on rasterization approximations. Unlike raytracing solutions for Gaussians, this method explicitly accounts for primitive overlap in the integration process.

### [View-dependent color with Spherical Harmonics](1369-view-dependent-color-with-spherical-harmonics.md)
This method represents the view-dependent color (radiance) of a 3D Gaussian primitive as a function of the viewing direction using spherical harmonics (SH) coefficients. It solves the problem of storing directional color variation (such as highlights and reflections) compactly without requiring per-ray neural network queries. Unlike simple point colors or view-independent RGB values, this model captures specular effects and directional shading. It is distinct from its siblings in the tree: [Ray-marching on 3D Gaussians](1036-ray-marching-on-3d-gaussians.md) uses this representation for color evaluation during rasterization, whereas [OTS Marching Ray Marching Algorithm](1073-ots-marching-ray-marching-algorithm.md) and [Physics-based Gaussian Simulation](1457-physics-based-gaussian-simulation.md) do not inherently rely on this specific view-dependent color encoding.

### [Physics-based Gaussian Simulation](1457-physics-based-gaussian-simulation.md)
Physics-based Gaussian Simulation integrates continuum mechanics or particle dynamics directly into 3D Gaussian Splatting (3DGS), treating each Gaussian kernel as a discrete physical particle with volume. This approach creates a unified simulation-to-rendering pipeline where the geometric representation serves as the simulation mesh, enabling physically accurate dynamics, real-time interaction, and editing of high-fidelity virtual content. Unlike standard 3DGS which optimizes static appearance, these methods decouple or tightly couple physical state (position, velocity, deformation) from radiometric properties to support dynamic scenes.

### [Weighted Blending (Fuzzy Metaballs)](3691-weighted-blending-fuzzy-metaballs.md)
Weighted Blending is a sort-free, image-order rendering technique for 3D Gaussians that resolves the hidden-surface problem by computing a weighted average of all ray-Gaussian intersection depths. Unlike NeRF-style methods that sort intersections and use closer Gaussians to attenuate further ones, this method blends all intersections simultaneously without sorting, extending Order Independent Transparency (OIT) principles to 3D opaque surfaces.

### [Two-Stage Optimization Pipeline](15284-two-stage-optimization-pipeline.md)
The Two-Stage Optimization Pipeline is a post-processing refinement method for 3D Gaussian Splatting (3DGS) that decouples geometric alignment from material/lighting optimization to improve reconstruction quality and enable relighting. It addresses the limitations of coarse geometry from initial regressors and accumulated pose errors by first correcting camera poses and refining geometry, then locking geometry to optimize material and lighting parameters using pre-computed visibility. It differs from standard 3DGS by introducing explicit normal vectors and a visibility pre-computation step, and from pose-free methods by using a dedicated pose correction stage.

### 3DGS Marcher
A ray marching-based renderer that follows the 3DGS model of opacity, reusing the data structures (tensors with weights, positions, scales, and rotations), activation functions, and learning rates of 3DGS [S1]. Differs from OTS Marcher in the model of opacity/extinction used, but shares the same ray marching pipeline structure (per-primitive pass, tiling, sorting, per-pixel kernel) [S1].

### Condor
A custom ray tracing solution for physically accurate volumetric integration of 3D Gaussians, potentially including scattering [S2]. Uses icosahedrons to bound Gaussians [S2]. Proposed an alternative kernel function to increase compactness [S2].

### Stochastic Subsampling for Pretrained 3D Gaussians
A proposal by Sun et al. to accelerate the ray tracing of pretrained 3D Gaussians using stochastic subsampling [S2]. Does not offer a backward pass [S2]. Reaches frame rates about 3× below 3DGS [S2].

## Operating characteristics
*   **Time Complexity:** Naive evaluation scales linearly with the number of Gaussians and samples per ray. Spatial acceleration structures can reduce active evaluations per ray [S1, S2].
*   **Memory Complexity:** $O(N)$ to store Gaussian parameters [S1].
*   **Performance:** Significantly slower than rasterization-based 3DGS due to the per-ray evaluation and lack of GPU-friendly parallelism in early implementations [S1]. Stochastic subsampling can improve speed but still trails 3DGS by ~3× [S2].
*   **Strengths:** Accurately handles overlapping Gaussians and self-attenuation, providing a ground-truth reference for 3DGS approximations [S1].
*   **Limitations:** High computational cost makes real-time rendering challenging without significant optimization or hardware acceleration [S1, S2].

## Background
The method was introduced to systematically analyze the approximations made in 3D Gaussian Splatting (3DGS), specifically regarding spatial overlap and sorting [S1]. It was developed as part of the work "Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?" [S1]. The study demonstrated that while 3DGS approximations have negligible impact on still images with many Gaussians [S1]. Subsequent works like Condor have explored ray-tracing variants for performance and physical accuracy [S2].

## Sources
[S1] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
[S2] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
