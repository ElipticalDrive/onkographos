# Reference-guided Controllable Inpainting of NeRFs

Also called: Reference-guided Inpainting

## Overview
A 3D inpainting algorithm that uses a single inpainted reference view to guide the geometry and appearance of a NeRF, avoiding view inconsistencies and generating view-dependent effects [S1]. It sets itself apart from multi-view inpainting methods by requiring only one user-provided inpainted image, and from diffusion-based methods by using explicit geometric supervision via monocular depth estimation and bilateral solvers for view-dependent effects [S1].

## How it works
**Inputs:** $n$ input images $\{I_i\}_{i=1}^n$, camera transform matrices $\{\Pi_i\}_{i=1}^n$, masks $\{M_i\}_{i=1}^n$ delineating unwanted regions, and a single inpainted reference view $I_r$ where $r \in \{1, ..., N\}$ [S1].
**Outputs:** A trained NeRF representation $f_\theta$ capable of rendering the scene with the inpainted content consistent across all views [S1].

**Data Structures:**
*   **NeRF Model $f_\theta$:** Maps 3D coordinates $\mathbf{x}$ and viewing directions $\mathbf{d}$ to density $\sigma$ and color $\mathbf{c}$.
*   **Ray Set $R$:** A collection of rays sampled through the camera frustum.
*   **Mask $M_i$:** Binary mask for image $I_i$, where 1 indicates the region to be inpainted.

**Numbered Steps:**

1.  **Initialization:** Initialize the NeRF parameters $\theta$.
2.  **Stage 1: Unmasked Supervision:** Supervise $f_\theta$ on unmasked pixels for $N_{depth}$ iterations using the standard NeRF reconstruction loss $L_{rec}^{unmasked}$ [S1]. The masked region remains under-fit to allow altering masked values in later stages [S1].
    *   Formula:
        $$ L_{rec}^{unmasked} = \mathbb{E}_{r \sim R_{unmasked}} \| \hat{C}(r) - C_{GT}(r) \|^2 $$
        where $R_{unmasked}$ is the set of rays corresponding to pixels in $\{I_i(1 - M_i)\}_{i=1}^n$ and $C_{GT}(r)$ is the ground truth color [S1].
