# Reference-guided Controllable Inpainting of Neural Radiance Fields

Also called: Ours, Ours-SD

## Overview
This method performs 3D inpainting of Neural Radiance Fields (NeRFs) by substituting a single reference view with a diffusion-generated target view, then optimizing the NeRF to match the new view while maintaining consistency with the remaining input views. It solves the problem of inserting or modifying objects in a 3D scene without requiring multi-view supervision for the target content. It differs from [Inpainting SDS Loss](11185-inpainting-sds-loss.md) by using a deterministic 2D inpainting prior (Stable Diffusion) followed by a bilateral solver for residual smoothing, rather than relying solely on gradient-based SDS guidance during optimization.

## How it works
**Inputs:**
- $N$ input images $\{I_{r,i}\}_{i=1}^N$ with known camera poses $\{P_i\}_{i=1}^N$.
- Binary masks $\{M_i\}_{i=1}^N$ indicating regions to be inpainted in each input view.
- A single reference view $I_{r,ref}$ (typically the 30th image of the 60 training views in SPIn-NeRF) with mask $M_{ref}$.
- A text prompt describing the desired content for the reference view (for Ours-SD variant).

**Outputs:**
- Optimized NeRF parameters (density $\sigma$, color $\mathbf{c}$).
- Novel view renderings $\{I_{t,j}\}$ corresponding to target camera poses $\{P_{t,j}\}$.

**Data Structures:**
- **NeRF:** Standard MLP mapping 3D position and viewing direction to density and color.
- **Masks:** Binary images, dilated to ensure full coverage of the occluded region.
- **Residuals:** Float32 images representing the difference between the original input view and the inpainted view.

**Parameters:**
- Downscale factor: 4 (all input images downsized by 4×).
- Distortion loss weight: $\gamma_{dist} = 0.01$.
- Bilateral solver brightness/color bandwidths: $\sigma_c = 4$.
- Bilateral solver spatial bandwidth: $\sigma_s = 128$.
- Bilateral solver smoothness strength: $\lambda = 128$.
- Bilateral solver PCG iterations: $K_{pcg} = 25$.
- Disocclusion handling interval: $N_{do} = 3000$ iterations.
- Depth supervision steps: $N_{depth} = 2000$.
- Bilateral solver steps: $N_{bilateral} = 2000$.
- Total training iterations: 10,000.
- Loss weights: $\gamma_{depth} = 4$, $\gamma_{rec} = 2$, $\gamma_{do} = 1$, $\eta_{do} = 0.25$, $\gamma_{smooth} = 1000$, $\gamma_{ei} = 2$.
- Mask dilation: 5 iterations with a 5×5 kernel for input masks; 4 iterations with a 3×3 kernel for disocclusion masks.

**Steps:**

1. **Preprocessing:**
   - Downscale all input images and masks by 4×.
   - Dilate input masks $M_i$ by 5 iterations using a 5×5 kernel to ensure the inpainted region fully covers the occluded area [G].
   - Dilate disocclusion masks by 4 iterations using a 3×3 kernel to remove noise [G].

2. **Reference View Inpainting:**
   - Generate the inpainted reference view $I_{t,ref}$ using Stable Diffusion Inpainting v2 [55] conditioned on $I_{r,ref}$, $M_{ref}$, and the text prompt.
   - If the generated image contains unwanted objects, regenerate until clean [G].

3. **View Substitution and Residual Calculation:**
   - For each input view $i$, compute the view-substituted image $I_{t,i}$ by replacing the masked region of $I_{r,i}$ with the corresponding region from $I_{t,ref}$ (warped via camera geometry).
   - Compute the absolute residual image $R_i = |I_{t,i} - I_{r,i}|$.
   - Apply the bilateral solver to $R_i$ to obtain a smooth residual map $\tilde{R}_i$. The bilateral solver minimizes:
     $$ \min_{\mathbf{u}} \sum_{p} \left( \lambda (\mathbf{u}_p - \mathbf{r}_p)^2 + \sigma_s^2 \sum_{q \in \mathcal{N}(p)} w_{pq} (\mathbf{u}_p - \mathbf{u}_q)^2 \right) $$
     where $\mathbf{u}$ is the output, $\mathbf{r}$ is the input residual, $w_{pq}$ is the bilateral weight based on spatial and color distance, and $\mathcal{N}(p)$ is the neighborhood. Solve using Preconditioned Conjugate Gradient (PCG) for 25 iterations [S1].

4. **Disocclusion Handling (Periodic):**
   - Every $N_{do} = 3000$ iterations, identify disoccluded regions (areas visible in target views but occluded in input views).
   - Use LaMa [61] as the 2D inpainter for these regions.
   - Use three target images (cameras furthest left, right, and upward) as context for LaMa.
   - Apply the smoothed residual $\tilde{R}_{do}$ to the disoccluded regions in the target views.

