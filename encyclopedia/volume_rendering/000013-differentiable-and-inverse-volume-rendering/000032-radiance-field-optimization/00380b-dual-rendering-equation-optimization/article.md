# Dual Rendering Equation Optimization

Also called: Deformable Radiance Field Optimization, Volume Rendering and Optimization, Volume and Surface Rendering Loss

## Overview
Dual Rendering Equation Optimization (DREO) is a differentiable volume rendering technique that jointly optimizes scene parameters by minimizing a combined loss derived from two distinct rendering models: a volumetric emission-absorption model and a surface reflectance model [S1, unofficial][S2]. Unlike standard NeRF methods that rely solely on the volume rendering equation, DREO explicitly supervises both the accumulated volume color and the surface radiance, enabling the reconstruction of scenes with complex surface properties and deformations [S1, unofficial][S2]. It sets itself apart from sibling methods like [NeRF++](6198-nerf.md) or [Surface-SDF Representation](10778-surface-sdf-representation.md) by treating surface and volume rendering as coupled optimization targets within a single unified loss function, rather than using surface representations merely as priors or constraints [S1, unofficial][S2].

## How it works
The method optimizes a scene representation (typically a neural field or tensor) by evaluating two rendering equations for each camera ray and computing gradients with respect to the scene parameters.

### Inputs and Outputs
*   **Inputs:** Camera poses, viewing directions, ground truth RGB images $C_{gt}$, and optionally ground truth depth maps $D_{gt}$ or normal maps $n_{gt}$ [S1, unofficial][S2].
*   **Outputs:** Optimized scene parameters (e.g., neural network weights $\Theta, \Phi$) and rendered images [S1, unofficial][S2].

### Data Structures
The specific data structures depend on the underlying representation (e.g., MLP weights, voxel grids). The sources describe the optimization of network parameters $\Theta$ and $\Phi$ [S2] or a scene tensor [S1, unofficial]. No specific bit-packing or layout details are provided in the sources [S1, unofficial][S2].

### Rendering Equations
The method employs two rendering equations [S1, unofficial]:

1.  **Volume Rendering Equation:** Computes the accumulated color $C_{vl}(r)$ along a ray $r(t) = o + td$ by sampling $N$ points [S1, unofficial].
    $$
    C_{vl}(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) c_i, \quad T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)
    $$
    where $\sigma_i$ is density, $\delta_i$ is step size, and $c_i$ is color [S1, unofficial].

2.  **Surface Rendering Equation:** Computes the radiance $L(x, d)$ at a surface point $x$ observed from direction $d$ [S1, unofficial]. The full surface integral is approximated using an incident light field [S1, unofficial]:
    $$
    L(x, d) = bval \cdot L_i(x, d) (n \cdot d)
    $$
    where $bval$ is the BRDF value decoded by a decoder $B_{dec}$, $L_i(x, d)$ is the ambient/incident light, and $n$ is the surface normal [S1, unofficial].

In the context of deformable fields (e.g., D-NeRF style), the rendered color $\hat{C}$ and depth $\hat{D}$ are computed as [S2]:
$$
\hat{C}(\mathbf{r}(s)) = \sum_{j=1}^{m-1} w_j c(\mathbf{x}_j, \mathbf{d}), \quad \hat{D}(\mathbf{r}(s)) = \sum_{j=1}^{m-1} w_j s_j
$$
with weights:
$$
w_j = (1 - \exp(-\sigma(\mathbf{x}_j) \Delta s_j)) \exp\left(-\sum_{k=1}^{j-1} \sigma(\mathbf{x}_k) \Delta s_k\right), \quad \Delta s_j = s_{j+1} - s_j
$$
where $s_j$ are sample depths [S2].

