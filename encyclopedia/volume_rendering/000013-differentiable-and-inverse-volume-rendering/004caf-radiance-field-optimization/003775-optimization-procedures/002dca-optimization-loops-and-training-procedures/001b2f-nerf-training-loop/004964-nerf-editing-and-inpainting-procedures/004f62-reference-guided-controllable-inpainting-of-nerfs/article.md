# Reference-guided Controllable Inpainting of NeRFs

Also called: Reference-guided Inpainting

## Overview
A 3D inpainting algorithm that uses a single inpainted reference view to guide the geometry and appearance of a NeRF, avoiding view inconsistencies and generating view-dependent effects [S1]. It sets itself apart from multi-view inpainting methods by requiring only one user-provided inpainted image to control the entire 3D scene, and from purely generative methods by explicitly modeling view-dependent effects (VDEs) and disocclusions through optimization rather than implicit diffusion priors [S1].

## How it works
**Inputs:** $n$ input images $\{I_i\}_{i=1}^n$, camera transform matrices $\{\Pi_i\}_{i=1}^n$, masks $\{M_i\}_{i=1}^n$ delineating unwanted regions, and a single inpainted reference view $I_r$ where $r \in \{1, ..., N\}$ [S1].

**Outputs:** A trained radiance field $f_\theta$ representing the inpainted scene [S1].

**Data Structures:**
*   $f_\theta$: MLP mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$.
*   $I_r$: 2D image tensor of the inpainted reference view.
*   $M_i$: Binary mask tensor for input image $i$.
*   $\hat{C}(r)$: Rendered color for ray $r$, computed via volume rendering [S1, 12491-nerf-editing-procedure.md].

**Steps:**

1.  **Initialization:** Initialize $f_\theta$ with standard NeRF parameters [S1].
2.  **Stage 1: Unmasked Supervision:** Supervise $f_\theta$ on unmasked pixels for $N_{depth}$ iterations using the standard NeRF reconstruction loss $L_{rec}^{unmasked}$ [S1]. The masked region remains under-fit to allow altering masked values in later stages [S1].
    $$ L_{rec}^{unmasked} = \mathbb{E}_{r \sim R_{unmasked}} \| \hat{C}(r) - C_{GT}(r) \|^2 $$
    where $R_{unmasked}$ is the set of rays corresponding to pixels in $\{I_i(1 - M_i)\}_{i=1}^n$ and $C_{GT}(r)$ is the ground truth color [S1].
3.  **Geometry Supervision via Depth:** Use a monocular depth estimator to predict depth for the inpainted reference view $I_r$. Back-project $I_r$ to 3D positions to compute a disparity map. Align this map with the current inpainted NeRF scene. Compute masked depth supervision loss $L_{depth}^{masked}$ [S1].
4.  **View-Dependent Effects (VDEs):** For non-reference views, use a view-substitution technique followed by bilaterally guided inpainting to obtain target colors $I_{br,t}$ [S1]. These targets are used to compute the masked reconstruction loss $L_{rec}^{masked}$ [S1].
5.  **Disocclusion Handling:** For non-reference disoccluded regions not supervised by the single reference view, use image inpainters to guide both geometry and appearance via loss $L_{do}$ [S1].
6.  **Joint Optimization:** Minimize the total loss:
    $$ L = L_{rec}^{unmasked} + \gamma_{depth} L_{depth}^{masked} + \gamma_{rec} L_{rec}^{masked} + \gamma_{do} L_{do} $$
    Losses $L_{depth}^{masked}$, $L_{rec}^{masked}$, and $L_{do}$ are computed every $N_{depth}$, $N_{bilateral}$, and $N_{do}$ iterations respectively [S1].

**Parameters:**
*   $N_{depth}$: Iterations for depth supervision. Not given in the sources.
*   $N_{bilateral}$: Iterations for bilateral VDE supervision. Not given in the sources.
*   $N_{do}$: Iterations for disocclusion supervision. Not given in the sources.
*   $\gamma_{depth}, \gamma_{rec}, \gamma_{do}$: Weighting coefficients. Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reference_guided_inpainting(images, cameras, masks, reference_view, net, optimizer):
    # Stage 1: Unmasked Supervision
    for i in range(N_depth):
        rays_unmasked = get_rays(images, cameras, masks, unmasked=True)
        colors_gt = get_gt_colors(rays_unmasked, images)
        colors_pred = volume_render(rays_unmasked, net)
        loss_rec_unmasked = mse(colors_pred, colors_gt)
        loss_rec_unmasked.backward()
        optimizer.step()
        optimizer.zero_grad()

    # Stage 2: Geometry and Appearance Supervision
    # Compute depth from reference view
    depth_map = monocular_depth_estimator(reference_view)
    points_3d = back_project(reference_view, depth_map, cameras[reference_view_idx])
    
    while not converged:
        # 1. Depth Loss
        if current_iter % N_depth == 0:
            rays_masked = get_rays(images, cameras, masks, unmasked=False)
            # Align depth map with NeRF geometry
            depth_pred = render_depth(rays_masked, net)
            loss_depth = align_and_compute_loss(depth_pred, depth_map, points_3d)
            loss_depth.backward()
        
        # 2. VDE Loss
        if current_iter % N_bilateral == 0:
            # Generate target colors for non-reference views using bilateral solver
            target_colors = bilateral_inpainting(reference_view, cameras, masks)
            rays_non_ref = get_rays(images, cameras, masks, exclude_reference=True)
            colors_pred = volume_render(rays_non_ref, net)
            loss_rec_masked = mse(colors_pred, target_colors)
            loss_rec_masked.backward()
            
        # 3. Disocclusion Loss
        if current_iter % N_do == 0:
            # Identify disoccluded regions in non-reference views
            disoccluded_rays = find_disocclusions(rays_non_ref, cameras)
            # Guide with image inpainter
            disoccluded_targets = image_inpainter(images, masks, disoccluded_rays)
            colors_pred_do = volume_render(disoccluded_rays, net)
            loss_do = compute_disocclusion_loss(colors_pred_do, disoccluded_targets)
            loss_do.backward()
            
        optimizer.step()
        optimizer.zero_grad()
        current_iter += 1
```

## Operating characteristics
*   **Time Complexity:** Depends on the number of iterations $N_{depth}, N_{bilateral}, N_{do}$ and the cost of monocular depth estimation and bilateral solving per iteration.
*   **Memory Complexity:** Standard NeRF memory plus storage for the reference view and intermediate depth maps.
*   **Strengths:** Avoids view inconsistencies by using a single reference; generates view-dependent effects; handles disocclusions.
*   **Limitations:** Requires a high-quality inpainted reference view; performance degrades if the reference view is inconsistent with the scene geometry.
*   **When to use:** When a user wants to inpaint a region in a NeRF and can provide a single example of what that region should look like.
*   **When not to use:** When no reference view is available or when the reference view is significantly different from the target inpainted content.

## Background
Originated in the paper "Reference-guided Controllable Inpainting of Neural Radiance Fields" [S1]. It addresses the limitations of previous NeRF inpainting methods which lacked view-dependent effects and struggled with disocclusions when using multiple views [S1].

## Sources
[S1] Reference-guided Controllable Inpainting of Neural Radiance Fields, https://doi.org/10.1109/ICCV51070.2023.01633, fetched 2026-10-29