5. **NeRF Optimization:**
   - Initialize NeRF with input views.
   - For iteration $k = 1$ to 10,000:
     a. Sample rays from target camera poses.
     b. Render color $\mathbf{c}_{render}$ and depth $d_{render}$.
     c. Compute losses:
        - **Depth Loss ($\mathcal{L}_{depth}$):** Supervise depth in the inpainted region using the warped depth from the reference view. Weight $\gamma_{depth} = 4$.
        - **Reconstruction Loss ($\mathcal{L}_{rec}$):** Match rendered color to the view-substituted target view $I_{t,i}$ in the masked region. Weight $\gamma_{rec} = 2$.
        - **Disocclusion Loss ($\mathcal{L}_{do}$):** Apply if disocclusion handling is active. Weight $\gamma_{do} = 1$, scaling factor $\eta_{do} = 0.25$.
        - **Smoothness Loss ($\mathcal{L}_{smooth}$):** Penalize high-frequency artifacts in the rendered image. Weight $\gamma_{smooth} = 1000$.
        - **Distortion Loss ($\mathcal{L}_{dist}$):** Reduce floater artifacts. Weight $\gamma_{dist} = 0.01$.
        - **Edge Inpainting Loss ($\mathcal{L}_{ei}$):** Ensure smooth transitions at mask boundaries. Weight $\gamma_{ei} = 2$.
     d. Total Loss: $\mathcal{L} = \gamma_{depth}\mathcal{L}_{depth} + \gamma_{rec}\mathcal{L}_{rec} + \gamma_{do}\mathcal{L}_{do} + \gamma_{smooth}\mathcal{L}_{smooth} + \gamma_{dist}\mathcal{L}_{dist} + \gamma_{ei}\mathcal{L}_{ei}$.
     e. Backpropagate and update NeRF parameters.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reference_guided_inpainting(input_views, input_masks, reference_view, reference_mask, text_prompt, target_cameras):
    # 1. Preprocessing
    input_views_down = [downscale_4x(v) for v in input_views]
    input_masks_down = [dilate_mask(m, kernel_size=5, iterations=5) for m in input_masks]
    
    # 2. Reference Inpainting
    ref_inpainted = stable_diffusion_inpaint_v2(reference_view_down, reference_mask_down, text_prompt)
    
    # 3. View Substitution
    view_substituted_views = []
    residuals = []
    for i in range(len(input_views_down)):
        # Warp reference view to input view i's pose
        ref_warped = warp_image(ref_inpainted, reference_pose, input_poses[i])
        # Replace masked region
        substituted = blend(input_views_down[i], ref_warped, input_masks_down[i])
        view_substituted_views.append(substituted)
        # Compute residual
        res = abs(substituted - input_views_down[i])
        # Bilateral Solver
        smooth_res = bilateral_solver(res, sigma_c=4, sigma_s=128, lambda_smooth=128, pcg_iters=25)
        residuals.append(smooth_res)
        
    # 4. Optimization Loop
    nerf = initialize_nerf(input_views_down)
    for k in range(10000):
        # Sample rays from target cameras
        rays = sample_rays(target_cameras)
        
        # Render
        colors, depths = render_nerf(nerf, rays)
        
        # Compute Losses
        loss_depth = 0
        loss_rec = 0
        loss_do = 0
        loss_smooth = 0
        loss_dist = 0
        loss_ei = 0
        
        # Depth Loss (in inpainted region)
        if k < 2000:
            loss_depth = depth_loss(depths, warped_ref_depth, masks)
            
        # Reconstruction Loss (in inpainted region)
        if k < 2000:
            loss_rec = reconstruction_loss(colors, view_substituted_views, masks)
            
        # Disocclusion Loss (periodic)
        if k % 3000 == 0:
            do_masks = compute_disocclusion_masks(rays, nerf)
            loss_do = disocclusion_loss(colors, do_masks, residuals)
            
        # Smoothness Loss
        loss_smooth = smoothness_loss(colors)
        
        # Distortion Loss
        loss_dist = distortion_loss(colors, depths)
        
        # Edge Inpainting Loss
        loss_ei = edge_inpainting_loss(colors, masks)
        
        # Total Loss
        total_loss = (4 * loss_depth + 
                      2 * loss_rec + 
                      1 * loss_do + 
                      1000 * loss_smooth + 
                      0.01 * loss_dist + 
                      2 * loss_ei)
        
        # Update
        optimizer.zero_grad()
        total_loss.backward()
        optimizer.step()
        
    return nerf
```

## Operating characteristics
- **Time Complexity:** Dominated by the 10,000 iterations of NeRF optimization and the periodic 2D inpainting (LaMa/SD). The bilateral solver is applied once per view during preprocessing.
- **Memory Complexity:** Reduced by downsizing inputs by 4×. The NeRF memory footprint is standard for the resolution.
- **Performance:** Produces sharp, consistent inpaintings. The bilateral solver ensures smooth residuals, reducing artifacts. The method is robust to disocclusions due to periodic LaMa updates.
- **Strengths:** Controllable via text prompt; handles complex occlusions; sharp results compared to SDS-only methods.
- **Limitations:** Requires a single high-quality reference view; dependent on the quality of Stable Diffusion and LaMa; may introduce semantic inconsistencies if the prompt is ambiguous.

## Background
Originated in the paper "Reference-guided Controllable Inpainting of Neural Radiance Fields" (ICCV 2023) [S1]. It builds on the SPInNeRF framework [42] and leverages the diversity of diffusion models for view synthesis. The method addresses the limitations of previous NeRF inpainting techniques that lacked controllability or suffered from artifacts in disoccluded regions.

## Sources
[S1] Reference-guided Controllable Inpainting of Neural Radiance Fields, https://doi.org/10.1109/ICCV51070.2023.01633, fetched 2026-10-29