### Optimization Steps
The training loop iteration proceeds as follows [S2]:
1.  Randomly select a frame $i$ from the training set [S2].
2.  Perform ray casting from the camera origin $o$ in direction $d$ [S1, unofficial][S2].
3.  Sample points along the ray. In deformable contexts, this may involve depth-cueing ray marching to concentrate samples near surfaces [S2].
4.  Query the scene representation (networks $F_\Theta$ and $G_\Phi$) to obtain density $\sigma$, color $c$, and potentially normal $n$ or displacement [S1, unofficial][S2].
5.  Evaluate the volume rendering integral to compute $\hat{C}$ and $\hat{D}$ [S2] or $C_{vl}$ [S1, unofficial].
6.  Evaluate the surface rendering equation $L(x, d)$ for surface points [S1, unofficial].
7.  Compute the total loss and optimize parameters [S1, unofficial][S2].

### Loss Function
The total loss combines volume, surface, and auxiliary terms [S1, unofficial][S2]:

For surface/volume joint optimization [S1, unofficial]:
$$
\text{loss} = \lambda_1 \| L(x, d) - C_{gt} \|_2^2 + \lambda_2 \| C(r) - C_{gt} \|_2^2 + \lambda_3 \| n - n_{gt} \|_2^2
$$
where $\lambda_1, \lambda_2, \lambda_3$ are loss weights [S1, unofficial].

For deformable field optimization with depth [S2]:
$$
\mathcal{C}(\mathbf{r}(s)) = \| \hat{C}(\mathbf{r}(s)) - \mathbf{I}[u, v] \|_2^2 + \lambda \| \hat{D}(\mathbf{r}(s)) - \mathbf{D}_i[u, v] \|
$$
where $\lambda$ weights the depth loss [S2].

