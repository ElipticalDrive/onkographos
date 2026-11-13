# Progressive Inpainting-and-Erasing

Also called: Conflicted Geometry Detection and Erasing, Inpainting-and-Erasing Strategy, PERF, PERF Algorithm

## Overview
Progressive Inpainting-and-Erasing (PERF) is a method for reconstructing a panoramic Neural Radiance Field (NeRF) from a single input panorama by iteratively expanding the viewpoint coverage. It solves the problem of geometry conflict between visible regions (observed in reference views) and invisible regions (occluded or unseen) by computing a mask of conflicted areas and eliminating them from the NeRF training supervision [S1]. Unlike methods that rely on multiple input views or semantic priors, PERF generates novel views via inpainting and progressively integrates them while erasing geometrically inconsistent pixels to ensure 3D consistency [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single input RGB panorama $I_{rgb}$ [S1].
*   **Output:** A panoramic NeRF representation $f_{nerf}$ [S1].

**Data Structures:**
*   **NeRF Representation:** Instant-NGP [S1]. Uses a hash table of size $2^{19}$ with 16 levels of feature vectors, each having a channel size of 2 [S1].
*   **Reference Views Set:** $I_{ref}$, initialized with the input RGBD panorama [S1].
*   **Trajectory Set:** $\{p_k\}$, a set of novel view positions generated based on the initial depth map [S1].

**Formulas:**
1.  **Initial RGBD Estimation:** The input $I_{rgb}$ is processed to estimate a depth map $I_d$, resulting in an RGBD panorama $I_{rgbd}$ [S1]. (Specific equation number referenced as Eq. (2) in source, but formula details for depth estimation are not provided in the text; [G] assumes a standard monocular depth estimator is used here).
2.  **Single-View NeRF Training:** The initial NeRF $f_{nerf}$ is trained using $I_d$ and $I_{rgb}$ [S1]. (Specific equation number referenced as Eq. (3) in source, but formula details are not provided; [G] assumes standard NeRF volume rendering loss minimization).
3.  **Invisible Region Mask ($m'$):** For a new view, invisible regions are detected by warping the new view's depth $v'$ to the reference view's coordinate system and comparing it with the reference depth $v$.
    $$
    m'_{ij} = \mathbb{I}(warp(v'_{ij}) - v_{ij} > \epsilon)
    $$
    where $m'_{ij}$ is the mask pixel (1 for invisible, 0 for visible), $v_{ij}$ is the reference depth, $v'_{ij}$ is the warped depth from the new view, $warp(\cdot)$ is the warping function, $\mathbb{I}(\cdot)$ is the indicator function, and $\epsilon$ is a small threshold value [S1].
