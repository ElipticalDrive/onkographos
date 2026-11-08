# Ray-marching on 3D Gaussians

Also called: 3D Gaussian Ray Marching, Gaussian Volume Integration

## Overview
Ray-marching on 3D Gaussians is a volumetric rendering technique that evaluates the integral of a 3D Gaussian mixture along camera rays to accurately resolve spatial overlap and self-attenuation, which are approximated in standard rasterization-based 3D Gaussian Splatting (3DGS) [S1]. It distinguishes itself from sibling methods like [Terrain Raymarching](1357-terrain-raymarching.md) by operating on a continuous probabilistic density field rather than a discrete Signed Distance Function (SDF), and from [Ray-marching with Neural SDF](2409-ray-marching-with-neural-sdf.md) by using explicit parametric primitives instead of neural networks [S1].

## How it works
**Inputs and Outputs**
*   **Input**: A set of $N$ 3D Gaussians, each defined by a mean position $\boldsymbol{\mu}_i \in \mathbb{R}^3$, a covariance matrix $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$ (or rotation $\mathbf{R}_i$ and scale $\mathbf{S}_i$), an opacity weight $\alpha_i$, and view-dependent color coefficients (e.g., Spherical Harmonics) [S1].
*   **Output**: A color value $C$ and accumulated opacity $\alpha$ for each pixel.

**Data Structures**
The method reuses the standard 3DGS data structures:
*   **Positions**: Tensor of shape $(N, 3)$, type float32.
*   **Scales**: Tensor of shape $(N, 3)$, type float32.
*   **Rotations**: Tensor of shape $(N, 4)$, quaternion type float32.
*   **Opacities**: Tensor of shape $(N, 1)$, type float32 (often sigmoid-activated).
*   **Colors**: Tensor of shape $(N, C \cdot K)$, where $C$ is color channels and $K$ is SH degree coefficients [S1].