### Parameters and Edge Cases
*   **Loss Weights:** $\lambda_1, \lambda_2, \lambda_3$ control the balance between surface, volume, and normal consistency [S1, unofficial]. $\lambda$ controls depth supervision [S2].
*   **Annealing Strategy:** In [S1, unofficial], $\lambda_1$ (surface loss) is set to 0 initially until a scene surface is obtained, after which it is activated. $\lambda_2$ (volume loss) is gradually decreased in subsequent iterations [S1, unofficial].
*   **Depth Noise:** Corrupt stereo depth (e.g., from specular highlights) can cause artifacts. The method relies on early-stage underfitting to smooth out minority corrupt depth samples [S2].
*   **BRDF Decoder:** The surface term requires a BRDF decoder $B_{dec}$ to compute $bval$ [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dual_rendering_optimization_step(scene_params, camera, gt_image, gt_depth=None, gt_normal=None):
    """
    One iteration of Dual Rendering Equation Optimization.
    """
    # 1. Ray Casting
    rays = generate_rays(camera)
    
    # 2. Sampling
    # Depth-cueing or uniform sampling along rays
    samples = sample_points_along_rays(rays, num_samples=64)
    
    # 3. Query Scene
    # Returns density, color, normal, and potentially displacement
    densities, colors, normals, _ = query_scene(scene_params, samples)
    
    # 4. Volume Rendering
    # Compute transmittance and accumulated color
    transmittances = compute_transmittance(densities, samples)
    volume_color = compute_volume_accumulation(transmittances, densities, samples, colors)
    
    # 5. Surface Rendering (if surface points are identified)
    # Identify surface points (e.g., highest density or via SDF)
    surface_points = identify_surface(samples, densities)
    if surface_points is not None:
        # Approximate incident light field
        incident_light = compute_ambient_light(surface_points)
        # Decode BRDF value
        brdf_val = decode_brdf(scene_params, surface_points)
        # Compute surface radiance
        surface_radiance = brdf_val * incident_light * dot(surface_points.normal, rays.direction)
    else:
        surface_radiance = None
        
    # 6. Loss Computation
    loss = 0.0
    
    # Volume Loss
    # In deformable contexts, use rendered depth if available
    if gt_depth is not None:
        rendered_depth = compute_depth_accumulation(transmittances, densities, samples)
        depth_loss = lambda_depth * l2_loss(rendered_depth, gt_depth)
        loss += depth_loss
        
    volume_loss = l2_loss(volume_color, gt_image)
    loss += lambda_volume * volume_loss
    
    # Surface Loss
    if surface_radiance is not None and gt_image is not None:
        # Only apply if surface is established (lambda_1 > 0)
        if is_surface_established(current_iteration):
            surface_loss = l2_loss(surface_radiance, gt_image)
            loss += lambda_surface * surface_loss
            
    # Normal Loss
    if gt_normal is not None:
        normal_loss = l2_loss(surface_points.normal, gt_normal)
        loss += lambda_normal * normal_loss
        
    # 7. Backpropagation
    gradients = compute_gradients(loss, scene_params)
    update_scene_params(scene_params, gradients)
    
    return loss
```

## Subtopics

### [Texture Optimization for Volume Synthesis](12807-texture-optimization-for-volume-synthesis.md)
A method to synthesize a 3D cloud volume $V$ from a set of $K$ example voxel blocks $C_{ex}$ by minimizing an energy function that measures the distance between input image pixel blocks and rendered voxel blocks. It adapts the texture synthesis technique of Kwatra et al. to the volume domain, solving for voxel values and example selection simultaneously. It differs from direct volume rendering by optimizing the internal structure of the volume to match photographic input, rather than just rendering a pre-defined volume.

### Dual Rendering for Multi-View Optimization
In the context of Time of Flight (C-ToF) and RGB optimization, Gaussians are rendered twice using a differentiable rasterizer: once for the C-ToF view to compute the quad loss, and once for the RGB view to compute the color loss [S3]. This allows for joint optimization of depth and color information from heterogeneous sensors [S3].

### Heterogeneous Volume Reconstruction with DRT
This variant uses Recursive Control Variates for Inverse Rendering. It processes full images instead of randomly sampled subsets for reconstruction, using 8-10 reference views per task [S4]. Control variates reduce primal rendering overhead by alleviating the need for large sample counts required by L1 loss to mitigate loss function bias [S4]. Plain DRT without control variates requires ~4-8 times more samples to reach similar quality, with adjoint phase runtime remaining identical [S4].

## Operating characteristics
*   **Complexity:** The method requires evaluating two rendering equations per ray, effectively doubling the forward pass computation compared to standard volume rendering [S1, unofficial][S2]. The memory complexity depends on the number of samples and the storage of intermediate transmittance values for backpropagation [44-radiance-field-optimization.md].
*   **Performance:** The use of control variates in adjoint-based variants (DRT) significantly reduces primal rendering overhead and improves reconstruction quality at equal time [S4]. Without control variates, plain DRT requires 4-8 times more samples [S4].
*   **Strengths:** Enables joint optimization of surface and volume properties, handling complex materials and deformations [S1, unofficial][S2]. The dual supervision helps resolve ambiguities in depth and color [S2].
*   **Limitations:** Requires careful tuning of loss weights ($\lambda_1, \lambda_2, \lambda_3$) and annealing schedules [S1, unofficial]. Sensitive to corrupt depth data if not handled properly [S2]. Computationally more expensive than single-equation methods [S1, unofficial][S4].

## Background
The concept of using multiple rendering equations for optimization originates from the need to reconstruct scenes with both volumetric and surface characteristics [S1, unofficial]. The specific formulation combining volume rendering (NeRF-style) and surface reflectance (BRDF-based) was introduced in [S1, unofficial] (STensoRRF). The application to deformable tissues using depth supervision was detailed in [S2] (Neural Rendering for Stereo 3D Reconstruction of Deformable Tissues in Robotic Surgery). The integration with Time of Flight sensors and Gaussian Splatting was explored in [S3] (ToRF). The adjoint-based optimization with control variates for inverse rendering was presented in [S4] (Recursive Control Variates for Inverse Rendering).

## Sources
[S1] STensoRRF:Tensorial able Relighting Radiance Field for Single-Tensor Scene Representation, https://doi.org/10.20944/preprints202401.1085.v1, fetched 2026-10-30 (unofficial)
[S2] Neural Rendering for Stereo 3D Reconstruction of Deformable Tissues in Robotic Surgery, https://doi.org/10.1007/978-3-031-16449-1_41, fetched 2026-10-26
[S3] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S4] Recursive Control Variates for Inverse Rendering, https://research.nvidia.com/labs/rtr/publication/nicolet2023recursive/, fetched 2026-11-08
