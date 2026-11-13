# Dual Rendering Equation Optimization

Also called: Deformable Radiance Field Optimization, Volume Rendering and Optimization, Volume and Surface Rendering Loss

## Overview
Dual Rendering Equation Optimization (DREO) is a differentiable volume rendering method that jointly optimizes scene geometry and appearance by minimizing a combined loss derived from two distinct rendering equations: one for volumetric density accumulation and one for surface reflectance [S1, unofficial]. It solves the problem of reconstructing scenes with complex lighting and geometry from limited views by leveraging both volume and surface cues simultaneously [S1, unofficial]. Unlike standard NeRF which relies solely on volumetric integration, DREO explicitly models surface normals and BRDF properties, allowing for more accurate relighting and geometry recovery [S1, unofficial]. It differs from [Deformable and Dynamic Fields](11311-deformable-and-dynamic-fields.md) by focusing on the dual-equation loss formulation rather than temporal deformation fields, and from [Surface-SDF Representation](10778-surface-sdf-representation.md) by using a neural radiance field tensor optimized via volume rendering integrals rather than explicit signed distance functions.

## How it works
The method optimizes a scene tensor (representing density, color, and BRDF parameters) by alternating or jointly minimizing losses from two rendering passes.

### Inputs and Outputs
- **Inputs**: Camera poses, input images $C_{gt}$, and optionally ground truth normals $n_{gt}$ [S1, unofficial].
- **Outputs**: Optimized scene tensor parameters, rendered colors, and surface normals [S1, unofficial].

### Data Structures
The scene is represented as a tensor field. Specific internal layouts (e.g., voxel grid vs. MLP) are not detailed in the primary source [S1, unofficial], but the optimization targets a "scene tensor" that encodes density $\sigma$, color $c$, and BRDF values [S1, unofficial].

### Numbered Steps
1. **Volume Rendering Pass**:
   - Emit a ray $r(t) = o + td$ from origin $o$ in direction $d$ [S1, unofficial].
   - Sample $N$ points along the ray [S1, unofficial].
   - Compute density-weighted color accumulation using the volume rendering equation [S1, unofficial]:
     $$ C_{vl}(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) c_i, \quad T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j) $$
     where $\delta_i$ is the distance between samples [S1, unofficial].
2. **Surface Rendering Pass**:
   - For surface points $x$, compute the reflected radiance $L(x, d)$ [S1, unofficial].
   - Approximate the surface integral of the reflectance equation using the incident light field $L_i(x, d)$ and normal $n$ [S1, unofficial]:
     $$ L(x, d) = bval \cdot L_i(x, d) (n \cdot d) $$
     where $bval$ is the BRDF value decoded by a decoder $B_{dec}$ [S1, unofficial].
3. **Loss Evaluation**:
   - Compute the total loss combining surface rendering, volume rendering, and normal consistency [S1, unofficial]:
     $$ loss = \lambda_1 || L(x, d) - C_{gt} ||_2^2 + \lambda_2 || C_{vl}(r) - C_{gt} ||_2^2 + \lambda_3 || n - n_{gt} ||_2^2 $$
     where $\lambda_1, \lambda_2, \lambda_3$ are loss weights [S1, unofficial].
