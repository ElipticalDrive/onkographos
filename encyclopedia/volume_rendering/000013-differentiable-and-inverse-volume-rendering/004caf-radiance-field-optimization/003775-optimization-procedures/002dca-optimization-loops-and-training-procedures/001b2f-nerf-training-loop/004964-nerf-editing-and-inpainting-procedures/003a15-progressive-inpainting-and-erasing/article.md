# Progressive Inpainting-and-Erasing

Also called: Conflicted Geometry Detection and Erasing, Inpainting-and-Erasing Strategy, PERF, PERF Algorithm

## Overview
Progressive Inpainting-and-Erasing (PERF) is a method for reconstructing a panoramic Neural Radiance Field (NeRF) from a single input panorama by iteratively expanding the field of view through novel view synthesis [S1]. It solves the problem of geometry conflicts that arise when inpainting invisible regions with generative models, which often produce structures inconsistent with the already-learned visible geometry [S1]. Unlike standard NeRF training that assumes known camera poses, PERF progressively samples new viewpoints, inpaints the resulting invisible regions using Stable Diffusion, and erases conflicting geometry from the training supervision to ensure consistency [S1].

## How it works
**Inputs and Outputs:**
- Input: A single RGB panorama image $I_{rgb}$ [S1].
- Output: A panoramic NeRF representation $f_{nerf}$ [S1].

**Data Structures:**
- **NeRF Representation:** Uses Instant-NGP [S1]. The hash table size is $2^{19}$ with 16 levels of feature vectors, each of channel size 2 [S1].
- **Reference Views Set ($I_{ref}$):** A set containing the initial RGBD panorama and subsequently added inpainted RGBD panoramas [S1].
- **Trajectories ($\{p_k\}$):** A set of novel view camera poses generated based on the initial depth map [S1].

**Formulas:**
1.  **Volume Rendering:** The expected pixel color $\hat{C}(r)$ for a ray $r(t) = o + td$ is computed as [S1]:
    $$ \hat{C}(r) = \sum_{i=1}^{N} w_i c_i $$
    where $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$, $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$, and $\delta_i = t_i - t_{i-1}$ [S1].

2.  **Invisible Region Mask ($m'$):** Detects regions in a reference view that become invisible from a new view [S1]:
    $$ m'_{ij} = \mathbb{I}(\text{warp}(v'_{ij}) - v_{ij} > \epsilon) $$
    where $v_{ij}$ is the reference depth, $v'_{ij}$ is the warped depth from the new view, $\epsilon$ is a small threshold, and $\mathbb{I}(\cdot)$ is the indicator function [S1]. A value of 1 indicates an invisible region [S1].

