# Reference-guided Controllable Inpainting of NeRFs

Also called: Reference-guided Inpainting

## Overview
This method performs 3D inpainting on a Neural Radiance Field (NeRF) using a single inpainted reference view to guide geometry and appearance, thereby avoiding view inconsistencies and generating view-dependent effects (VDEs) [S1]. It differs from multi-view inpainting approaches by relying on one user-provided image to define the target content, propagating this information to other viewpoints via depth supervision and bilateral rendering, while handling disocclusions via image inpainters [S1].

## How it works
**Inputs:**
*   $n$ input images $\{I_i\}_{i=1}^n$.
*   Camera transform matrices $\{\Pi_i\}_{i=1}^n$.
*   Masks $\{M_i\}_{i=1}^n$ delineating unwanted regions in each view.
*   A single inpainted reference view $I_r$, where $r \in \{1, \dots, N\}$ [S1].

**Outputs:**
*   A trained NeRF model $f_\theta$ representing the inpainted scene [S1].

**Data Structures:**
*   The NeRF $f_\theta$ maps 3D coordinates $\mathbf{x}$ and viewing direction $\mathbf{v}$ to density $\sigma$ and color $\mathbf{c}$ [S1].
*   Rays $r$ are defined by origin and direction. Transmittance is $T(r) = \exp(-\sum_{j=1}^k \sigma_j \delta_j)$, where $\delta_i = t_{i+1} - t_i$ [S1].
*   Depth $\hat{\zeta}(r)$ is estimated by replacing color $c_i$ with depth $t_i$ in the volume rendering equation [S1].

**Parameters:**
*   $N_{depth}$: Number of iterations for Stage 1 supervision.
*   $\gamma_{depth}, \gamma_{rec}, \gamma_{do}$: Loss weights.
*   $N_{bilateral}, N_{do}$: Iteration intervals for applying bilateral and disocclusion losses [S1].

**Steps:**

1.  **Stage 1: Unmasked Supervision.**
    Train $f_\theta$ on unmasked pixels for $N_{depth}$ iterations using the standard reconstruction loss. The masked region remains under-fit to facilitate later alterations [S1].
    $$L_{rec}^{unmasked} = \mathbb{E}_{r \sim R_{unmasked}} \| \hat{C}(r) - C_{GT}(r) \|^2$$
    where $R_{unmasked}$ is the set of rays corresponding to pixels in $\{I_i (1 - M_i)\}_{i=1}^n$ and $C_{GT}(r)$ is the ground truth color [S1].

2.  **Geometry Supervision via Monocular Depth.**
    Use a monocular depth estimator to generate a depth map for the inpainted reference view $I_r$. Back-project $I_r$ to 3D positions to compute a disparity map. Align this with the current NeRF scene to compute masked depth supervision $L_{depth}^{masked}$ [S1].

3.  **View-Dependent Effects (VDEs) via Bilateral Solver.**
    For non-reference views, use a view-substitution technique followed by bilaterally guided inpainting to obtain target colors $I_{br,t}$. These colors are used to compute the masked reconstruction loss $L_{rec}^{masked}$, ensuring VDEs are consistent with the reference [S1].

4.  **Disocclusion Handling.**
    For non-reference disoccluded regions not supervised by the single reference view, use image inpainters to guide both geometry and appearance via loss $L_{do}$ [S1].

5.  **Total Loss and Optimization.**
    The total loss is:
    $$L = L_{rec}^{unmasked} + \gamma_{depth} L_{depth}^{masked} + \gamma_{rec} L_{rec}^{masked} + \gamma_{do} L_{do}$$
    Losses $L_{depth}$, $L_{rec}$, and $L_{do}$ are computed every $N_{depth}$, $N_{bilateral}$, and $N_{do}$ iterations respectively [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reference_guided_inpainting(input_images, camera_matrices, masks, inpainted_ref_view, n_iterations):
    """
    Trains a NeRF for inpainting using a single reference view.
    """
    # Initialize NeRF parameters theta
    theta = initialize_nerf()
    
    # Stage 1: Supervise unmasked pixels
    for i in range(n_iterations):
        # Compute rays for unmasked pixels
        rays_unmasked = get_rays(input_images, masks, unmasked_only=True)
        
        # Render colors for unmasked rays
        colors_unmasked = render_nerf(theta, rays_unmasked)
        
        # Compute reconstruction loss for unmasked pixels
        gt_colors_unmasked = get_gt_colors(rays_unmasked, input_images)
        loss_unmasked = mse_loss(colors_unmasked, gt_colors_unmasked)
        
        # Backpropagate only unmasked loss for first N_depth iterations
        if i < N_depth:
            loss_unmasked.backward()
            optimizer.step()
            optimizer.zero_grad()
            continue
            
        # Stage 2: Multi-task optimization
        loss_total = loss_unmasked
        
        # Geometry Supervision
        # Back-project inpainted reference view to 3D
        depth_map_ref = monocular_depth_estimator(inpainted_ref_view)
        points_3d_ref = back_project(depth_map_ref, camera_matrices[r])
        
        # Compute masked depth loss
        rays_masked = get_rays(input_images, masks, masked_only=True)
        # Note: Specific alignment and loss computation details for L_depth are not fully specified in source [S1]
        # [G] Assuming standard depth consistency loss between rendered depth and back-projected depth
        rendered_depth = render_depth(theta, rays_masked)
        loss_depth = depth_consistency_loss(rendered_depth, points_3d_ref)
        loss_total += gamma_depth * loss_depth
        
        # View-Dependent Effects (VDEs)
        # Generate target colors for non-reference views using bilateral solver
        target_colors_non_ref = bilateral_inpainting(inpainted_ref_view, camera_matrices, masks)
        
        # Compute masked reconstruction loss using target colors
        # Note: Source mentions L_rec_masked but does not provide explicit formula beyond reference to I_br,t
        # [G] Assuming MSE between rendered color and target color for masked rays
        rendered_colors_masked = render_nerf(theta, rays_masked)
        loss_rec_masked = mse_loss(rendered_colors_masked, target_colors_non_ref)
        loss_total += gamma_rec * loss_rec_masked
        
        # Disocclusion Handling
        # Identify disoccluded regions in non-reference views
        rays_disoccluded = identify_disocclusions(rays_masked, camera_matrices)
        
        # Guide geometry and appearance via image inpainters
        # Note: Source mentions L_do but does not provide explicit formula
        # [G] Assuming loss based on image inpainter output for disoccluded regions
        disoccluded_targets = image_inpainter(input_images, masks, rays_disoccluded)
        rendered_colors_disoccluded = render_nerf(theta, rays_disoccluded)
        loss_do = mse_loss(rendered_colors_disoccluded, disoccluded_targets)
        loss_total += gamma_do * loss_do
        
        # Update parameters
        loss_total.backward()
        optimizer.step()
        optimizer.zero_grad()
        
    return theta
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The approach shows superior performance to NeRF inpainting baselines [S1].
*   **Strengths:** Avoids view inconsistencies by using a single reference; generates view-dependent effects; allows user control via a single image [S1].
*   **Limitations:** Relies on the quality of the single inpainted reference view and monocular depth estimators [S1].

## Background
The method was introduced in "Reference-guided Controllable Inpainting of Neural Radiance Fields" [S1]. It addresses the desire for NeRF editing tools that can inpaint regions in a view-consistent and controllable manner [S1].

## Sources
[S1] Reference-guided Controllable Inpainting of Neural Radiance Fields, https://doi.org/10.1109/ICCV51070.2023.01633, fetched 2026-10-29