4.  **Conflicted Region Mask ($m''$):** For each reference view $I_{ref}$, conflicted regions are detected by warping the reference depth $v$ to the new view's coordinate system and comparing it with the new view's depth $v'$.
    $$
    m''_{ij} = \mathbb{I}(v_{ij} - warp(v'_{ij}) > \epsilon)
    $$
    where $m''_{ij}$ is the mask pixel (1 for inconsistent/conflicted, 0 for consistent) [S1].
5.  **Final Conflict Mask:** The final conflict mask is computed by performing a logic AND operation across the individual masks $m''$ from all reference views [S1].
6.  **Collaborative RGBD Inpainting:** Invisible regions are inpainted using Stable Diffusion for RGB and a depth estimator for depth stitching [S1]. (Specific equation number referenced as Eq. (5) in source, but formula details are not provided; [G] implies using the inpainted RGB and estimated depth to complete the panorama).

**Parameters:**
*   $\epsilon$: A small value used as a threshold for depth difference [S1].
*   Hash table size: $2^{19}$ [S1].
*   Levels: 16 [S1].
*   Feature vector channel size: 2 [S1].
*   Initial training iterations: 10,000 iterations for the single input panorama [S1].
*   New view sampling: 32 new positions with approximately gradually increased distance from the original position [S1].
*   Fine-tuning iterations per new view: 2,000 iterations total [S1].
    *   First 100 iterations: Supervision only from the new reference view [S1].
    *   Remaining 1,900 iterations: Supervision from all reference views [S1].
*   Optimizer: Adam [S1].
*   Learning rate: Initially $10^{-3}$, decayed to $10^{-4}$ with cosine annealing strategy for each learning stage of a newly sampled view [S1].
*   Ray sampling: 2,048 rays sampled per iteration [S1].

**Numbered Steps:**
1.  Estimate the initial depth map $I_d$ of the input single panorama $I_{rgb}$ to obtain $I_{rgbd}$ [S1].
2.  Train a single-view NeRF $f_{nerf}$ using $I_d$ and $I_{rgb}$ for 10,000 iterations [S1].
3.  Generate trajectories $\{p_k\}$ of novel views based on the depth map $I_d$ [S1].
4.  Initialize the set of reference views $I_{ref} = \{I_{rgbd}\}$ [S1].
5.  Iterate $n$ from 1 to `iters` (where `iters` corresponds to the number of new views, e.g., 32):
    1.  Randomly sample a new camera view $p$ from $\{p_k\}$ [S1].
    2.  Compute the invisible regions according to Eq. (4) [S1].
    3.  Perform collaborative RGBD inpainting using Stable Diffusion and a depth estimator to complete the RGB and depth maps of the invisible regions [S1].
    4.  For all $I_{ref} \in I_{ref}$:
        1.  Compute the conflicted regions according to Eq. (6) [S1].
        2.  Erase these conflicted regions from the supervision of the NeRF training [S1].
    5.  Add the new inpainted view to the reference views $I_{ref}$ [S1].
    6.  Fine-tune $f_{nerf}$ with the updated $I_{ref}$ for 2,000 iterations (100 iterations with only the new view, 1,900 with all views) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_inpainting_and_erasing(I_rgb):
    # 1. Estimate depth and create RGBD panorama
    I_d = estimate_depth(I_rgb) # Eq. (2)
    I_rgbd = combine_rgb_depth(I_rgb, I_d)
    
    # 2. Train initial single-view NeRF
    f_nerf = train_single_view_nerf(I_rgbd, iterations=10000) # Eq. (3)
    
    # 3. Generate novel view trajectories
    p_k = generate_trajectories(I_d, num_views=32)
    
    # 4. Initialize reference views
    I_ref = {I_rgbd}
    
    # 5. Progressive loop
    for n in range(1, len(p_k) + 1):
        # Sample new view
        p = random_sample(p_k)
        
        # Render initial depth for new view from current NeRF
        v_new = render_depth(f_nerf, p)
        
        # Compute invisible regions (Eq. 4)
        # Warp new view depth to reference view coordinates
        v_new_warped = warp(v_new, p, I_ref[0].pose)
        v_ref = I_ref[0].depth
        epsilon = 0.01 # Small value
        m_invisible = indicator_function(v_new_warped - v_ref > epsilon)
        
        # Perform collaborative RGBD inpainting (Eq. 5)
        # Use Stable Diffusion for RGB and depth estimator for depth
        I_new_rgb = inpaint_rgb(I_rgb, m_invisible, model="StableDiffusion")
        I_new_depth = inpaint_depth(I_d, m_invisible, model="DepthEstimator")
        I_new_rgbd = combine_rgb_depth(I_new_rgb, I_new_depth)
        
        # Compute conflicted regions and erase (Eq. 6)
        # Initialize final conflict mask
        m_conflict_final = None
        
        for I_ref_view in I_ref:
            # Warp reference view depth to new view coordinates
            v_ref_warped = warp(I_ref_view.depth, I_ref_view.pose, p)
            m_conflict = indicator_function(I_ref_view.depth - v_ref_warped > epsilon)
            
            if m_conflict_final is None:
                m_conflict_final = m_conflict
            else:
                # Logic AND operation across all reference views
                m_conflict_final = logical_and(m_conflict_final, m_conflict)
        
        # Erase conflicted regions from supervision
        # [G] Implementation detail: The loss calculation for the new view
        # should ignore pixels where m_conflict_final is 1.
        
        # Add new view to reference set
        I_ref.add(I_new_rgbd)
        
        # Fine-tune NeRF
        # First 100 iterations: only new view
        train_nerf(f_nerf, {I_new_rgbd}, iterations=100, lr=1e-3)
        # Remaining 1900 iterations: all reference views
        train_nerf(f_nerf, I_ref, iterations=1900, lr_schedule="cosine_anneal", 
                   start_lr=1e-3, end_lr=1e-4, exclude_conflicted=m_conflict_final)
        
    return f_nerf
```

**Edge Cases and Pitfalls:**
*   **Depth Estimation Error:** The initial depth estimation from a single panorama may be inaccurate, leading to incorrect trajectory generation or invisible region detection [S1].
*   **Geometry Conflict:** Invisible regions inpainted in one view may conflict with visible regions in another reference view. The erasing mechanism is crucial to prevent this [S1].
*   **Precision:** The threshold $\epsilon$ must be carefully chosen to allow for small errors while detecting significant conflicts [S1].
*   **Boundary Conditions:** The warping function must handle occlusions and depth discontinuities correctly [S1].

## Subtopics

### Collaborative RGBD Inpainting
A sub-method that uses Stable Diffusion for RGB inpainting and a depth estimator for depth stitching to complete invisible regions [S1]. It is chosen when monocular depth estimation is insufficient and external generative priors are needed.

### Single-View NeRF Training
The initial training phase using a single RGBD panorama [S1]. It is chosen as the starting point for the progressive expansion.

### Depth Warping
The geometric operation used to detect invisible and conflicted regions by projecting depth maps between camera poses [S1]. It is chosen for its efficiency in identifying visibility changes.

## Operating characteristics
*   **Time Complexity:** The method involves iterative training and inpainting. Training a scene takes about 2 hours on a single Nvidia V100 GPU with 32G memory [S1].
*   **Memory Complexity:** Uses Instant-NGP representation with a hash table of size $2^{19}$ [S1].
*   **Measured Performance:** Evaluated on the Replica dataset (2048x1024 resolution) and in-the-wild data [S1].
*   **Strengths:** Can reconstruct panoramic NeRFs from a single input view; handles geometry conflicts effectively; ensures 3D consistency through erasing [S1].
*   **Limitations:** Relies on the quality of the initial depth estimation and the inpainting model; may struggle with highly dynamic or textureless scenes [S1].
*   **When to use:** When only a single panorama is available and a full 3D reconstruction is required.
*   **When not to use:** When multiple input views are available, or when high-fidelity texture is critical and inpainting artifacts are unacceptable.

## Background
*   **Origin:** Proposed in "PERF: Panoramic Neural Radiance Field From a Single Panorama" [S1].
*   **History:** The method builds on the concept of NeRF [1] and uses Instant-NGP [40] for efficient representation. It addresses the limitation of NeRF requiring multiple views by introducing a progressive inpainting-and-erasing strategy [S1].
*   **Usage:** Used for reconstructing indoor scenes from single panoramic images, as demonstrated on the Replica dataset and in-the-wild data [S1].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
