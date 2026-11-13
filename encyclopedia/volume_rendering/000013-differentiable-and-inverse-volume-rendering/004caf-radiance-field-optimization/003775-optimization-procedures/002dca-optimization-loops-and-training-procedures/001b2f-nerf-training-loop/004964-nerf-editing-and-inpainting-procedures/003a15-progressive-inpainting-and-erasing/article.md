# Progressive Inpainting-and-Erasing

Also called: Conflicted Geometry Detection and Erasing, Inpainting-and-Erasing Strategy, PERF, PERF Algorithm

## Overview
Progressive Inpainting-and-Erasing (PERF) is a method for reconstructing a panoramic Neural Radiance Field (NeRF) from a single input panorama by iteratively expanding the field of view through novel view synthesis [S1]. It solves the problem of geometry conflict between visible regions (observed in the initial view) and invisible regions (synthesized via inpainting) by detecting and erasing inconsistent geometry from the training supervision [S1]. Unlike methods that rely on multiple input views or explicit 3D priors, PERF progressively builds the 3D representation by alternating between inpainting missing regions in 2D projections and erasing conflicting 3D geometry detected via depth warping [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single input panorama image $I_{rgb}$ [S1].
*   **Output:** A panoramic NeRF representation $f_{nerf}$ [S1].

**Data Structures:**
*   **NeRF Representation ($f_{nerf}$):** Uses the Instant-NGP architecture [S1].
    *   **Hash Table:** Size $2^{19}$ [S1].
    *   **Levels:** 16 levels of feature vectors [S1].
    *   **Channel Size:** 2 per feature vector [S1].
*   **Reference Views Set ($I_{ref}$):** A set containing the initial RGBD panorama and subsequently added inpainted RGBD panoramas [S1].
*   **Trajectory Set ($\{p_k\}$):** A set of novel camera poses generated based on the initial depth map [S1].

**Parameters:**
*   $\epsilon$: A small threshold value for depth difference comparison [S1].
*   Hash table size: $2^{19}$ [S1].
*   Initial training iterations: 10,000 iterations for the single input panorama [S1].
*   New view sampling count: 32 new positions with approximately gradually increased distance from the original position [S1].
*   Fine-tuning iterations per new view: 2,000 iterations total [S1].
    *   First 100 iterations: Supervision from the new reference view only [S1].
    *   Remaining 1,900 iterations: Supervision from all reference views [S1].
*   Optimizer: Adam [S1].
*   Learning rate: Initially $10^{-3}$, decayed to $10^{-4}$ with cosine annealing strategy for each learning stage of a newly sampled view [S1].
*   Ray sampling: 2,048 rays sampled per iteration [S1].

**Numbered Steps:**

1.  **Initial Depth Estimation:** Estimate the initial depth map $I_d$ of the input single panorama $I_{rgb}$ using a depth estimator (Eq. 2 in [S1]) to obtain the initial RGBD panorama $I_{rgbd}$ [S1].
2.  **Initial NeRF Training:** Train a single-view NeRF $f_{nerf}$ using $I_d$ and $I_{rgb}$ based on the standard NeRF volume rendering equation (Eq. 3 in [S1]) for 10,000 iterations [S1].
3.  **Trajectory Generation:** Generate a set of novel view trajectories $\{p_k\}$ based on the estimated depth map $I_d$ [S1].
4.  **Initialization:** Initialize the set of reference views $I_{ref} = \{I_{rgbd}\}$ [S1].
5.  **Progressive Loop:** Iterate $n$ from 1 to `iters` (where `iters` corresponds to the 32 new positions) [S1]:
    1.  **Sample View:** Randomly sample a new camera view pose $p$ from $\{p_k\}$ [S1].
    2.  **Invisible Region Detection:** Compute the invisible regions for the new view $p$ relative to the current NeRF geometry. This is done by warping depth from the new view to the reference view and identifying pixels where the warped depth is significantly larger than the reference depth, indicating occlusion/invisibility (Eq. 4 in [S1]) [S1].
    3.  **Collaborative RGBD Inpainting:** Perform inpainting on the invisible regions of the new view. Use Stable Diffusion for RGB inpainting and a depth estimator for depth stitching (Eq. 5 in [S1]) to generate a completed panorama for the new view [S1].
    4.  **Conflict Detection and Erasing:** For every view $I_{ref}$ in the reference set $I_{ref}$:
        *   Compute conflicted regions by warping the depth of the new view to the reference view and identifying pixels where the reference depth is significantly larger than the warped depth, indicating geometry inconsistency (Eq. 6 in [S1]) [S1].
        *   Erase these conflicted regions from the supervision of the NeRF training for that reference view [S1].
    5.  **Update Reference Set:** Add the inpainted and erased new view to the reference views set $I_{ref}$ [S1].
    6.  **Fine-tuning:** Fine-tune $f_{nerf}$ with the updated $I_{ref}$ for 2,000 iterations [S1].

**Formulas:**

*   **Volume Rendering (Eq. 3 in [S1]):**
    The expected pixel color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ is computed as:
    $$ \hat{C}(\mathbf{r}) = \sum_{i=1}^{N} w_i c_i $$
    where:
    *   $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$ is the weight of the $i$-th sample.
    *   $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance.
    *   $\sigma_i$ is the volume density, $c_i$ is the color, and $\delta_i = t_i - t_{i-1}$ is the distance between samples [S1].

*   **Invisible Region Mask (Eq. 4 in [S1]):**
    $$ m'_{ij} = \mathbb{I}(\text{warp}(v'_{ij}) - v_{ij} > \epsilon) $$
    where:
    *   $m'_{ij}$ is the mask pixel (1 for invisible, 0 for visible) [S1].
    *   $v_{ij}$ is the reference depth value [S1].
    *   $v'_{ij}$ is the warped depth value from the new view to the reference view [S1].
    *   $\text{warp}(\cdot)$ is the warping function [S1].
    *   $\mathbb{I}(\cdot)$ is the indicator function [S1].
    *   $\epsilon$ is a small threshold value [S1].

