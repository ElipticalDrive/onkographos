# Reference-guided Controllable Inpainting of NeRFs

Also called: Reference-guided Inpainting

## Overview
Reference-guided Controllable Inpainting of NeRFs is a 3D inpainting algorithm that uses a single inpainted reference view to guide the geometry and appearance of a Neural Radiance Field (NeRF), avoiding view inconsistencies and generating view-dependent effects [S1]. It solves the problem of editing NeRFs with limited guidance by back-projecting a single 2D inpainted image into 3D space and propagating its appearance to other viewpoints via a bilateral solver, while handling disocclusions using image inpainters [S1]. Unlike methods requiring multi-view consistency checks or extensive user interaction, this approach allows a user to control the generated scene via a single inpainted image [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   $n$ input images $\{I_i\}_{i=1}^n$.
*   Camera transform matrices $\{\Pi_i\}_{i=1}^n$.
*   Masks $\{M_i\}_{i=1}^n$ delineating unwanted regions in each view.
*   A single inpainted reference view $I_r$, where $r \in \{1, \dots, n\}$ [S1].

**Outputs:**
*   A trained NeRF $f_\theta: (\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$ representing the inpainted scene [S1].
*   Novel views synthesized from the edited NeRF [S1].

### Data Structures
The method operates on a standard NeRF representation $f_\theta$ mapping 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to volume density $\sigma$ and color $\mathbf{c}$ [Parent]. Rays $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ are sampled at $N$ points with distances $\delta_i = t_{i+1} - t_i$ and accumulated transmittance $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ [Parent].

### Parameters
*   $N_{depth}$: Number of iterations for Stage 1 supervision.
*   $N_{bilateral}$: Iteration interval for bilateral solver updates.
*   $N_{do}$: Iteration interval for disocclusion handling.
*   $\gamma_{depth}, \gamma_{rec}, \gamma_{do}$: Loss weights (values not given in the sources) [S1].

### Steps

1.  **Stage 1: Unmasked Supervision**
    Supervise $f_\theta$ on unmasked pixels for $N_{depth}$ iterations using the standard NeRF reconstruction loss $L_{rec}^{unmasked}$ [S1]. The masked region remains under-fit to allow altering masked values in later stages [S1].
    $$
    L_{rec}^{unmasked} = \mathbb{E}_{r \sim R_{unmasked}} \| \hat{C}(r) - C_{GT}(r) \|^2
    $$
    where $R_{unmasked}$ is the set of rays corresponding to pixels in $\{I_i (1 - M_i)\}_{i=1}^n$ and $C_{GT}(r)$ is the ground truth color [S1].

2.  **Geometry Supervision via Depth Estimation**
    Use a monocular depth estimator to compute a disparity map from the inpainted reference view $I_r$. Back-project $I_r$ to 3D positions using the camera parameters $\Pi_r$. Align this depth information with the current NeRF scene. Compute the masked depth loss $L_{depth}^{masked}$ to supervise the geometry of the inpainted region [S1].

3.  **View-Dependent Effects (VDEs) via Bilateral Solver**
    For non-reference views, generate target colors $I_{br,t}$ using a view-substitution technique followed by bilaterally guided inpainting [S1]. This simulates VDEs of non-reference views from the reference viewpoint. Compute the masked reconstruction loss $L_{rec}^{masked}$ using these target colors [S1].

4.  **Disocclusion Handling**
    For non-reference disoccluded regions not supervised by the single reference view, use image inpainters to guide both geometry and appearance. Compute the disocclusion loss $L_{do}$ [S1].

5.  **Total Loss and Optimization**
    The total loss is computed as:
    $$
    L = L_{rec}^{unmasked} + \gamma_{depth} L_{depth}^{masked} + \gamma_{rec} L_{rec}^{masked} + \gamma_{do} L_{do}
    $$
    Losses $L_{depth}^{masked}$, $L_{rec}^{masked}$, and $L_{do}$ are computed every $N_{depth}$, $N_{bilateral}$, and $N_{do}$ iterations respectively [S1]. Optimize $\theta$ using gradient descent (e.g., Adam) [Parent].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reference_guided_inpainting(images, cameras, masks, reference_view, 
                                n_depth_iters, n_bilateral_iters, n_do_iters,
                                gamma_depth, gamma_rec, gamma_do):
    """
    Perform reference-guided inpainting on a NeRF.
    
    Args:
        images: List of n input images {I_i}.
        cameras: List of n camera transform matrices {Pi}.
        masks: List of n masks {Mi} delineating unwanted regions.
        reference_view: The single inpainted reference view Ir.
        n_depth_iters: Iterations for Stage 1.
        n_bilateral_iters: Interval for bilateral solver updates.
        n_do_iters: Interval for disocclusion handling.
        gamma_depth, gamma_rec, gamma_do: Loss weights.
        
    Returns:
        Trained NeRF model f_theta.
    """
    # Initialize NeRF f_theta
    f_theta = initialize_nerf()
    
    # Stage 1: Supervise unmasked pixels
    for step in range(n_depth_iters):
        loss = 0.0
        for i in range(len(images)):
            # Get rays for unmasked pixels
            rays = get_unmasked_rays(images[i], masks[i], cameras[i])
            # Render colors
            rendered_colors = render_nerf(f_theta, rays)
            # Compute reconstruction loss
            gt_colors = get_gt_colors(rays, images[i])
            loss += mean_squared_error(rendered_colors, gt_colors)
        
        # Backpropagate and update
        update_nerf(f_theta, loss)
        
    # Stage 2: Iterative refinement with geometry, VDEs, and disocclusions
    for step in range(n_depth_iters, total_training_steps):
        total_loss = 0.0
        
        # 1. Unmasked reconstruction (always active)
        for i in range(len(images)):
            rays = get_unmasked_rays(images[i], masks[i], cameras[i])
            rendered_colors = render_nerf(f_theta, rays)
            gt_colors = get_gt_colors(rays, images[i])
            loss_unmasked = mean_squared_error(rendered_colors, gt_colors)
            total_loss += loss_unmasked
            
        # 2. Geometry supervision (every n_depth_iters)
        if step % n_depth_iters == 0:
            # Back-project reference view to 3D using monocular depth estimator
            depth_map = monocular_depth_estimator(reference_view)
            disparity_map = 1.0 / depth_map
            # Align with NeRF geometry
            # Compute masked depth loss L_depth_masked
            # [G]: Implementation details of alignment and loss computation depend on specific depth estimator output format
            loss_depth = compute_masked_depth_loss(f_theta, cameras, masks, disparity_map)
            total_loss += gamma_depth * loss_depth
            
        # 3. View-Dependent Effects via Bilateral Solver (every n_bilateral_iters)
        if step % n_bilateral_iters == 0:
            # Generate target colors for non-reference views using bilateral solver
            target_colors = bilateral_solver_inpainting(reference_view, cameras, masks)
            # Compute masked reconstruction loss L_rec_masked
            # [G]: Implementation details of bilateral solver and target color application
            loss_rec_masked = compute_masked_reconstruction_loss(f_theta, cameras, masks, target_colors)
            total_loss += gamma_rec * loss_rec_masked
            
        # 4. Disocclusion Handling (every n_do_iters)
        if step % n_do_iters == 0:
            # Identify disoccluded regions in non-reference views
            disoccluded_regions = identify_disocclusions(f_theta, cameras, masks)
            # Use image inpainters to guide geometry and appearance
            # Compute disocclusion loss L_do
            # [G]: Implementation details of disocclusion identification and image inpainter integration
            loss_do = compute_disocclusion_loss(f_theta, cameras, masks, disoccluded_regions)
            total_loss += gamma_do * loss_do
            
        # Backpropagate and update
        update_nerf(f_theta, total_loss)
        
    return f_theta
```

### Edge Cases and Pitfalls
*   **Precision:** Monocular depth estimation may introduce inaccuracies, affecting geometry supervision [S1].
*   **Boundaries:** The method relies on a single reference view; if the reference view is poorly chosen or contains artifacts, these may propagate [S1].
*   **Degenerate Input:** If the reference view is not representative of the scene, the inpainting may fail to generalize to other viewpoints [S1].

## Subtopics

### Bilateral Solver Inpainting
*   **What it is:** A technique to generate target colors for non-reference views by substituting views and applying bilateral guidance [S1].
*   **What sets it apart:** It simulates view-dependent effects without multiview appearance information [S1].
*   **When to choose it:** When view-dependent effects are needed in non-reference views but only a single reference view is available [S1].
*   **Link:** [Bilateral Solver Inpainting](link-to-bilateral-solver.md)

### Monocular Depth Estimation for Geometry
*   **What it is:** Using a monocular depth estimator to back-project a 2D reference view to 3D positions for geometry supervision [S1].
*   **What sets it apart:** It provides geometric guidance from a single 2D image, avoiding the need for multi-view stereo depth [S1].
*   **When to choose it:** When geometric consistency is required but multi-view depth is not available [S1].
*   **Link:** [Monocular Depth Estimation](link-to-monocular-depth.md)

### Disocclusion Handling via Image Inpainters
*   **What it is:** Using image inpainters to guide geometry and appearance in disoccluded regions not supervised by the reference view [S1].
*   **What sets it apart:** It handles regions that are not visible in the reference view, ensuring complete inpainting [S1].
*   **When to choose it:** When disocclusions are present in non-reference views [S1].
*   **Link:** [Disocclusion Handling](link-to-disocclusion-handling.md)

## Operating characteristics

*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The approach shows superior performance to NeRF inpainting baselines [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Avoids view inconsistencies, generates view-dependent effects, allows user control via a single inpainted image [S1].
*   **Limitations:** Relies on the quality of the single reference view and monocular depth estimation [S1].
*   **When to use it:** When editing NeRFs with limited guidance (single reference view) and requiring view-consistent inpainting [S1].
*   **When not to use it:** When high-quality multi-view depth is available and view-dependent effects are not critical [S1].

## Background

*   **Origin:** The method was introduced in "Reference-guided Controllable Inpainting of Neural Radiance Fields" [S1].
*   **History and Development:** The work addresses the desire for NeRF editing tools, focusing on inpainting regions in a view-consistent and controllable manner [S1].
*   **Where it is used:** NeRF editing and inpainting applications [S1].

## Sources

[S1] Reference-guided Controllable Inpainting of Neural Radiance Fields, https://doi.org/10.1109/ICCV51070.2023.01633, fetched 2026-10-29