3.  **Conflicted Region Mask ($m''$):** Detects geometry conflicts between the inpainted new view and existing reference views [S1]:
    $$ m''_{ij} = \mathbb{I}(v_{ij} - \text{warp}(v'_{ij}) > \epsilon) $$
    where $v_{ij}$ is the depth from the reference view and $v'_{ij}$ is the warped depth from the new view [S1]. A value of 1 indicates inconsistent geometry [S1].

4.  **Final Conflict Mask:** Computed by performing a logic AND operation across the individual conflict masks $m''$ from all reference views in $I_{ref}$ [S1].

**Parameters:**
- $\epsilon$: A small value used as the threshold for depth difference in warping operations [S1].
- Hash table size: $2^{19}$ [S1].
- Feature vector levels: 16 [S1].
- Feature vector channel size: 2 [S1].
- Initial training iterations: 10,000 iterations for the single input panorama [S1].
- New view sampling: 32 new positions with approximately gradually increased distance from the original position [S1].
- Fine-tuning iterations per new view: 2,000 iterations total [S1].
    - First 100 iterations: Supervision from only the new reference view [S1].
    - Remaining 1,900 iterations: Supervision from all reference views [S1].
- Optimizer: Adam [S1].
- Learning rate: Initially $10^{-3}$, decayed to $10^{-4}$ with cosine annealing strategy for each learning stage of a newly sampled view [S1].
- Ray sampling: 2,048 rays sampled per iteration for supervision [S1].

**Steps:**
1.  **Initialization:** Estimate the initial depth map $I_d$ of the input panorama $I_{rgb}$ to obtain an RGBD panorama $I_{rgbd}$ [S1]. Train a single-view NeRF $f_{nerf}$ using $I_d$ and $I_{rgb}$ [S1]. Generate trajectories $\{p_k\}$ of novel views based on $I_d$ [S1]. Initialize the reference view set $I_{ref} = \{I_{rgbd}\}$ [S1].
2.  **Iterative Expansion:** For $n$ from 1 to `iters`:
    a.  Randomly sample a new camera view $p$ from $\{p_k\}$ [S1].
    b.  Compute the invisible regions in the reference views according to Eq. (4) [S1].
    c.  Perform collaborative RGBD inpainting using Stable Diffusion for RGB and a depth estimator for depth stitching to fill the invisible regions [S1].
    d.  For all $I_{ref} \in I_{ref}$, compute the conflicted regions according to Eq. (6) [S1].
    e.  Eliminate the identified conflicted regions from the supervision of the NeRF training [S1].
    f.  Add the new inpainted view to the reference views $I_{ref}$ [S1].
    g.  Fine-tune $f_{nerf}$ with the updated $I_{ref}$ for 2,000 iterations [S1].
3.  **Convergence:** The algorithm iterates until it converges to a completed 3D geometry [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_inpainting_and_erasing(I_rgb):
    # 1. Initialization
    I_d = estimate_depth(I_rgb) # Eq. (2)
    I_rgbd = combine(I_rgb, I_d)
    f_nerf = train_single_view_neRF(I_d, I_rgb) # Eq. (3)
    trajectories = generate_trajectories(I_d)
    I_ref = {I_rgbd}
    
    # 2. Progressive Loop
    for n in range(1, iters + 1):
        # Sample new view
        p = random_sample(trajectories)
        
        # Compute invisible regions in existing reference views
        # This uses warping from new view p to reference views
        invisible_masks = compute_invisible_masks(I_ref, p) # Eq. (4)
        
        # Inpaint invisible regions
        # Uses Stable Diffusion for RGB and depth estimator for depth
        I_new_rgbd = collaborative_rgb_inpainting(I_ref, p, invisible_masks) # Eq. (5)
        
        # Compute conflicted regions
        conflict_masks = []
        for I_ref_view in I_ref:
            # Warp new view depth to reference view depth
            # Detect inconsistencies
            mask = compute_conflict_mask(I_ref_view, I_new_rgbd) # Eq. (6)
            conflict_masks.append(mask)
        
        # Final conflict mask is AND of all individual masks
        final_conflict_mask = logical_and(conflict_masks)
        
        # Erase conflicted regions from supervision
        # The specific erasing mechanism removes these pixels from the loss calculation
        erase_conflicts(final_conflict_mask)
        
        # Add new view to reference set
        I_ref.add(I_new_rgbd)
        
        # Fine-tune NeRF
        # First 100 iters: only new view supervision
        # Next 1900 iters: all reference views supervision
        fine_tune_neRF(f_nerf, I_ref, iterations=2000)
        
    return f_nerf
```

**Edge Cases and Pitfalls:**
- **Geometry Conflict:** The primary pitfall is the generation of invisible regions that conflict with visible geometry from other views. This is mitigated by the erasing step [S1].
- **Perspective vs. Panorama:** The inpainting model (Stable Diffusion) is trained on perspective images, which may cause artifacts when applied to panoramic projections [S1].
- **Depth Estimation Error:** The initial depth estimation and warping operations rely on a small threshold $\epsilon$ to handle errors, but significant depth estimation inaccuracies may lead to incorrect conflict detection [S1].

## Operating characteristics
- **Time Complexity:** Training a scene takes approximately 2 hours on a single Nvidia V100 GPU [S1]. The process involves iterative training and inpainting, scaling with the number of sampled views (32 views) and iterations per view (2,000) [S1].
- **Memory Complexity:** Uses Instant-NGP with a hash table size of $2^{19}$ and 16 levels of feature vectors of channel size 2 [S1]. Requires 32G memory on the GPU [S1].
- **Strengths:** Effectively reconstructs panoramic NeRFs from a single input by handling the lack of multi-view consistency through generative inpainting and conflict erasing [S1].
- **Limitations:** Relies on the quality of the initial depth estimation and the generative inpainting model [S1]. Performance may degrade on in-the-wild data compared to synthetic datasets like Replica [S1].
- **When to use:** When only a single panorama is available and a full 3D reconstruction is required [S1].
- **When not to use:** When multiple views are available, as standard NeRF training would be more accurate and efficient [S1].

## Background
- **Origin:** Proposed in "PERF: Panoramic Neural Radiance Field From a Single Panorama" [S1].
- **History:** The method builds upon Neural Radiance Fields (NeRF) [1] and Instant-NGP [40] for efficient representation, and Stable Diffusion [61] for generative inpainting [S1].
- **Usage:** Evaluated on the Replica dataset and in-the-wild indoor scenes [S1].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
