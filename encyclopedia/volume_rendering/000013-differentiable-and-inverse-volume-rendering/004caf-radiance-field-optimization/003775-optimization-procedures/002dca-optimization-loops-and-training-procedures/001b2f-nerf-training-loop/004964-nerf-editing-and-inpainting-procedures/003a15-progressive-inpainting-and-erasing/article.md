# Progressive Inpainting-and-Erasing

Also called: Conflicted Geometry Detection and Erasing, Inpainting-and-Erasing Strategy, PERF, PERF Algorithm

## Overview
Progressive Inpainting-and-Erasing (PERF) is a training procedure for generating a panoramic Neural Radiance Field (NeRF) from a single input panorama by iteratively synthesizing novel views and refining the geometry. It solves the problem of geometry conflicts that arise when inpainting invisible regions of a single-view scene, ensuring that the generated geometry remains consistent with the original visible observations. Unlike standard single-view NeRF methods that may produce inconsistent depth, PERF explicitly detects and erases conflicted regions from the training supervision while using collaborative RGBD inpainting to fill invisible areas [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A single input panorama image $I_{rgb}$ (resolution $2048 \times 1024$ in the source implementation) [S1].
*   **Output:** A panoramic NeRF function $f_{nerf}$ mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$ [S1].

**Data Structures**
*   **NeRF Representation:** Instant-NGP [S1].
    *   Hash table size: $2^{19}$ [S1].
    *   Levels: 16 [S1].
    *   Feature vector channel size: 2 [S1].
*   **Reference Views Set:** $I_{ref}$, initialized with the input RGBD panorama $I_{rgbd}$ [S1].
*   **Novel View Trajectories:** $\{p_k\}$, generated based on the initial depth map [S1].
*   **Ray Sampling:** 2048 rays sampled per iteration [S1].

**Formulas**
1.  **Initial RGBD Estimation:**
    The depth map $I_d$ is estimated from $I_{rgb}$ using an external depth estimator (Eq. 2 in [S1]). The input becomes $I_{rgbd} = (I_{rgb}, I_d)$ [S1].
2.  **Single-View NeRF Training:**
    $f_{nerf}$ is trained using $I_{rgbd}$ based on Eq. 3 in [S1]. The loss combines photometric and depth consistency terms [S1].
3.  **Invisible Region Detection:**
    For a new view $p'$, depth $v'_{ij}$ is warped to the reference view coordinate system. Invisible regions $m'$ are identified where the warped depth is significantly larger than the reference depth $v_{ij}$:
    $$ m'_{ij} = \mathbb{I}(warp(v_{ij}) - v'_{ij} > \epsilon) $$
    where $\mathbb{I}(\cdot)$ is the indicator function, $warp(\cdot)$ is the warping function, and $\epsilon$ is a small threshold [S1].
4.  **Collaborative RGBD Inpainting:**
    Invisible regions are inpainted using Stable Diffusion for RGB and a depth estimator for depth stitching (Eq. 5 in [S1]) [S1].
