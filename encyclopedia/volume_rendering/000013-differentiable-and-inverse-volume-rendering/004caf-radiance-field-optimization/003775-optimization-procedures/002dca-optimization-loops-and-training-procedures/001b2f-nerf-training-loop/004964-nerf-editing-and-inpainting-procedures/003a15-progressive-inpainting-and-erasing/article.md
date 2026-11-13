# Progressive Inpainting-and-Erasing

Also called: Conflicted Geometry Detection and Erasing, Inpainting-and-Erasing Strategy, PERF, PERF Algorithm

## Overview
Progressive Inpainting-and-Erasing (PERF) is a training procedure for generating a panoramic Neural Radiance Field (NeRF) from a single input panorama. It solves the problem of geometry conflict between visible regions (observed in the input) and invisible regions (synthesized via inpainting) by iteratively detecting and erasing inconsistent geometry from the supervision signal. Unlike methods that rely on multi-view consistency from the start, PERF progressively expands the viewpoint range by inpainting novel views and erasing regions that violate geometric consistency with existing reference views [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single input panorama image $I_{rgb}$ [S1].
*   **Output:** A panoramic NeRF function $f_{nerf}$ mapping 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to density $\sigma$ and color $\mathbf{c}$ [S1].

**Data Structures:**
*   **NeRF Representation:** Instant-NGP [S1].
    *   Hash table size: $2^{19}$ [S1].
    *   Levels: 16 [S1].
    *   Feature vector channel size: 2 [S1].
*   **Reference Views Set ($I_{ref}$):** A set of RGB-D images used for supervision. Initially contains the input panorama with estimated depth [S1].
*   **Trajectory Set ($\{p_k\}$):** A set of novel camera poses generated based on the initial depth map, with gradually increasing distance from the original position [S1].

**Formulas:**
1.  **Initial Depth Estimation:** Estimate depth map $I_d$ from $I_{rgb}$ (Eq. 2 in [S1]). The initial RGB-D image is $I_{rgbd} = (I_{rgb}, I_d)$ [S1].
2.  **NeRF Loss (Eq. 3 in [S1]):** Standard volume rendering loss combining RGB and depth supervision [S1].
3.  **Invisible Region Mask ($m'$):** Detects regions in a reference view that are occluded or inconsistent with a new view's depth.
    $$ m'_{ij} = \mathbb{I}(warp(v'_{ij}) - v_{ij} > \epsilon) $$
    Where:
    *   $v_{ij}$ is the depth value at pixel $(i,j)$ in the reference view's depth map $I_d$.
    *   $v'_{ij}$ is the depth value at pixel $(i,j)$ in the new view's depth map $I'_d$.
    *   $warp(\cdot)$ is the warping function projecting the new view's depth to the reference view's coordinate system [S1].
    *   $\epsilon$ is a small threshold value for depth difference [S1].
    *   $\mathbb{I}(\cdot)$ is the indicator function (1 if true, 0 if false) [S1].
    *   $m'_{ij}=1$ indicates an invisible region (conflict) [S1].
