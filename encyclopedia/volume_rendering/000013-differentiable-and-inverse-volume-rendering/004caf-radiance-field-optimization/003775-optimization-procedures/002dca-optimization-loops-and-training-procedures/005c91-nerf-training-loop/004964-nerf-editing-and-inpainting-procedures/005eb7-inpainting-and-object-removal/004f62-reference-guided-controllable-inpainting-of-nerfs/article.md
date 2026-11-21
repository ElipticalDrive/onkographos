# Reference-guided Controllable Inpainting of NeRFs

Also called: Reference-guided Inpainting

## Overview
Reference-guided Controllable Inpainting of NeRFs is a method for 3D scene inpainting that utilizes a single inpainted reference view to guide the geometry and appearance of a Neural Radiance Field (NeRF), ensuring view consistency and generating view-dependent effects [S2]. It solves the problem of view inconsistencies and lack of view-dependent effects (VDEs) in non-reference views by back-projecting the reference view using monocular depth estimation and employing a bilateral solver to propagate appearance [S2]. Unlike methods that require multi-view inpainted inputs or complex text prompts, this approach allows user control via a single 2D image, avoiding the visual quality deterioration often seen in methods that rely on multiple inconsistent inpainted views [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $n$ input images $\{I_i\}_{i=1}^n$, camera transform matrices $\{\Pi_i\}_{i=1}^n$, binary masks $\{M_i\}_{i=1}^n$ delineating unwanted regions, and a single inpainted reference view $I_r$ where $r \in \{1, \dots, N\}$ [S2].
*   **Outputs:** An optimized NeRF representation $f_\theta: (\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$ that reconstructs the scene with the object removed and the reference content extended consistently [S2].

**Data Structures:**
*   **NeRF Representation:** Standard MLP-based NeRF storing density $\sigma$ and color $\mathbf{c}$ for 3D coordinates $\mathbf{x}$ and viewing directions $\mathbf{d}$ [Parent].
*   **Ray Set:** Rays $r$ are sampled from input views. Rays are classified into $R_{unmasked}$ (pixels where $M_i=0$) and $R_{masked}$ (pixels where $M_i=1$) [S2].
*   **Depth Map:** A disparity map $D(r)$ is computed from the monocular depth estimator applied to $I_r$, converted to inverse depth for supervision [S2].

**Parameters:**
*   $N_{depth}$: Number of iterations for initial unmasked supervision.
*   $N_{bilateral}$: Iteration interval for bilateral solver updates.
*   $N_{do}$: Iteration interval for disocclusion handling.
*   $\gamma_{depth}, \gamma_{rec}, \gamma_{do}$: Loss weights (values not given in the sources) [S2].

**Numbered Steps:**

1.  **Initialization:** Initialize NeRF parameters $\theta$. Compute the set of unmasked rays $R_{unmasked}$ corresponding to pixels in $\{I_i(1 - M_i)\}_{i=1}^n$ [S2].
2.  **Stage 1: Unmasked Supervision:** Optimize $f_\theta$ on $R_{unmasked}$ for $N_{depth}$ iterations using the standard reconstruction loss $L_{rec}^{unmasked}$. The masked region remains under-fit to allow altering masked values in later stages [S2].
    $$
    L_{rec}^{unmasked} = \mathbb{E}_{r \sim R_{unmasked}} \| \hat{C}(r) - C_{GT}(r) \|^2
    $$
    where $\hat{C}(r)$ is the rendered color and $C_{GT}(r)$ is the ground truth color [S2].
3.  **Geometry Supervision via Monocular Depth:**
    *   Use a monocular depth estimator to predict depth for the inpainted reference view $I_r$.
    *   Back-project $I_r$ to 3D positions using the camera matrix $\Pi_r$ and the estimated depth.
    *   Compute a disparity map. Align this with the current NeRF scene.
    *   Supervise the masked region's geometry using the depth loss $L_{depth}^{masked}$ [S2].
4.  **View-Dependent Effects (VDEs) via Bilateral Solver:**
    *   For non-reference views, use a view-substitution technique followed by bilaterally guided inpainting.
    *   The bilateral solver constructs VDEs in non-reference views from the reference viewpoint, ensuring that the inpainted region appears consistent from any view [S2].
    *   Obtain target colors $I_{br,t}$ for non-reference views.
    *   Compute the reconstruction loss on masked pixels $L_{rec}^{masked}$ using these target colors [S2].
5.  **Disocclusion Handling:**
    *   Identify non-reference disoccluded regions (areas visible in other views but not covered by the single reference view).
    *   Use image inpainters to guide both geometry and appearance for these regions via loss $L_{do}$ [S2].
6.  **Total Loss Optimization:**
    The total loss is computed as:
    $$
    L = L_{rec}^{unmasked} + \gamma_{depth} L_{depth}^{masked} + \gamma_{rec} L_{rec}^{masked} + \gamma_{do} L_{do}
    $$
    Losses $L_{depth}$, $L_{rec}$, and $L_{do}$ are computed every $N_{depth}$, $N_{bilateral}$, and $N_{do}$ iterations respectively [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reference_guided_inpainting(images, cameras, masks, inpainted_ref_view, depth_estimator, bilateral_solver, image_inpainter, n_iterations):
    """
    Performs reference-guided inpainting of a NeRF.
    
    Args:
        images: List of n input images {I_i}
        cameras: List of n camera matrices {Pi}
        masks: List of n binary masks {Mi}
        inpainted_ref_view: Single inpainted reference image Ir
        depth_estimator: Monocular depth estimation model
        bilateral_solver: Bilateral solver for view-dependent effects
        image_inpainter: 2D image inpainting model
        n_iterations: Total training iterations
        
    Returns:
        Optimized NeRF parameters theta
    """
    # Initialize NeRF
    theta = initialize_nerf()
    
    # Precompute ray sets
    unmasked_rays = []
    masked_rays = []
    for i in range(len(images)):
        for ray in sample_rays(images[i], cameras[i]):
            if masks[i][ray.pixel] == 0:
                unmasked_rays.append(ray)
            else:
                masked_rays.append(ray)
                
    # Stage 1: Unmasked Supervision
    for step in range(1, n_iterations + 1):
        # Compute unmasked reconstruction loss
        loss_unmasked = compute_reconstruction_loss(theta, unmasked_rays, images, cameras)
        
        # Gradient step for unmasked rays
        update_theta(theta, loss_unmasked)
        
        # Stage 2: Geometry and Appearance Guidance
        if step % N_depth == 0:
            # Geometry Supervision
            # 1. Estimate depth for reference view
            ref_depth = depth_estimator(inpainted_ref_view)
            # 2. Back-project to 3D and compute disparity
            ref_disparity = compute_disparity(ref_depth, cameras[r])
            # 3. Supervise masked geometry
            loss_depth = compute_depth_loss(theta, masked_rays, ref_disparity)
            update_theta(theta, loss_depth)
            
        if step % N_bilateral == 0:
            # View-Dependent Effects
            # 1. Use bilateral solver to get target colors for non-reference views
            target_colors = bilateral_solver(inpainted_ref_view, cameras, theta)
            # 2. Compute reconstruction loss on masked rays using target colors
            loss_rec_masked = compute_reconstruction_loss_with_targets(theta, masked_rays, target_colors)
            update_theta(theta, loss_rec_masked)
            
        if step % N_do == 0:
            # Disocclusion Handling
            # 1. Identify disoccluded regions in non-reference views
            disoccluded_rays = identify_disocclusions(theta, cameras, masks)
            # 2. Guide geometry and appearance using image inpainter
            loss_do = compute_disocclusion_loss(theta, disoccluded_rays, image_inpainter)
            update_theta(theta, loss_do)
            
    return theta
```

## Subtopics

### MVInpainter
MVInpainter is a multi-view consistent inpainting model designed to bridge 2D and 3D scene editing [S1]. It employs domain adapters and motion modules as video priors to enhance cross-view consistency [S1]. It introduces a Reference Key&Value concatenation technique for appearance consistency while incorporating slot attention to aggregate optical flow features for controlling camera movement without explicit pose conditions [S1].

## Operating characteristics
*   **Time Complexity:** The method requires iterative optimization of the NeRF. The complexity is dominated by the number of iterations $N_{depth}, N_{bilateral}, N_{do}$ and the cost of monocular depth estimation and bilateral solving per iteration.
*   **Memory Complexity:** Requires storing the NeRF parameters, input images, masks, and intermediate depth maps.
*   **Strengths:** Avoids view inconsistencies by relying on a single reference view; generates view-dependent effects; allows user control via a single image [S2].
*   **Limitations:** Relies on the quality of the single inpainted reference view; may struggle with complex geometry and textures in disoccluded regions [S1, S2].
*   **When to use:** When a user wants to inpaint a NeRF with specific content provided as a single 2D image and requires view-consistent results [S2].
*   **When not to use:** When high-quality multi-view inpainted inputs are available and consistency is less critical than texture fidelity, or when the reference view is of poor quality [S2].

## Background
Reference-guided Controllable Inpainting of NeRFs was introduced in 2023 [S2]. It addresses limitations in existing scene inpainting methods, such as inaccurate removal of large objects and lack of multi-view consistency [S1]. The method builds on the principles of differentiable volume rendering and NeRF optimization [Parent].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S2] Reference-guided Controllable Inpainting of Neural Radiance Fields, https://doi.org/10.1109/ICCV51070.2023.01633, fetched 2026-10-29