4. **Optimization**:
   - Update the scene tensor parameters to minimize the loss [S1, unofficial].
   - **Schedule**: Set $\lambda_1 = 0$ during early iterations (before the scene surface is obtained) [S1, unofficial]. Once a threshold is reached, gradually decrease $\lambda_2$ in subsequent iterations [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize scene tensor parameters
scene_tensor = initialize_tensor()

# Hyperparameters
lambda_1 = 0.0
lambda_2 = 1.0
lambda_3 = 1.0
surface_threshold = 1000 # Example iteration count
current_iter = 0

for iteration in range(max_iterations):
    # 1. Sample rays and points
    rays = sample_rays(cameras)
    points = sample_points_along_rays(rays, N=128)
    
    # 2. Forward pass: Volume Rendering
    # Compute density, color, and transmittance for each point
    sigma, c, n = scene_tensor.query(points)
    
    # Volume Rendering Integral
    T_prev = 1.0
    C_vl = 0.0
    for i in range(len(points)):
        delta = distance(points[i], points[i+1]) if i < len(points)-1 else 0.0
        # T_i calculation
        T_i = T_prev * exp(-sigma[i] * delta)
        # Color contribution
        alpha = 1.0 - exp(-sigma[i] * delta)
        C_vl += T_i * alpha * c[i]
        T_prev = T_i
        
    # 3. Forward pass: Surface Rendering
    # Approximate surface integral using incident light field
    # Note: L_i(x, d) is precomputed or approximated ambient light
    L_i = get_incident_light_field(points)
    bval = scene_tensor.decode_brdf(points) # Decoded by B_dec
    L_surf = bval * L_i * dot(n, rays.direction)
    
    # 4. Compute Losses
    # Volume Loss
    loss_vol = mean_squared_error(C_vl, ground_truth_colors)
    
    # Surface Loss
    loss_surf = mean_squared_error(L_surf, ground_truth_colors)
    
    # Normal Loss (if ground truth normals available)
    loss_norm = mean_squared_error(n, ground_truth_normals)
    
    # Combined Loss
    total_loss = lambda_1 * loss_surf + lambda_2 * loss_vol + lambda_3 * loss_norm
    
    # 5. Backpropagation and Update
    gradients = compute_gradients(total_loss, scene_tensor)
    update_scene_tensor(scene_tensor, gradients)
    
    # 6. Schedule Adjustment
    current_iter += 1
    if current_iter > surface_threshold:
        lambda_1 = 1.0 # Enable surface loss
        lambda_2 = lambda_2 * decay_factor # Gradually decrease volume loss weight
        
    if current_iter % checkpoint_interval == 0:
        save_checkpoint(scene_tensor)
```

### Parameters
- $\lambda_1, \lambda_2, \lambda_3$: Loss weights for surface rendering, volume rendering, and normal consistency, respectively [S1, unofficial].
- $bval$: BRDF value decoded by the BRDF decoder $B_{dec}$ [S1, unofficial].
- $T_i$: Discretization calculation of transmittance [S1, unofficial].

### Edge Cases and Pitfalls
- **Early Training**: Surface loss ($\lambda_1$) must be zero initially because the scene surface is not yet well-defined; enabling it too early leads to poor convergence [S1, unofficial].
- **Lighting Approximation**: The surface integral is approximated using an incident light field, which may introduce errors if the lighting is complex or dynamic [S1, unofficial].
- **BRDF Decoder**: The choice of BRDF model (e.g., simplified Disney principle-based) depends on the scene dataset characteristics [S1, unofficial].

## Subtopics

### [Texture Optimization for Volume Synthesis](12807-texture-optimization-for-volume-synthesis.md)
This method synthesizes a 3D volumetric texture (specifically cloud volumes) from a single input photograph by minimizing an energy function that measures the distance between the input image pixel blocks and the rendered voxel blocks. It adapts the patch-based texture synthesis technique of Kwatra et al. to the 3D volume domain, treating voxel blocks as the fundamental units of synthesis. Unlike standard volume rendering which uses fixed parameters, this method optimizes the voxel values themselves to match visual evidence. It is distinguished from its parent [Dual Rendering Equation Optimization](8167-dual-rendering-equation-optimization.md) by explicitly solving for the volume geometry/appearance via an Expectation-Maximization (EM) framework rather than optimizing only lighting parameters.

### Dual Rendering for Multi-View Optimization
This variant renders Gaussians twice using a differentiable rasterizer: once for the C-ToF view to compute the quad loss, and once for the RGB view to compute the color loss [S3]. It enables joint optimization of depth and color from multi-view data, leveraging the complementary information from both modalities [S3].

### Heterogeneous Volume Reconstruction with DRT
This variant processes full images instead of randomly sampled subsets for reconstruction, using 8-10 reference views per task [S4]. It employs recursive control variates to reduce primal rendering overhead, requiring ~4-8 times fewer samples than plain DRT to reach similar quality, while the adjoint phase runtime remains identical [S4]. Control variates alleviate the need for large sample counts required by L1 loss to mitigate loss function bias [S4].

## Operating characteristics
- **Time Complexity**: Not explicitly given, but DREO involves two rendering passes (volume and surface) per iteration, potentially doubling the computational cost compared to single-equation methods [S1, unofficial].
- **Memory Complexity**: Depends on the scene tensor representation; not explicitly given [S1, unofficial].
- **Performance**: The dual-equation approach allows for more accurate geometry and appearance reconstruction, especially in scenes with complex lighting and surfaces [S1, unofficial].
- **Strengths**: Joint optimization of volume and surface cues leads to better reconstruction quality; explicit BRDF modeling enables relighting [S1, unofficial].
- **Limitations**: Requires careful scheduling of loss weights; approximation of surface integral may introduce errors; computationally more expensive due to dual passes [S1, unofficial].
- **When to Use**: Scenes with complex geometry and lighting where both volume and surface information are important [S1, unofficial].
- **When Not to Use**: Simple scenes where volumetric rendering alone suffices; real-time applications where computational cost is critical [S1, unofficial].

## Background
Dual Rendering Equation Optimization was introduced in the paper "STensoRRF: Tensorial Able Relighting Radiance Field for Single-Tensor Scene Representation" [S1, unofficial]. The method builds on the principles of neural radiance fields (NeRF) and differentiable rendering, extending them to include surface reflectance modeling [S1, unofficial]. It addresses the limitations of standard NeRF in handling complex lighting and geometry by incorporating a surface rendering equation alongside the volume rendering equation [S1, unofficial].

## Sources
[S1] STensoRRF:Tensorial able Relighting Radiance Field for Single-Tensor Scene Representation, https://doi.org/10.20944/preprints202401.1085.v1, fetched 2026-10-30 (unofficial)
[S2] Neural Rendering for Stereo 3D Reconstruction of Deformable Tissues in Robotic Surgery, https://doi.org/10.1007/978-3-031-16449-1_41, fetched 2026-10-26
[S3] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S4] Recursive Control Variates for Inverse Rendering, https://research.nvidia.com/labs/rtr/publication/nicolet2023recursive/, fetched 2026-11-08
