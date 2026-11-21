# Multi-Stage Albedo Optimization

Also called: Multi-stage albedo refinement, staged albedo optimization

## Overview
Multi-stage albedo optimization is a training procedure for differentiable volume rendering and radiance fields that decouples the optimization of surface color (albedo) from other scene parameters like geometry, density, or lighting. It addresses the problem of uneven convergence and texture blurring in inverse rendering by optimizing albedo at a low resolution first, then progressively refining it at higher resolutions. This approach stabilizes training when observation density is uneven, preventing high-frequency details from being incorrectly assigned to shading or geometry fields [S1, S2]. It sets itself apart from single-stage optimization by explicitly managing the resolution of the albedo representation during the training loop [S1].

## How it works
The method optimizes an albedo texture through a sequence of resolution stages. The core inputs are the scene geometry (or primitives), camera poses, and target images. The primary output is a high-resolution albedo texture and the optimized scene parameters.

### Data Structures
1.  **Albedo Texture:** A 2D RGB texture. The method initializes this at a base resolution (e.g., $1024 \times 1024$) and upsamples it for subsequent stages.
2.  **Scene Parameters:** Density $\sigma$, position, and other geometric attributes, optimized alongside or after albedo stages.

### Steps
1.  **Initialization:** Initialize the albedo texture $\mathbf{A}$ at the base resolution $R_0$ (e.g., $1024 \times 1024$). Initialize other scene parameters (geometry, lighting) to default or coarse estimates.
2.  **Low-Resolution Optimization:** Run the differentiable rendering loop to minimize the pixel-wise mean squared error (MSE) between rendered images $\hat{C}$ and target images $C$. Optimize $\mathbf{A}$ at resolution $R_0$ and other scene parameters.
    $$ \mathcal{L} = \frac{1}{N} \sum_{i=1}^N \| \hat{C}_i - C_i \|^2 $$
3.  **Resolution Upgrade:** Upsample the albedo texture $\mathbf{A}$ to the next resolution $R_{k+1}$ (e.g., $2048 \times 2048$, then $4096 \times 4096$). This is typically done via bilinear or nearest-neighbor interpolation followed by re-initialization of high-frequency details as zeros or small noise.
4.  **Refinement:** Continue optimization at the new resolution $R_{k+1}$. Optimize $\mathbf{A}$ at $R_{k+1}$ while keeping other parameters fixed or jointly optimizing them with a lower learning rate for albedo.
5.  **Sparsification (Optional):** After the first stage (or intermediate stages), identify emitters or primitives contributing a negligible fraction of the total scene radiance. Round their contribution to zero to reduce noise and computational load [S1].
6.  **Final Stage:** Repeat steps 3-5 until the target resolution (e.g., $4096 \times 4096$) is reached.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_stage_albedo_optimization(scene_params, target_images, camera_poses, 
                                    resolutions=[1024, 2048, 4096], 
                                    optimizer_albedo, optimizer_scene):
    # Initialize albedo at base resolution
    albedo_texture = initialize_texture(resolutions[0])
    
    for i, res in enumerate(resolutions):
        if i > 0:
            # Upsample albedo from previous resolution
            albedo_texture = upsample_texture(albedo_texture, res)
            # Re-initialize high-frequency components if necessary [G]
            # This step ensures the optimizer can learn new details
        
        # Optional: Sparsify emitters after first stage [S1]
        if i == 1:
            albedo_texture = sparsify_low_contributors(albedo_texture, scene_params)
            
        # Optimization loop for current resolution
        for step in range(num_steps_per_stage):
            # Render current scene with current albedo
            rendered_images = differentiable_render(scene_params, albedo_texture, camera_poses)
            
            # Compute loss
            loss = mean_squared_error(rendered_images, target_images)
            
            # Backpropagate and update
            # Update albedo with higher priority to stabilize texture [G]
            optimizer_albedo.zero_grad()
            optimizer_scene.zero_grad()
            
            loss.backward()
            
            # Update parameters
            optimizer_albedo.step()
            optimizer_scene.step()
            
    return albedo_texture, scene_params
```

### Parameters
-   **Resolutions:** Sequence of resolutions for albedo texture (e.g., $[1024, 2048, 4096]$) [S1].
-   **Learning Rates:** Typically, albedo may require different learning rates than geometry to prevent blurring [G].
-   **Sparsification Threshold:** Fraction of total radiance below which emitters are zeroed [S1]. Not given in the sources.

### Edge Cases and Pitfalls
-   **Blurriness:** Low-resolution optimization can lead to blurry albedo if not refined properly. High-frequency details may be captured by the shading field instead [S2].
-   **Non-linearity:** If using diffusion-based albedo extraction as a prior, non-linear outputs and gamma mismatches can cause color shifts [S2].
-   **Uneven Convergence:** Without staged optimization, uneven observation density can lead to artifacts [S1].

## Subtopics

### Intrinsic Decomposition with DiffusionRenderer
A pipeline that uses DiffusionRenderer to extract initial albedo maps from input images, which are then used to reconstruct intrinsic fields (albedo, shading, residual) in a 3D Gaussian Splatting framework [S2]. It differs from standard multi-stage optimization by relying on a pre-trained diffusion model for initial albedo estimation rather than optimizing albedo from scratch. It is chosen when high-quality initial albedo priors are available from 2D diffusion models. Pitfalls include blurry predicted albedo maps due to the diffusion model's nature, which can cause high-frequency details to be reconstructed by the shading field instead [S2]. Additionally, DiffusionRenderer produces non-linear outputs; while gamma inversion is applied, color shifts may remain [S2].

## Operating characteristics
-   **Time Complexity:** Increases with the number of stages and resolutions. Each stage requires a full optimization loop. The total time is proportional to $\sum_{k} T_k$, where $T_k$ is the time per stage at resolution $R_k$ [S1, S2].
-   **Memory Complexity:** Dominated by the albedo texture size. A $4096 \times 4096$ RGB texture requires significant VRAM. The method does not inherently reduce memory compared to single-stage high-res optimization, but it may allow for lower memory usage in early stages [S1].
-   **Performance:** Improves convergence stability in scenes with uneven observation density [S1]. Can lead to longer training times due to multiple render calls per stage [S2].
-   **Strengths:** Prevents texture blurring, stabilizes optimization, handles uneven data density.
-   **Limitations:** Requires careful tuning of resolution steps and learning rates. Dependent on the quality of initial albedo if using diffusion priors [S2].

## Background
The method originates from texture-space differentiable rendering for material and lighting reconstruction [S1]. It was developed to handle complex indoor scenes captured with handheld video, where observation density is uneven. The approach builds on the principle that optimizing albedo at multiple resolutions helps converge to a stable solution [S1]. It has been adapted in subsequent works for intrinsic decomposition of 3D Gaussian Splatting, often combined with diffusion-based priors [S2].

## Sources
[S1] Material and Lighting Reconstruction for Complex Indoor Scenes with Texture-space Differentiable Rendering, https://doi.org/10.2312/sr.20211292, fetched 2026-10-29
[S2] Intrinsic decomposition and editing of 3D Gaussian splats, https://doi.org/10.1145/3804495, fetched 2026-10-27