**Algorithm Steps**
1.  **Ray Definition**: For each pixel, define a ray $r(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the camera origin and $\mathbf{d}$ is the normalized direction vector [G].
2.  **Initialization**: Initialize accumulated color $C_{acc} = \mathbf{0}$, accumulated opacity $\alpha_{acc} = 0$, and current ray distance $t = t_{near}$ [S1, G].
3.  **Adaptive Sampling Loop**:
    *   Determine the step size $\Delta t$. Unlike SDF sphere tracing which uses the distance to the surface, Gaussian ray marching uses adaptive sampling based on the local density or a fixed small step to resolve overlap [S1].
    *   Evaluate the current position $\mathbf{p} = r(t)$.
    *   **Gaussian Evaluation**: For each Gaussian $i$, compute the Mahalanobis distance squared:
        $$ d_i^2 = (\mathbf{p} - \boldsymbol{\mu}_i)^T \boldsymbol{\Sigma}_i^{-1} (\mathbf{p} - \boldsymbol{\mu}_i) $$
        where $\boldsymbol{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^T \mathbf{R}_i^T$ [G].
    *   Compute the Gaussian density contribution:
        $$ g_i(\mathbf{p}) = \alpha_i \exp\left(-\frac{1}{2} d_i^2\right) $$
    *   **Integration**: Update the accumulated transmittance and color. The exact formulation depends on the variant (see Subtopics). A common extinction-based model updates transmittance $T$ as:
        $$ T_{new} = T_{old} \exp\left(-\sum_i g_i(\mathbf{p}) \Delta t\right) $$
        $$ C_{acc} = C_{acc} + T_{new} \left(1 - \exp\left(-\sum_i g_i(\mathbf{p}) \Delta t\right)\right) \bar{c}(\mathbf{p}) $$
        where $\bar{c}(\mathbf{p})$ is the weighted average color at $\mathbf{p}$ [S1].
    *   Advance $t \leftarrow t + \Delta t$.
    *   **Termination**: Stop if $t > t_{far}$ or $T < \epsilon$ (e.g., $10^{-4}$) [G].
4.  **Shading**: The final pixel color is $C_{acc}$. The background color is added if $T > 0$ at termination [G].

**Parameters**
*   **Step Size**: Not explicitly defined in sources; adaptive strategies are implied [S1].
*   **Thresholds**: $\epsilon$ for transmittance termination [G].

**Edge Cases**
*   **Precision**: High precision is required to resolve overlapping Gaussians accurately [S1].
*   **Degenerate Input**: Singular covariance matrices should be avoided; standard 3DGS initialization prevents this [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_gaussians(ray_origin, ray_direction, gaussians, t_near, t_far, max_steps):
    """
    Ray marches through a set of 3D Gaussians using an extinction-based model.
    """
    accumulated_color = np.array([0.0, 0.0, 0.0])
    transmittance = 1.0
    t = t_near
    
    # Precompute inverse covariances for efficiency
    inv_covs = [np.linalg.inv(g['covariance']) for g in gaussians]
    
    for step in range(max_steps):
        if transmittance < 1e-4 or t > t_far:
            break
            
        # Adaptive step size logic would go here [S1]
        delta_t = 0.01 # Placeholder for adaptive step [G]
        t += delta_t
        
        if t > t_far:
            break
            
        p = ray_origin + ray_direction * t
        
        # Evaluate all Gaussians at point p
        density_sum = 0.0
        color_sum = np.array([0.0, 0.0, 0.0])
        weight_sum = 0.0
        
        for i, g in enumerate(gaussians):
            diff = p - g['mean']
            mahalanobis_sq = np.dot(diff, np.dot(inv_covs[i], diff))
            
            # Gaussian density
            density = g['opacity'] * np.exp(-0.5 * mahalanobis_sq)
            
            # Accumulate density for extinction
            density_sum += density
            
            # Accumulate weighted color
            # Color evaluation depends on view direction, simplified here
            c = evaluate_color(g, ray_direction) 
            color_sum += density * c
            weight_sum += density
            
        # Update transmittance and color
        # Extinction coefficient is proportional to density_sum
        extinction = density_sum * delta_t
        
        new_transmittance = transmittance * np.exp(-extinction)
        
        # Contribution of this step
        if weight_sum > 0:
            avg_color = color_sum / weight_sum
        else:
            avg_color = np.array([0.0, 0.0, 0.0])
            
        accumulated_color += new_transmittance * (1.0 - np.exp(-extinction)) * avg_color
        transmittance = new_transmittance
        
    return accumulated_color + transmittance * background_color
```

## Subtopics

### [OTS Marching Ray Marching Algorithm](1073-ots-marching-ray-marching-algorithm.md)
The OTS Marching Ray Marching Algorithm is a volumetric rendering method for 3D Gaussians that evaluates the full Gaussian mixture to accurately handle primitive overlap and compute the volumetric integral using an extinction-based model. It distinguishes itself from splatting-based approaches by performing adaptive binning and analytical integration along rays, ensuring that overlapping Gaussians are resolved with precision governed by sample count rather than relying on rasterization approximations. Unlike raytracing solutions for Gaussians, this method explicitly accounts for primitive overlap in the integration process.

### [View-dependent color with Spherical Harmonics](1369-view-dependent-color-with-spherical-harmonics.md)
View-dependent color with Spherical Harmonics (SH) is a technique used in 3D Gaussian Splatting to model the view-dependent appearance (e.g., reflections, highlights) of each Gaussian primitive by representing its color as a function of the viewing direction. Unlike static color models that store a single RGB value per Gaussian, this method stores a set of coefficients that, when evaluated against the camera-to-Gaussian direction, produce a directionally varying color. This approach solves the problem of capturing specular details and reflections in point-based rendering without requiring per-ray neural network queries, keeping the rendering path closed-form and computationally cheap. It builds on [Ray-marching on 3D Gaussians](1036-ray-marching-on-3d-gaussians.md) by providing the appearance model for the primitives being marched.

### [Physics-based Gaussian Simulation](1457-physics-based-gaussian-simulation.md)
Physics-based Gaussian Simulation is a method that integrates physics-based continuum mechanics directly within 3D Gaussian Splatting (3DGS), treating each Gaussian as a discrete physical particle with volume for direct simulation. It solves the problem of animating 3D representations by creating a unified simulation-to-rendering pipeline, allowing for physically accurate dynamics rather than relying on pre-recorded motion or simple deformation. It sets itself apart from standard 3DGS by embedding physical laws into the representation itself, enabling the Gaussians to behave like material particles.

### [Weighted Blending (Fuzzy Metaballs)](3691-weighted-blending-fuzzy-metaballs.md)
Weighted Blending is a sort-free ray-marching technique for 3D Gaussians that computes the intersection of a ray with each Gaussian independently and combines these intersections via a weighted average of their depths. Unlike NeRF-style methods that sort intersections by depth and use closer Gaussians to attenuate further ones, this method blends all intersections simultaneously, avoiding the computational cost and complexity of sorting. It builds on [Ray-marching on 3D Gaussians](1036-ray-marching-on-3d-gaussians.md) by providing a specific heuristic for combining ray-Gaussian intersections without explicit ordering.

### 3DGS Marcher
A ray marching-based renderer that follows the 3DGS model of opacity, reusing the data structures (tensors with weights, positions, scales, and rotations), activation functions, and learning rates of 3DGS [S1]. Differs from OTS Marcher in the model of opacity/extinction used, but shares the same ray marching pipeline structure (per-primitive pass, tiling, sorting, per-pixel kernel) [S1].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot S)$ per pixel, where $N$ is the number of Gaussians and $S$ is the number of samples per ray [S1]. This is significantly slower than rasterization ($O(N)$ total) but provides accurate volumetric integration.
*   **Memory Complexity**: $O(N)$ to store Gaussian parameters [S1].
*   **Performance**: The method is computationally expensive due to the per-sample evaluation of all Gaussians. However, it allows for the reuse of 3DGS data structures and learning rates [S1].
*   **Strengths**: Accurately resolves spatial overlap and self-attenuation, which are approximations in standard 3DGS [S1].
*   **Limitations**: Slow rendering speed compared to rasterization.
*   **When to use**: When accurate volumetric rendering is required, such as for scientific analysis or high-fidelity still images where overlap artifacts are visible [S1].
*   **When not to use**: For real-time interactive applications where speed is critical [S1].

## Background
The method was introduced in the paper "Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?" to systematically analyze and assess the approximations made in 3DGS, specifically regarding spatial overlap and sorting [S1]. The authors implemented this ray-marching algorithm to demonstrate that while these approximations have negligible impact on still images with a large number of Gaussians, they are significant for accurate volumetric analysis [S1].

## Sources
[S1] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
