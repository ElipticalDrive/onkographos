# Progressive Inpainting-and-Erasing

Also called: Conflicted Geometry Detection and Erasing, Inpainting-and-Erasing Strategy, PERF, PERF Algorithm

## Overview
Progressive Inpainting-and-Erasing (PERF) is a method for reconstructing a panoramic Neural Radiance Field (NeRF) from a single input panorama by iteratively synthesizing novel views and refining the geometry. It solves the problem of geometry conflicts that arise when inpainting invisible regions with generative models, which may violate the consistent depth constraints of the already-learned visible regions. Unlike methods that rely on multiple input views or explicit 3D priors, PERF progressively enlarges the viewpoint range by randomly sampling new camera poses, inpainting missing RGBD data, and erasing conflicting regions from the training supervision to ensure geometric consistency [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single input panorama image $I_{rgb}$ with resolution $W \times H$ (e.g., $2048 \times 1024$).
*   **Output:** A panoramic NeRF representation $f_{nerf}$ mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$.

**Data Structures:**
*   **NeRF Representation:** Uses Instant-NGP [S1].
    *   **Hash Grid:** A hash table of size $2^{19}$ with 16 levels of feature vectors, each of channel size 2.
    *   **MLP:** A small MLP network taking hashed features and viewing direction to predict density $\sigma$ and color $\mathbf{c}$.
*   **Reference Views Set:** $I_{ref}$, a set of RGBD images (RGB + Depth) used for supervision. Initially contains the estimated depth of the input panorama.
*   **Trajectory Set:** $\{p_k\}$, a set of candidate novel camera poses generated based on the initial depth map.

**Formulas:**
1.  **NeRF Rendering:** The expected pixel color $\hat{C}(r)$ for a ray $r(t) = o + td$ is computed via volumetric rendering:
    $$ \hat{C}(r) = \sum_{i=1}^{N} w_i c_i $$
    where $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$, $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$, and $\delta_i = t_i - t_{i-1}$ [S1].

2.  **Invisible Region Mask ($m'$):** Detects regions in a reference view that become invisible from a new view $p'$ due to occlusion or depth inconsistency.
    $$ m'_{ij} = \mathbb{I}(\text{warp}(v_{ij}) - v'_{ij} > \epsilon) $$
    where $v_{ij}$ is the depth value at pixel $(i,j)$ in the reference view, $v'_{ij}$ is the warped depth from the new view, $\text{warp}(\cdot)$ projects the new view's depth to the reference view coordinates, $\epsilon$ is a small threshold, and $\mathbb{I}(\cdot)$ is the indicator function (1 if true, 0 otherwise) [S1].

3.  **Conflicted Region Mask ($m''$):** Detects regions in the reference view where the inpainted geometry from the new view conflicts with the existing reference geometry.
    $$ m''_{ij} = \mathbb{I}(v_{ij} - \text{warp}(v'_{ij}) > \epsilon) $$
    where $v_{ij}$ is the reference depth and $v'_{ij}$ is the warped depth from the new view. A pixel is conflicted if the reference depth is significantly larger than the warped depth from the new view, indicating the new view's geometry is "behind" the reference surface or inconsistent [S1].

4.  **Final Conflict Mask:** The final mask of conflicted regions is computed by performing a logic AND operation across the individual conflict masks $m''$ from all views in $I_{ref}$ [S1].

**Parameters:**
*   $\epsilon$: A small value used as the threshold for depth difference in warping operations [S1].
*   Hash table size: $2^{19}$ with 16 levels, channel size 2 [S1].
*   Initial training iterations: 10,000 iterations for the single input panorama [S1].
*   New view sampling: 32 new positions with approximately gradually increased distance from the original position [S1].
*   Fine-tuning iterations per new view: 2,000 iterations total. First 100 iterations use only the new reference view for supervision; remaining 1,900 iterations use all reference views [S1].
*   Optimizer: Adam [S1].
*   Learning rate: Initially $10^{-3}$, decayed to $10^{-4}$ with cosine annealing strategy for each learning stage of a newly sampled view [S1].
*   Ray sampling: 2,048 rays sampled per iteration for supervision [S1].

**Algorithm Steps:**
1.  **Initialization:**
    *   Estimate the initial depth map $I_d$ of the input single panorama $I_{rgb}$ using a depth estimator (Eq. 2 in source) to obtain the initial RGBD image $I_{rgbd}$ [S1].
    *   Train a single-view NeRF $f_{nerf}$ using $I_d$ and $I_{rgb}$ based on the standard NeRF loss (Eq. 3 in source) for 10,000 iterations [S1].
    *   Generate a trajectory set $\{p_k\}$ of novel views based on the estimated depth map $I_d$ [S1].
    *   Initialize the set of reference views $I_{ref} = \{I_{rgbd}\}$ [S1].

2.  **Progressive Loop:**
    *   Iterate $n$ from 1 to `iters` (total number of new views to synthesize, e.g., 32):
        1.  Randomly sample a new camera view pose $p$ from the trajectory set $\{p_k\}$ [S1].
        2.  **Inpainting:**
            *   Compute the invisible regions in the reference views according to Eq. (4) [S1].
            *   Perform collaborative RGBD inpainting using Stable Diffusion for RGB inpainting and a depth estimator for depth stitching (Eq. 5 in source) to generate the completed panorama for the new view [S1].
        3.  **Erasing:**
            *   For all $I_{ref} \in I_{ref}$:
                *   Compute the conflicted regions according to Eq. (6) [S1].
                *   Erase these conflicted regions from the supervision of the NeRF training (i.e., do not include these pixels in the loss calculation for the new view) [S1].
        4.  **Update:**
            *   Add the new inpainted and erased view to the reference views set $I_{ref}$ [S1].
            *   Fine-tune $f_{nerf}$ with the updated $I_{ref}$ for 2,000 iterations [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def PERF(I_rgb):
    # 1. Initialization
    I_d = EstimateDepth(I_rgb)  # Eq. (2)
    I_rgbd = (I_rgb, I_d)
    
    # Train initial single-view NeRF
    f_nerf = TrainNeRF(I_rgbd, iterations=10000)
    
    # Generate trajectory of novel views
    p_k = GenerateTrajectory(I_d)
    
    # Initialize reference views
    I_ref = {I_rgbd}
    
    # 2. Progressive Loop
    for n in range(1, 33):  # 32 new positions
        # Sample new camera pose
        p = RandomSample(p_k)
        
        # Render depth from current NeRF for reference views to warp
        # Note: Source implies warping new view depth to reference view coordinates
        # and reference view depth to new view coordinates for conflict detection
        
        # Compute invisible regions in reference views relative to new view p
        # This identifies areas in reference views that are occluded or inconsistent
        # with the new view's perspective
        invisible_mask = ComputeInvisibleMask(I_ref, p)
        
        # Perform collaborative RGBD inpainting
        # Stable Diffusion inpaints RGB, depth estimator inpaints depth
        I_new_rgbd = CollaborativeRGBDInpainting(I_ref, p, invisible_mask)
        
        # Erase conflicted regions
        # Compute conflict mask for each reference view
        conflict_masks = []
        for I_ref_view in I_ref:
            # Eq. (6): Check if reference depth > warped new depth
            conflict_mask = ComputeConflictMask(I_ref_view, I_new_rgbd)
            conflict_masks.append(conflict_mask)
        
        # Final conflict mask is AND of all individual masks
        final_conflict_mask = LogicalAND(conflict_masks)
        
        # Add new view to reference set
        I_ref.add(I_new_rgbd)
        
        # Fine-tune NeRF
        # First 100 iterations: only new view
        # Next 1900 iterations: all reference views
        # Exclude pixels in final_conflict_mask from loss
        f_nerf = FineTuneNeRF(f_nerf, I_ref, iterations=2000, 
                              exclude_mask=final_conflict_mask)
        
    return f_nerf
```

**Edge Cases and Pitfalls:**
*   **Depth Estimation Error:** The initial depth estimation from a single panorama is critical. Errors here propagate to the trajectory generation and warping operations [S1].
*   **Geometry Conflict:** Inpainted regions may generate geometry that conflicts with the existing NeRF structure. The erasing mechanism is essential to prevent this, but if too much is erased, the NeRF may fail to converge [S1].
*   **Warping Accuracy:** The warping function assumes accurate camera poses and depth values. Inaccuracies in depth or pose estimation can lead to false positives in conflict detection [S1].
*   **Stable Diffusion Limitations:** The inpainting model is trained on perspective FOV images, which may not perfectly generalize to panoramic projections, potentially introducing artifacts [S1].

## Operating characteristics
*   **Time Complexity:** The method is iterative, with 32 new views synthesized. Each view requires 2,000 iterations of NeRF fine-tuning. Total training time is approximately 2 hours on a single Nvidia V100 GPU [S1].
*   **Memory Complexity:** Uses Instant-NGP with a hash table of size $2^{19}$ and 16 levels of feature vectors of channel size 2. Memory usage is moderate, fitting within 32GB GPU memory [S1].
*   **Performance:** Evaluated on the Replica dataset (8 scenes) with 2048x1024 resolution. The method produces high-quality panoramic NeRFs from a single input [S1].
*   **Strengths:** Handles single-input reconstruction effectively; resolves geometry conflicts through progressive erasing; leverages powerful 2D generative priors (Stable Diffusion) [S1].
*   **Limitations:** Relies on accurate initial depth estimation; performance depends on the quality of the inpainting model; may struggle with highly dynamic or non-rigid scenes [S1].

## Background
PERF was introduced in the paper "PERF: Panoramic Neural Radiance Field From a Single Panorama" published in IEEE Transactions on Pattern Analysis and Machine Intelligence in 2024 [S1]. The method addresses the challenge of creating panoramic NeRFs from a single input, a task that is ill-posed without additional priors. It builds on the Instant-NGP representation and integrates 2D generative models (Stable Diffusion) with 3D geometric consistency checks. The core innovation is the progressive inpainting-and-erasing strategy, which iteratively expands the viewpoint range while ensuring that inpainted regions do not conflict with the learned geometry [S1].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
