# Multi-Stage Albedo Optimization


## Overview
Multi-stage albedo optimization is a training procedure for differentiable volume rendering and radiance fields that optimizes the albedo texture at progressively higher resolutions [S1]. It addresses uneven convergence caused by uneven observation density in inverse rendering [S1]. It sets itself apart from single-stage optimization by explicitly managing the resolution of the albedo representation during the training loop [S1].

## How it works
The method optimizes an albedo texture through a sequence of resolution stages. The core inputs are the scene parameters, camera poses, and target images. The primary output is a high-resolution albedo texture and the optimized scene parameters.

### Data Structures
1. **Albedo Texture:** A 2D RGB texture initialized at a base resolution of $1024 \times 1024$ and refined to a final resolution of $4096 \times 4096$ [S1].

### Steps
1. **Initialization:** Initialize the albedo texture $\mathbf{A}$ at the base resolution $R_0 = 1024 \times 1024$ [S1].
2. **Low-Resolution Optimization:** Run the differentiable rendering loop to minimize the pixel-wise mean squared error (MSE) between rendered images $\hat{C}$ and target images $C$ [S1].
   $$ \mathcal{L} = \frac{1}{N} \sum_{i=1}^N \| \hat{C}_i - C_i \|^2 $$
3. **Sparsification:** After the first stage, round to zero the contributions of emitters that contribute a small fraction of the total scene radiance [S1].
4. **Resolution Upgrade:** Refine the albedo texture in two subsequent stages to reach the final resolution of $4096 \times 4096$ [S1].
5. **Refinement:** Continue optimization at each new resolution. Any subset of steps can be repeated by restarting the optimization from the previous step's results, though a single run through the sequence yields good convergence [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_stage_albedo_optimization(scene_params, target_images, camera_poses, resolutions):
    # Initialize albedo at base resolution 1024x1024 [S1]
    albedo = zeros(resolutions[0], resolutions[0], 3)
    
    for i, res in enumerate(resolutions):
        if i > 0:
            # Upsample albedo to next resolution [S1]
            # Source does not specify interpolation method [G]
            albedo = interpolate(albedo, res)
        
        # Optimization loop for current resolution [S1]
        for step in range(num_steps):
            # Render current scene with current albedo [S1]
            rendered = differentiable_render(scene_params, albedo, camera_poses)
            
            # Compute pixel-wise mean squared error [S1]
            loss = mean_squared_error(rendered, target_images)
            
            # Backpropagate and update [S1]
            optimizer_albedo.zero_grad()
            optimizer_scene.zero_grad()
            loss.backward()
            optimizer_albedo.step()
            optimizer_scene.step()
        
        # Sparsify emitters after first stage [S1]
        if i == 1:
            # Round emitters contributing a small fraction of total scene radiance to zero [S1]
            albedo = sparsify_low_contributors(albedo, scene_params)
            
    return albedo, scene_params
```

### Parameters
- **Resolutions:** Sequence of resolutions for albedo texture (e.g., $[1024, 2048, 4096]$) [S1].
- **Loss Function:** Pixel-wise mean squared error [S1].
- **Sparsification Threshold:** Fraction of total radiance below which emitters are zeroed. Not given in the sources.

### Edge Cases and Pitfalls
- **Blurriness:** Low-resolution optimization can lead to blurry albedo if not refined properly. High-frequency details may be captured by the shading field instead [S2].
- **Non-linearity:** If using diffusion-based albedo extraction as a prior, non-linear outputs and gamma mismatches can cause color shifts [S2].
- **Uneven Convergence:** Without staged optimization, uneven observation density can lead to artifacts [S1].

## Subtopics

### Intrinsic Decomposition with DiffusionRenderer
A pipeline that uses DiffusionRenderer to extract initial albedo maps from input images, which are then used to reconstruct intrinsic fields (albedo, shading, residual) in a 3D Gaussian Splatting framework [S2]. It differs from standard multi-stage optimization by relying on a pre-trained diffusion model for initial albedo estimation rather than optimizing albedo from scratch. It is chosen when high-quality initial albedo priors are available from 2D diffusion models. Pitfalls include blurry predicted albedo maps due to the diffusion model's nature, which can cause high-frequency details to be reconstructed by the shading field instead [S2]. Additionally, DiffusionRenderer produces non-linear outputs; while gamma inversion is applied, color shifts may remain [S2]. The method requires an additional render call per reconstructed layer (two for diffuse, three with residual), yielding slower rendering and longer training times [S2].

## Operating characteristics
- **Time Complexity:** Increases with the number of stages and resolutions, as each stage requires a full optimization loop [S1]. The method requires additional render calls per reconstructed layer, leading to longer training times [S2].
- **Memory Complexity:** Dominated by the albedo texture size (e.g., $4096 \times 4096$ RGB) [S1].
- **Performance:** Improves convergence stability in scenes with uneven observation density [S1]. Can lead to longer training times due to multiple render calls per stage [S2].
- **Strengths:** Prevents texture blurring, stabilizes optimization, handles uneven data density.
- **Limitations:** Requires careful tuning of resolution steps and learning rates. Dependent on the quality of initial albedo if using diffusion priors [S2].

## Background
The method originates from texture-space differentiable rendering for material and lighting reconstruction [S1]. It was developed to handle complex indoor scenes captured with handheld video, where observation density is uneven [S1]. The approach builds on the principle that optimizing albedo at multiple resolutions helps converge to a stable solution [S1]. It has been adapted in subsequent works for intrinsic decomposition of 3D Gaussian Splatting, often combined with diffusion-based priors [S2].

## Sources
[S1] Material and Lighting Reconstruction for Complex Indoor Scenes with Texture-space Differentiable Rendering, https://doi.org/10.2312/sr.20211292, fetched 2026-10-29
[S2] Intrinsic decomposition and editing of 3D Gaussian splats, https://doi.org/10.1145/3804495, fetched 2026-10-27
