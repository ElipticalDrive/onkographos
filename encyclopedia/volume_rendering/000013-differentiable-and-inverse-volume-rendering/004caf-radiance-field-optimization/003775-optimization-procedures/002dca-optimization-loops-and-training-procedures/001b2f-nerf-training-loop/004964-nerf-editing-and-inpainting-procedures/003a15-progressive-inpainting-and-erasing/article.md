# Progressive Inpainting-and-Erasing

Also called: Conflicted Geometry Detection and Erasing, Inpainting-and-Erasing Strategy, PERF, PERF Algorithm

## Overview
Progressive Inpainting-and-Erasing (PERF) is a training procedure for generating a panoramic Neural Radiance Field (NeRF) from a single input panorama. It solves the problem of geometry conflict that arises when inpainting invisible regions of a scene: the generated content often contradicts the geometry observed in the original visible regions. PERF sets itself apart from standard NeRF editing methods by explicitly detecting these conflicts via depth warping and erasing the conflicting pixels from the training supervision, while progressively expanding the viewpoint coverage through iterative inpainting and fine-tuning [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single input panorama image $I_{rgb}$ (resolution $2048 \times 1024$ in the source implementation) [S1].
*   **Output:** A panoramic NeRF representation $f_{nerf}$ [S1].

**Data Structures:**
*   **NeRF Representation:** The source uses Instant-NGP [S1]. This consists of a hash table of size $2^{19}$ with 16 levels of feature vectors, each of channel size 2 [S1]. The neural network maps 3D coordinates $\mathbf{x}$ and viewing directions $\mathbf{d}$ to density $\sigma$ and color $\mathbf{c}$ [S1].
*   **Reference Views Set ($I_{ref}$):** A set containing the initial RGBD panorama and subsequently added inpainted RGBD panoramas [S1].
*   **Trajectory Set ($\{p_k\}$):** A set of novel camera poses generated based on the initial depth map [S1].

**Parameters:**
*   $\epsilon$: A small threshold value for depth difference comparison [S1].
*   Hash table size: $2^{19}$ [S1].
*   Initial training iterations: 10,000 iterations for the single input panorama [S1].
*   New view sampling count: 32 new positions with approximately gradually increased distance from the original position [S1].
*   Fine-tuning iterations per new view: 2,000 iterations total [S1].
    *   First 100 iterations: Supervision from only the new reference view [S1].
    *   Remaining 1,900 iterations: Supervision from all reference views [S1].
*   Optimizer: Adam [S1].
*   Learning rate: Initially $10^{-3}$, decayed to $10^{-4}$ with cosine annealing strategy for each learning stage of a newly sampled view [S1].
*   Ray sampling: 2,048 rays sampled per iteration for supervision [S1].

**Steps:**

1.  **Initial Depth Estimation:** Estimate the initial depth map $I_d$ of the input single panorama $I_{rgb}$ using a depth estimator (Eq. 2 in source) to obtain an initial RGBD panorama $I_{rgbd}$ [S1].
2.  **Single-View NeRF Training:** Train a single-view NeRF $f_{nerf}$ using $I_d$ and $I_{rgb}$ based on the differentiable volume rendering loss (Eq. 3 in source) [S1].
3.  **Trajectory Generation:** Generate a set of trajectories $\{p_k\}$ for novel views based on the estimated depth map $I_d$ [S1].
4.  **Initialization:** Initialize the set of reference views $I_{ref} = \{I_{rgbd}\}$ [S1].
5.  **Iterative Expansion:** Iterate $n$ from 1 to `iters` (where `iters` corresponds to the 32 new positions):
    a.  **Sample View:** Randomly sample a new camera view pose $p$ from $\{p_k\}$ [S1].
    b.  **Invisible Region Detection:** Compute the invisible regions for the new view. This involves warping the depth map from the reference view to the new view. Let $v_{ij}$ be the reference depth and $v'_{ij}$ be the warped depth. The mask $m''_{ij}$ for invisible regions is computed as [S1]:
        $$
        m''_{ij} = \mathbb{I}(v'_{ij} - v_{ij} > \epsilon)
        $$
        where $\mathbb{I}(\cdot)$ is the indicator function, and $m''_{ij}=1$ indicates an invisible region (where the warped depth is significantly larger than the reference depth, implying the surface is occluded or further away) [S1].
    c.  **Collaborative RGBD Inpainting:** Perform inpainting on the invisible regions using Stable Diffusion for RGB and a depth estimator for depth, stitching the results to create a completed panorama for the new view (Eq. 5 in source) [S1].
    d.  **Conflict Detection and Erasing:** For all existing reference views $I_{ref} \in I_{ref}$, compute conflicted regions. This is done by warping the depth from the *new* view back to the *reference* view. Let $v_{ij}$ be the reference depth and $v'_{ij}$ be the warped depth from the new view. The inconsistency mask $m''_{ij}$ is computed as [S1]:
        $$
        m''_{ij} = \mathbb{I}(v_{ij} - \text{warp}(v'_{ij}) > \epsilon)
        $$
        Here, $m''_{ij}=1$ indicates inconsistent geometry (conflict). The final conflict mask is computed by performing a logic AND operation across the individual masks from all reference views [S1]. These conflicted regions are erased (removed from supervision) [S1].
    e.  **Update Reference Set:** Add the new inpainted view to the reference views $I_{ref}$ [S1].
    f.  **Fine-Tuning:** Fine-tune $f_{nerf}$ with the updated $I_{ref}$ for 2,000 iterations (100 with only the new view, 1,900 with all views) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_inpainting_and_erasing(I_rgb):
    """
    Implements the PERF algorithm.
    Input: I_rgb - Single panorama image
    Output: f_nerf - Trained NeRF model
    """
    # 1. Initial Depth Estimation
    I_d = estimate_depth(I_rgb) # Eq. 2 in source
    I_rgbd = combine_rgb_depth(I_rgb, I_d)
    
    # 2. Single-View NeRF Training
    # Uses Instant-NGP representation: hash table 2^19, 16 levels, channel size 2
    f_nerf = train_single_view_nerf(I_rgbd, iterations=10000)
    
    # 3. Generate Trajectories
    # 32 new positions with gradually increased distance
    trajectories = generate_trajectories(I_d, num_views=32)
    
    # 4. Initialize Reference Views
    I_ref = {I_rgbd}
    
    # 5. Iterative Expansion
    for n in range(1, 33): # iters = 32
        # a. Sample View
        p = random_sample(trajectories)
        
        # b. Invisible Region Detection
        # Warp reference depth to new view
        v_ref = get_depth_from_nerf(f_nerf, I_ref) # Or from stored I_rgbd
        v_warped = warp_depth(v_ref, p)
        epsilon = 0.01 # Small value, exact value not specified in text, assume small positive float
        mask_invisible = (v_warped - v_ref > epsilon)
        
        # c. Collaborative RGBD Inpainting
        # Use Stable Diffusion for RGB and depth estimator for depth
        I_new_rgb = inpaint_rgb(I_rgb, mask_invisible, p) # Eq. 5 in source
        I_new_depth = inpaint_depth(I_d, mask_invisible, p)
        I_new_rgbd = combine_rgb_depth(I_new_rgb, I_new_depth)
        
        # d. Conflict Detection and Erasing
        # Compute conflicted regions for all reference views
        final_conflict_mask = None
        for I_ref_view in I_ref:
            v_ref_current = get_depth_from_nerf(f_nerf, I_ref_view)
            v_new_warped = warp_depth(get_depth_from_nerf(f_nerf, I_new_rgbd), p, target_view=I_ref_view)
            # Eq. 6: m''_ij = I(v_ij - warp(v'_ij) > epsilon)
            mask_conflict = (v_ref_current - v_new_warped > epsilon)
            
            if final_conflict_mask is None:
                final_conflict_mask = mask_conflict
            else:
                # Logic AND operation across all reference views
                final_conflict_mask = final_conflict_mask & mask_conflict
        
        # Erase conflicted regions from supervision
        # This implies masking out these pixels during the loss calculation in fine-tuning
        
        # e. Update Reference Set
        I_ref.add(I_new_rgbd)
        
        # f. Fine-Tuning
        # 2000 iterations total
        # First 100: only new view
        fine_tune(f_nerf, {I_new_rgbd}, iterations=100, lr_start=1e-3, lr_end=1e-4)
        # Remaining 1900: all reference views
        fine_tune(f_nerf, I_ref, iterations=1900, lr_start=1e-3, lr_end=1e-4)
        
    return f_nerf

def fine_tune(model, views, iterations, lr_start, lr_end):
    """
    Fine-tunes the NeRF model.
    Samples 2048 rays per iteration.
    Uses Adam optimizer.
    """
    optimizer = Adam(model.parameters(), lr=lr_start)
    scheduler = CosineAnnealingLR(optimizer, T_max=iterations, eta_min=lr_end)
    
    for i in range(iterations):
        rays = sample_rays(views, num_rays=2048)
        rendered_color, rendered_depth = render_rays(model, rays)
        target_color, target_depth = get_targets(rays)
        
        # Mask out conflicted regions if provided (handled in main loop logic)
        loss = compute_loss(rendered_color, target_color) + \
               compute_loss(rendered_depth, target_depth)
        
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        scheduler.step()
```

**Edge Cases and Pitfalls:**
*   **Precision:** The threshold $\epsilon$ must be small to allow for minor errors but large enough to detect significant geometry conflicts [S1].
*   **Boundary Conditions:** The warping function must handle occlusions correctly; points that are visible in the new view but occluded in the reference view (or vice versa) must be correctly identified as invisible or conflicted [S1].
*   **Degenerate Input:** The method assumes a single panorama input. If the input panorama has significant noise or inaccurate depth estimation, the initial NeRF training may be poor, affecting subsequent inpainting and conflict detection [S1].

## Operating characteristics
*   **Time Complexity:** The method involves iterative training and fine-tuning. Training a scene takes about 2 hours on a single Nvidia V100 GPU with 32G memory [S1].
*   **Memory Complexity:** Uses Instant-NGP representation with a hash table of size $2^{19}$ and 16 levels of feature vectors of channel size 2 [S1].
*   **Scaling:** The method scales with the number of sampled views (32 in the source) and the resolution of the input panorama ($2048 \times 1024$) [S1].
*   **Strengths:** Effectively avoids geometry conflict between visible and invisible regions by explicitly detecting and erasing conflicted areas [S1].
*   **Limitations:** Relies on the accuracy of the initial depth estimation and the inpainting model (Stable Diffusion) [S1].
*   **When to use:** When generating a NeRF from a single panorama and avoiding geometry artifacts in inpainted regions is critical [S1].
*   **When not to use:** When computational resources are limited (2 hours per scene on V100) or when high-quality depth estimation is not available [S1].

## Background
*   **Origin:** Proposed in "PERF: Panoramic Neural Radiance Field From a Single Panorama" [S1].
*   **History:** The method builds on previous work in NeRF and inpainting, specifically addressing the issue of geometry conflict in single-view NeRF generation [S1].
*   **Usage:** Evaluated on the Replica dataset and in-the-wild data [S1].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