3.  **Stage 2: Geometry Supervision via Depth:** Use monocular depth estimators to back-project the inpainted reference view $I_r$ to 3D positions. Compute a disparity map, align it with the current inpainted NeRF scene, and use it for masked depth supervision via $L_{depth}^{masked}$ [S1].
4.  **Stage 3: View-Dependent Effects (VDEs):** Use a view-substitution technique followed by bilaterally guided inpainting to obtain target colors $I_{br,t}$ for non-reference views. These are used for $L_{rec}^{masked}$ [S1].
5.  **Stage 4: Disocclusion Handling:** For non-reference disoccluded regions not supervised by the single reference view, use image inpainters to guide both geometry and appearance via loss $L_{do}$ [S1].
6.  **Joint Optimization:** Minimize the total loss:
    $$ L = L_{rec}^{unmasked} + \gamma_{depth} L_{depth}^{masked} + \gamma_{rec} L_{rec}^{masked} + \gamma_{do} L_{do} $$
    where $\gamma$ are weighting factors [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reference_guided_inpainting(images, cameras, masks, reference_view, num_iterations):
    """
    Perform reference-guided inpainting of a NeRF.
    
    Args:
        images: List of n input images {I_i}
        cameras: List of n camera transform matrices {Pi}
        masks: List of n masks {Mi}
        reference_view: The single inpainted reference view Ir
        num_iterations: Total number of optimization iterations
        
    Returns:
        Trained NeRF model f_theta
    """
    # Initialize NeRF model
    model = initialize_nerf()
    
    # Stage 1: Supervise unmasked pixels
    for i in range(N_depth):
        rays_unmasked = sample_rays(images, cameras, masks, unmasked_only=True)
        colors_gt = get_ground_truth_colors(rays_unmasked, images)
        colors_pred = render_rays(model, rays_unmasked)
        loss_unmasked = mean_squared_error(colors_pred, colors_gt)
        loss_unmasked.backward()
        optimizer.step()
        optimizer.zero_grad()
        
    # Stage 2: Geometry Supervision
    # Use monocular depth estimator on reference view
    depth_map = monocular_depth_estimator(reference_view)
    # Back-project to 3D positions
    points_3d = back_project(depth_map, cameras[reference_view_index])
    # Compute disparity and align with NeRF
    disparity_map = compute_disparity(depth_map)
    
    for i in range(N_depth, num_iterations):
        # Sample rays including masked regions
        rays_all = sample_rays(images, cameras, masks, unmasked_only=False)
        
        # Render current NeRF
        colors_pred = render_rays(model, rays_all)
        depths_pred = render_depth(model, rays_all)
        
        # Compute losses
        # 1. Unmasked reconstruction loss
        rays_unmasked = [r for r in rays_all if not r.masked]
        colors_gt_unmasked = get_ground_truth_colors(rays_unmasked, images)
        loss_rec_unmasked = mean_squared_error(
            [c for c in colors_pred if not r.masked], 
            colors_gt_unmasked
        )
        
        # 2. Masked depth loss (using reference view geometry)
        # Align predicted depth with reference depth in masked regions
        loss_depth_masked = compute_depth_loss(depths_pred, disparity_map, masks)
        
        # 3. Masked reconstruction loss (using VDEs)
        # Generate target colors for non-reference views using bilateral solver
        target_colors = generate_vde_colors(reference_view, cameras, masks)
        loss_rec_masked = compute_masked_reconstruction_loss(
            colors_pred, target_colors, masks
        )
        
        # 4. Disocclusion loss
        # Handle disoccluded regions using image inpainters
        loss_do = compute_disocclusion_loss(model, rays_all, masks)
        
        # Total loss
        total_loss = (loss_rec_unmasked + 
                      gamma_depth * loss_depth_masked + 
                      gamma_rec * loss_rec_masked + 
                      gamma_do * loss_do)
        
        total_loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        
    return model
```

**Parameters:**
*   $N_{depth}$: Number of iterations for Stage 1. Not given in the sources.
*   $\gamma_{depth}, \gamma_{rec}, \gamma_{do}$: Weighting factors for the respective losses. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Monocular depth estimation errors can propagate into the 3D geometry, causing artifacts in the inpainted region [S1].
*   **Boundaries:** The transition between the inpainted region and the original scene must be smooth to avoid visible seams [S1].
*   **Degenerate Input:** If the reference view is not representative of the scene, the inpainting may fail to generalize to other views [S1].

## Subtopics

### Progressive Inpainting-and-Erasing
Uses a progressive approach to erase and inpaint regions. Choose when gradual refinement is needed. [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)

### Deceptive-NeRF/3DGS Pipeline
Combines NeRF and 3D Gaussian Splatting for inpainting. Choose for high-fidelity results with complex geometry. [Deceptive-NeRF/3DGS Pipeline](11860-deceptive-nerf-3dgs-pipeline.md)

### Blended-NeRF
Blends inpainted regions with the original NeRF. Choose for simple blending without complex view-dependent effects. [Blended-NeRF](12273-blended-nerf.md)

### NeRF-In Inpainting Optimization
Optimizes the NeRF to match inpainted images. Choose for optimization-based approaches. [NeRF-In Inpainting Optimization](12372-nerf-in-inpainting-optimization.md)

### Adaptive View Selection for Inpainting
Selects optimal views for inpainting. Choose when view selection is critical. [Adaptive View Selection for Inpainting](12405-adaptive-view-selection-for-inpainting.md)

### NeRF Editing Procedure
General procedure for editing NeRFs. Choose for broader editing tasks. [NeRF Editing Procedure](12491-nerf-editing-procedure.md)

### RePaint-NeRF Limitations
Discusses limitations of RePaint-NeRF. Choose to understand constraints. [RePaint-NeRF Limitations](14340-repaint-nerf-limitations.md)

### Semantic and Language-Guided Editing
Uses semantic and language guidance. Choose for text-based editing. [Semantic and Language-Guided Editing](14393-semantic-and-language-guided-editing.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Shows superior performance to NeRF inpainting baselines [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Avoids view inconsistencies, generates view-dependent effects, requires only a single reference view [S1].
*   **Limitations:** Relies on the quality of the reference view and monocular depth estimation [S1].
*   **When to Use:** When a single reference view is available and view-dependent effects are desired [S1].
*   **When Not to Use:** When multiple reference views are available and view-dependent effects are not critical [S1].

## Background
Originated in the paper "Reference-guided Controllable Inpainting of Neural Radiance Fields" [S1]. It addresses the need for NeRF editing tools that can inpaint regions in a view-consistent and controllable manner [S1].

## Sources
[S1] Reference-guided Controllable Inpainting of Neural Radiance Fields, https://doi.org/10.1109/ICCV51070.2023.01633, fetched 2026-10-29