4.  **Conflicted Region Mask ($m''$):** Detects regions in the reference view that conflict with the inpainted geometry of the new view.
    $$ m''_{ij} = \mathbb{I}(v_{ij} - warp(v'_{ij}) > \epsilon) $$
    Where:
    *   $v_{ij}$ is the depth value in the reference view.
    *   $v'_{ij}$ is the depth value in the new view.
    *   $m''_{ij}=1$ indicates inconsistent geometry (1) or consistent (0) [S1].
5.  **Final Conflict Mask:** Computed by performing a logic AND operation across the individual conflict masks $m''$ from all reference views in $I_{ref}$ [S1].
6.  **Collaborative RGBD Inpainting (Eq. 5 in [S1]):** Uses Stable Diffusion for RGB inpainting and a depth estimator for depth stitching to complete the invisible regions of the new view [S1].

**Parameters:**
*   **$\epsilon$:** Small value for depth difference threshold [S1].
*   **Hash Table Size:** $2^{19}$ [S1].
*   **Initial Training Iterations:** 10,000 iterations for the single input panorama [S1].
*   **New View Sampling:** 32 new positions with approximately gradually increased distance from the original position [S1].
*   **Fine-tuning Iterations per New View:** 2,000 iterations total [S1].
    *   First 100 iterations: Supervision only from the new reference view [S1].
    *   Remaining 1,900 iterations: Supervision from all reference views [S1].
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** Initially $10^{-3}$, decayed to $10^{-4}$ with cosine annealing strategy for each learning stage of a newly sampled view [S1].
*   **Ray Sampling:** 2,048 rays sampled per iteration for supervision [S1].

**Steps:**
1.  Estimate the initial depth map $I_d$ of the input single panorama $I_{rgb}$ using the depth estimator (Eq. 2) to obtain $I_{rgbd}$ [S1].
2.  Train a single-view NeRF $f_{nerf}$ using $I_d$ and $I_{rgb}$ based on Eq. 3 [S1].
3.  Generate trajectories $\{p_k\}$ of novel views based on the depth map $I_d$ [S1].
4.  Initialize the set of reference views $I_{ref} = \{I_{rgbd}\}$ [S1].
5.  Iterate $n$ from 1 to `iters` (where `iters` corresponds to the number of new views, e.g., 32):
    a.  Randomly sample a new camera view $p$ from $\{p_k\}$ [S1].
    b.  Compute invisible regions according to Eq. 4 [S1].
    c.  Perform collaborative RGBD inpainting using Stable Diffusion and a depth estimator (Eq. 5) to complete the RGB and depth of the new view [S1].
    d.  For all $I_{ref} \in I_{ref}$:
        i.  Compute conflicted regions according to Eq. 6 [S1].
        ii. Erase these conflicted regions from the supervision of the NeRF training [S1].
    e.  Add the new view (inpainting-and-erased) to the reference views $I_{ref}$ [S1].
    f.  Fine-tune $f_{nerf}$ with the updated $I_{ref}$ for 2,000 iterations (100 with only the new view, 1,900 with all views) using Adam optimizer with cosine annealing learning rate [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_inpainting_and_erasing(I_rgb):
    # 1. Initial Depth Estimation
    I_d = estimate_depth(I_rgb) # Eq. 2
    I_rgbd = (I_rgb, I_d)
    
    # 2. Initial NeRF Training
    f_nerf = train_single_view_nerf(I_rgb, I_d, iterations=10000) # Eq. 3
    
    # 3. Generate Trajectories
    trajectories = generate_trajectories(I_d, num_views=32)
    
    # 4. Initialize Reference Views
    I_ref = {I_rgbd}
    
    # 5. Progressive Loop
    for n in range(1, len(trajectories) + 1):
        # Sample new view
        p = random_sample(trajectories)
        
        # Render initial depth for new view
        I_d_new = render_depth(f_nerf, p)
        
        # Compute invisible regions (mask m')
        # Warp new view depth to reference view coordinates
        I_d_new_warped = warp_depth(I_d_new, p, reference_view_pose)
        m_prime = indicator_function(I_d_new_warped - I_d_ref > epsilon)
        
        # Collaborative RGBD Inpainting (Eq. 5)
        # Use Stable Diffusion for RGB and depth estimator for depth
        I_rgb_inpainted, I_d_inpainted = collaborative_rgb_d_inpainting(
            I_rgb_new, I_d_new, m_prime
        )
        I_new_rgbd = (I_rgb_inpainted, I_d_inpainted)
        
        # Compute conflicted regions and erase
        # For each reference view in I_ref
        conflict_mask_global = None
        for I_ref_view in I_ref:
            I_d_ref = I_ref_view.depth
            I_rgb_ref = I_ref_view.rgb
            
            # Warp new view depth to reference view
            I_d_new_warped_ref = warp_depth(I_d_inpainted, p, I_ref_view.pose)
            
            # Compute conflict mask m'' (Eq. 6)
            m_double_prime = indicator_function(I_d_ref - I_d_new_warped_ref > epsilon)
            
            # Logic AND across all reference views
            if conflict_mask_global is None:
                conflict_mask_global = m_double_prime
            else:
                conflict_mask_global = conflict_mask_global & m_double_prime
        
        # Erase conflicted regions from supervision
        # Create masks for RGB and Depth supervision
        rgb_supervision_mask = ~conflict_mask_global
        depth_supervision_mask = ~conflict_mask_global
        
        # Add new view to reference set
        I_ref.add(I_new_rgbd)
        
        # Fine-tune NeRF
        # First 100 iterations: only new view
        for i in range(100):
            rays = sample_rays(I_new_rgbd, rgb_supervision_mask, depth_supervision_mask)
            loss = compute_nerf_loss(f_nerf, rays)
            optimizer.step(loss)
            
        # Remaining 1900 iterations: all reference views
        for i in range(1900):
            rays = sample_rays_from_all_references(I_ref, rgb_supervision_mask, depth_supervision_mask)
            loss = compute_nerf_loss(f_nerf, rays)
            optimizer.step(loss)
            
    return f_nerf
```

**Edge Cases and Pitfalls:**
*   **Depth Estimation Error:** The initial depth estimation from a single panorama may be inaccurate, affecting the trajectory generation and warping accuracy [S1].
*   **Warping Errors:** Incorrect warping between views can lead to false positives in conflict detection [S1].
*   **Inpainting Artifacts:** Stable Diffusion may generate geometrically inconsistent textures, which are then detected and erased, potentially leaving holes or requiring further inpainting [S1].
*   **Convergence:** The algorithm iterates until convergence, which is defined by the completion of the predefined number of new views (32) and fine-tuning [S1].

## Subtopics

### Collaborative RGBD Inpainting
A method to complete RGB images and depth maps of regions of interest using Stable Diffusion and a depth estimator. It is chosen when geometry conflict needs to be resolved by synthesizing plausible content for invisible regions. [S1]

### Single-View NeRF Training
The initial training of a NeRF from a single panorama with estimated depth. It is chosen as the starting point for the progressive expansion. [S1]

## Operating characteristics
*   **Time Complexity:** Training a scene takes about 2 hours on a single Nvidia V100 GPU with 32G memory [S1].
*   **Memory Complexity:** Uses Instant-NGP representation with a hash table size of $2^{19}$ and 16 levels of feature vectors of channel size 2 [S1].
*   **Performance:** Evaluated on the Replica dataset (2048x1024 resolution) and in-the-wild data [S1].
*   **Strengths:** Avoids geometry conflict between visible and invisible regions by progressively erasing inconsistent geometry. Allows generation of panoramic NeRF from a single input. [S1]
*   **Limitations:** Relies on the accuracy of single-view depth estimation and inpainting quality. The progressive nature requires multiple fine-tuning stages. [S1]
*   **When to use:** When only a single panorama is available and a full 360-degree NeRF is required. [S1]
*   **When not to use:** When multi-view images are available, as other methods may be more efficient and accurate. [S1]

## Background
*   **Origin:** Proposed in "PERF: Panoramic Neural Radiance Field From a Single Panorama" [S1].
*   **History:** The method builds on the concept of Neural Radiance Fields [1] and uses Instant-NGP [40] for efficient representation. It addresses the limitation of single-view NeRF reconstruction by incorporating inpainting and conflict detection. [S1]
*   **Usage:** Used for generating panoramic NeRFs from single images, evaluated on Replica dataset and in-the-wild data. [S1]

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