*   **Conflicted Region Mask (Eq. 6 in [S1]):**
    $$ m''_{ij} = \mathbb{I}(v_{ij} - \text{warp}(v'_{ij}) > \epsilon) $$
    where:
    *   $m''_{ij}$ is the mask pixel (1 for inconsistent/conflicted, 0 for consistent) [S1].
    *   $v_{ij}$ is the reference depth value [S1].
    *   $v'_{ij}$ is the warped depth value from the new view to the reference view [S1].
    *   The final mask of conflict regions is computed by performing a logic AND operation across the individual masks $m''$ from all reference views [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_inpainting_and_erasing(I_rgb):
    """
    Reconstruct a panoramic NeRF from a single panorama using PERF.
    
    Args:
        I_rgb: Input single panorama image.
        
    Returns:
        f_nerf: Trained panoramic NeRF model.
    """
    # Step 1: Estimate initial depth
    I_d = estimate_depth(I_rgb) # Eq. (2) in [S1]
    I_rgbd = combine_rgb_depth(I_rgb, I_d)
    
    # Step 2: Train initial single-view NeRF
    f_nerf = train_single_view_nerf(I_d, I_rgb, iterations=10000) # Eq. (3) in [S1]
    
    # Step 3: Generate trajectories
    trajectories = generate_trajectories(I_d)
    
    # Step 4: Initialize reference views
    I_ref = {I_rgbd}
    
    # Step 5: Progressive loop
    for n in range(1, 33): # 32 new positions [S1]
        # Sample new view
        p = random_sample(trajectories)
        
        # Compute invisible regions for new view
        # Warp depth from new view to reference view
        v_prime = warp_depth(f_nerf, p, I_ref)
        v_ref = get_depth(I_ref)
        m_invisible = (v_prime - v_ref > epsilon) # Eq. (4) in [S1]
        
        # Perform collaborative RGBD inpainting
        I_new_rgb = inpaint_rgb(I_new, m_invisible, model="Stable Diffusion") # Eq. (5) in [S1]
        I_new_depth = inpaint_depth(I_new, m_invisible, model="Depth Estimator") # Eq. (5) in [S1]
        I_new_rgbd = combine_rgb_depth(I_new_rgb, I_new_depth)
        
        # Compute conflicted regions and erase from supervision
        conflict_mask_global = None
        for I_ref_view in I_ref:
            # Warp depth from new view to this reference view
            v_prime_ref = warp_depth(f_nerf, p, I_ref_view)
            v_ref_view = get_depth(I_ref_view)
            m_conflict = (v_ref_view - v_prime_ref > epsilon) # Eq. (6) in [S1]
            
            if conflict_mask_global is None:
                conflict_mask_global = m_conflict
            else:
                conflict_mask_global = conflict_mask_global & m_conflict # Logic AND [S1]
        
        # Add new view to reference set
        I_ref.add(I_new_rgbd)
        
        # Fine-tune NeRF
        # First 100 iterations: only new view
        train_nerf(f_nerf, {I_new_rgbd}, conflict_mask_global, iterations=100, lr=1e-3)
        # Remaining 1900 iterations: all reference views
        train_nerf(f_nerf, I_ref, conflict_mask_global, iterations=1900, lr=1e-4)
        
    return f_nerf

def train_nerf(f_nerf, views, conflict_mask, iterations, lr):
    """
    Train NeRF with given views and conflict mask.
    
    Args:
        f_nerf: NeRF model.
        views: Set of RGBD views.
        conflict_mask: Mask indicating regions to erase from supervision.
        iterations: Number of training iterations.
        lr: Learning rate.
    """
    optimizer = Adam(f_nerf.parameters(), lr=lr)
    for i in range(iterations):
        # Randomly sample 2048 rays [S1]
        rays = sample_rays(views, count=2048)
        
        # Render colors and depths
        rendered_colors, rendered_depths = render(f_nerf, rays)
        
        # Get ground truth colors and depths
        gt_colors, gt_depths = get_ground_truth(rays, views)
        
        # Apply conflict mask to exclude conflicted regions from loss
        # [G] Implementation detail: mask out pixels where conflict_mask is 1
        valid_mask = ~conflict_mask[rays] # Assuming conflict_mask is aligned with rays
        
        # Compute loss (e.g., MSE)
        loss = compute_loss(rendered_colors, gt_colors, valid_mask)
        loss += compute_depth_loss(rendered_depths, gt_depths, valid_mask)
        
        # Backpropagate
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
```

**Edge Cases and Pitfalls:**
*   **Precision:** The threshold $\epsilon$ must be carefully chosen to allow for small errors while detecting significant geometry conflicts [S1].
*   **Boundaries:** The method assumes the initial depth estimation is sufficiently accurate to generate meaningful trajectories [S1].
*   **Degenerate Input:** The method requires a single panorama; if the input is not a panorama, the trajectory generation and warping may fail [S1].

## Subtopics

### Collaborative RGBD Inpainting
A sub-method used within PERF to inpaint invisible regions using Stable Diffusion for RGB and a depth estimator for depth, ensuring consistency between RGB and depth [S1].

### Single-View NeRF Training
The initial training phase using a single RGBD input, serving as the foundation for the progressive expansion [S1].

## Operating characteristics
*   **Time Complexity:** Training a scene takes about 2 hours on a single Nvidia V100 GPU with 32G memory [S1]. The complexity scales with the number of iterations (10k initial + 32 * 2k fine-tuning) and the number of reference views [S1].
*   **Memory Complexity:** Uses Instant-NGP hash table with size $2^{19}$ and 16 levels of feature vectors of channel size 2 [S1].
*   **Measured Performance:** Evaluated on the Replica dataset (8 scenes) with 2048x1024 resolution panoramas [S1]. Also evaluated on in-the-wild data (PERF-in-the-wild, 8 indoor scenes) [S1].
*   **Strengths:** Avoids geometry conflict between visible and invisible regions, enabling high-quality panoramic reconstruction from a single view [S1].
*   **Limitations:** Relies on the accuracy of the initial depth estimation and the inpainting model [S1]. The progressive nature may accumulate errors if early views are incorrect [S1].
*   **When to use:** When only a single panorama is available and a full 360-degree NeRF is required [S1].
*   **When not to use:** When multiple input views are available, as other methods may be more efficient or accurate [S1].

## Background
*   **Origin:** Proposed in "PERF: Panoramic Neural Radiance Field From a Single Panorama" [S1].
*   **History:** The method builds on the foundation of NeRF [S1] and Instant-NGP [S1], incorporating techniques from inpainting models like Stable Diffusion [S1] and depth estimation [S1].
*   **Usage:** Used for reconstructing panoramic scenes from a single input, particularly in indoor environments (Replica dataset) and in-the-wild scenes [S1].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