5.  **Conflicted Region Detection:**
    To ensure the inpainted geometry does not conflict with existing reference views, the reference depth $v_{ij}$ is warped to the new view's coordinate system. Conflicted regions $m''$ are identified where the reference depth is significantly larger than the warped new view depth $v'_{ij}$:
    $$ m''_{ij} = \mathbb{I}(v_{ij} - warp(v'_{ij}) > \epsilon) $$
    where $v_{ij}$ is the reference depth and $v'_{ij}$ is the warped depth from the new view [S1].
6.  **Final Conflict Mask:**
    The final conflict mask is computed by performing a logic AND operation across the individual conflict masks $m''$ from all views in $I_{ref}$ [S1].

**Parameters**
*   $\epsilon$: Threshold for depth difference in conflict/invisible detection. Value: "A small value" [S1].
*   Hash table size: $2^{19}$ [S1].
*   Feature levels: 16 [S1].
*   Feature channel size: 2 [S1].
*   Initial training iterations: 10,000 [S1].
*   New view sampling: 32 new positions with gradually increased distance from the original position [S1].
*   Fine-tuning iterations per new view: 2,000 total [S1].
    *   First 100 iterations: Supervision from the new reference view only [S1].
    *   Remaining 1,900 iterations: Supervision from all reference views [S1].
*   Optimizer: Adam [S1].
*   Learning rate: Initially $10^{-3}$, decayed to $10^{-4}$ with cosine annealing for each new view stage [S1].
*   Ray count per iteration: 2,048 [S1].

**Steps**
1.  Estimate depth map $I_d$ of $I_{rgb}$ to obtain $I_{rgbd}$ [S1].
2.  Train single-view NeRF $f_{nerf}$ using $I_{rgbd}$ for 10,000 iterations [S1].
3.  Generate novel view trajectories $\{p_k\}$ based on $I_d$ [S1].
4.  Initialize reference views $I_{ref} = \{I_{rgbd}\}$ [S1].
5.  For $n = 1$ to $iters$:
    a.  Randomly sample a new camera view $p$ from $\{p_k\}$ [S1].
    b.  Compute invisible regions in $p$ using Eq. (4) [S1].
    c.  Perform collaborative RGBD inpainting on invisible regions using Stable Diffusion and depth estimator (Eq. 5) [S1].
    d.  For each $I_{ref} \in I_{ref}$:
        i.  Compute conflicted regions using Eq. (6) [S1].
        ii. Erase conflicted regions from the supervision of the NeRF training for this view [S1].
    e.  Add the inpainted and erased new view to $I_{ref}$ [S1].
    f.  Fine-tune $f_{nerf}$ with the updated $I_{ref}$ for 2,000 iterations (100 with new view only, 1900 with all) [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_inpainting_and_erasing(I_rgb):
    # 1. Initial Depth Estimation
    I_d = estimate_depth(I_rgb) # Eq. 2 in [S1]
    I_rgbd = (I_rgb, I_d)
    
    # 2. Single-View NeRF Initialization
    # Instant-NGP: hash_size=2^19, levels=16, channels=2
    f_nerf = InstantNGP()
    f_nerf.train(I_rgbd, iterations=10000, optimizer=Adam, lr=1e-3)
    
    # 3. Generate Trajectories
    # 32 positions with gradually increased distance
    trajectories = generate_trajectories(I_d, num_views=32)
    
    # 4. Initialize Reference Views
    I_ref = [I_rgbd]
    
    # 5. Progressive Loop
    for n in range(iters):
        # Sample new view
        p = random_sample(trajectories)
        
        # Render depth from current NeRF for new view p
        v_prime = f_nerf.render_depth(p)
        
        # Compute invisible regions (Eq. 4)
        # Warp reference depth to new view coordinates
        v_ref_warped = warp_depth(I_ref[0].depth, p, I_ref[0].pose)
        epsilon = 0.01 # Small value [S1]
        mask_invisible = (v_ref_warped - v_prime > epsilon)
        
        # Collaborative RGBD Inpainting (Eq. 5)
        # Use Stable Diffusion for RGB, depth estimator for depth
        I_new_rgb = inpaint_rgb(I_rgb, mask_invisible, model="StableDiffusion")
        I_new_depth = inpaint_depth(I_d, mask_invisible, estimator="DepthEstimator")
        I_new_rgbd = (I_new_rgb, I_new_depth)
        
        # Erase conflicted regions
        # Compute conflict mask against all existing reference views
        mask_conflict_global = None
        for ref_view in I_ref:
            # Warp new view depth to reference view coordinates
            v_new_warped_to_ref = warp_depth(v_prime, ref_view.pose, p)
            # Eq. 6: Conflict if reference depth > warped new depth
            mask_conflict = (ref_view.depth - v_new_warped_to_ref > epsilon)
            
            if mask_conflict_global is None:
                mask_conflict_global = mask_conflict
            else:
                # Logic AND across all reference views
                mask_conflict_global = mask_conflict_global & mask_conflict
        
        # Erase conflicted regions from supervision
        # [G] Implementation detail: mask out pixels in loss calculation
        # where mask_conflict_global is True.
        
        # Add new view to reference set
        I_ref.append(I_new_rgbd)
        
        # Fine-tune NeRF
        # First 100 iters: new view only
        f_nerf.train([I_new_rgbd], iterations=100, optimizer=Adam, lr=1e-3)
        # Remaining 1900 iters: all reference views
        f_nerf.train(I_ref, iterations=1900, optimizer=Adam, lr=1e-3)
        
    return f_nerf
```

**Edge Cases and Pitfalls**
*   **Depth Estimation Error:** The initial depth map $I_d$ is critical. Errors here propagate to trajectory generation and conflict detection [S1].
*   **Warping Accuracy:** The warping function $warp(\cdot)$ must accurately project depth maps between camera poses. Inaccurate warping leads to false positive conflict detection [S1].
*   **Threshold $\epsilon$:** The choice of $\epsilon$ affects sensitivity. Too small may miss conflicts; too large may erase valid geometry [S1].
*   **Inpainting Consistency:** Stable Diffusion may generate RGB content inconsistent with the depth map. The collaborative RGBD inpainting aims to mitigate this, but artifacts may persist [S1].

## Subtopics

### Collaborative RGBD Inpainting
A method to fill invisible regions using Stable Diffusion for RGB and a depth estimator for depth, ensuring RGBD consistency. Chosen when inpainting is required for novel views. [S1]

### Single-View NeRF Training
The base training of a NeRF from a single RGBD image. Chosen as the initialization step. [S1]

## Operating characteristics
*   **Time Complexity:** Training a scene takes approximately 2 hours on a single Nvidia V100 GPU with 32G memory [S1]. The iterative nature (32 new views, 2000 iterations each) adds significant time compared to standard NeRF training.
*   **Memory Complexity:** Depends on the Instant-NGP hash table ($2^{19}$ entries) and the storage of reference views. The source specifies 32G memory usage [S1].
*   **Performance:** Evaluated on the Replica dataset (2048x1024 resolution) and in-the-wild data. The method produces panoramic NeRFs from a single panorama [S1].
*   **Strengths:** Avoids geometry conflicts between visible and invisible regions. Enables panoramic NeRF generation from a single input.
*   **Limitations:** Relies on external models (Stable Diffusion, depth estimator). Computationally expensive due to iterative training and inpainting. Limited to panoramic scenes in the source implementation.
*   **When to use:** When a panoramic NeRF is needed from a single panorama and geometry consistency is critical.
*   **When not to use:** When computational resources are limited or when perspective views are sufficient.

## Background
*   **Origin:** Proposed in "PERF: Panoramic Neural Radiance Field From a Single Panorama" [S1].
*   **History:** The method addresses the limitations of single-view NeRFs in generating consistent panoramic geometry. It builds on Instant-NGP for efficient representation and Stable Diffusion for inpainting [S1].
*   **Usage:** Evaluated on the Replica dataset and in-the-wild indoor scenes [S1].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
