# Progressive Inpainting-and-Erasing

Also called: Conflicted Geometry Detection and Erasing, Inpainting-and-Erasing Strategy, PERF, PERF Algorithm

## Overview
Progressive Inpainting-and-Erasing (PERF) is a training procedure for synthesizing a panoramic Neural Radiance Field (NeRF) from a single input panorama. It solves the problem of incomplete geometry and texture in single-view reconstruction by iteratively generating novel views, inpainting missing regions, and erasing geometrically inconsistent areas. It differs from sibling methods like [Deceptive-NeRF/3DGS Pipeline](11860-deceptive-nerf-3dgs-pipeline.md) or [DFF](12068-dff-decomposing-nerf-for-editing-via-feature-field-distillat.md) by explicitly managing the conflict between visible and invisible regions through a progressive loop of view synthesis and geometric consistency masking, rather than relying solely on multi-view consistency or feature distillation.

## How it works
**Inputs and Outputs:**
*   **Input:** A single input panorama image $I_{rgb}$ with resolution $W \times H$.
*   **Output:** A trained panoramic NeRF representation $f_{nerf}$ mapping 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to density $\sigma$ and color $\mathbf{c}$.

**Data Structures:**
*   **NeRF Representation ($f_{nerf}$):** Uses the Instant-NGP [S1] implementation.
    *   **Hash Grid:** A hash table of size $2^{19}$ with 16 levels.
    *   **Feature Vectors:** Each entry in the hash grid stores a feature vector of channel size 2 [S1].
*   **Reference Views Set ($I_{ref}$):** A set containing RGB-D images. Initially contains the estimated depth map and input panorama.
*   **Trajectory Set ($\{p_k\}$):** A set of novel camera poses generated based on the initial depth map.

**Formulas:**
1.  **Initial Depth Estimation:** The depth map $I_d$ is estimated from $I_{rgb}$ using an external depth estimator (Eq. 2 in [S1]). The initial RGB-D image is $I_{rgbd} = (I_{rgb}, I_d)$.
2.  **Single-View NeRF Training:** The initial NeRF $f_{nerf}$ is trained using $I_{rgbd}$ based on the volume rendering equation (Eq. 3 in [S1]).
3.  **Invisible Region Mask ($m'$):** For a new view $p'$, depth $I'_d$ is rendered. The depth is warped to the reference view coordinate system. The mask $m'$ identifies pixels where the warped depth is significantly larger than the reference depth, indicating invisible regions [S1]:
    $$ m'_{ij} = \mathbb{I}(warp(v'_{ij}) - v_{ij} > \epsilon) $$
    where $v_{ij}$ is the reference depth, $v'_{ij}$ is the warped depth, $\epsilon$ is a small threshold, and $\mathbb{I}(\cdot)$ is the indicator function.
4.  **Collaborative RGBD Inpainting:** Invisible regions are inpainted using Stable Diffusion for RGB and a depth estimator for depth stitching (Eq. 5 in [S1]).
5.  **Conflicted Region Mask ($m''$):** For each reference view $I_{ref} \in I_{ref}$, the inpainted new view is warped back to the reference view. The mask $m''$ identifies pixels where the geometry is inconsistent (i.e., the inpainted depth is larger than the reference depth, implying the inpainted region occludes existing visible geometry) [S1]:
    $$ m''_{ij} = \mathbb{I}(v_{ij} - warp(v'_{ij}) > \epsilon) $$
    where $v_{ij}$ is the reference depth and $v'_{ij}$ is the warped depth from the inpainted new view.
6.  **Final Conflict Mask:** The final mask is computed by performing a logic AND operation across the individual $m''$ masks from all reference views [S1].

**Parameters:**
*   **$\epsilon$:** A small value used as the threshold for depth difference in warping checks [S1].
*   **Hash Table Size:** $2^{19}$ [S1].
*   **Levels:** 16 [S1].
*   **Channel Size:** 2 [S1].
*   **Initial Training Iterations:** 10,000 iterations for the single input panorama [S1].
*   **New View Sampling:** 32 new positions with approximately gradually increased distance from the original position [S1].
*   **Fine-tuning Iterations per View:** 2,000 iterations total [S1].
    *   First 100 iterations: Supervision from the new reference view only.
    *   Remaining 1,900 iterations: Supervision from all reference views.
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** Initially $10^{-3}$, decayed to $10^{-4}$ with cosine annealing for each learning stage of a newly sampled view [S1].
*   **Ray Sampling:** 2,048 rays sampled per iteration for supervision [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_inpainting_and_erasing(I_rgb):
    """
    Implements the PERF algorithm [S1].
    """
    # 1. Estimate initial depth and RGB-D
    I_d = estimate_depth(I_rgb) # Eq. 2 in [S1]
    I_rgbd = (I_rgb, I_d)
    
    # 2. Train initial single-view NeRF
    f_nerf = train_single_view_nerf(I_rgbd) # Eq. 3 in [S1]
    
    # 3. Generate trajectory of novel views
    p_k = generate_trajectory(I_d)
    
    # 4. Initialize reference views
    I_ref = {I_rgbd}
    
    # 5. Progressive loop
    for n in range(1, iters + 1):
        # Randomly sample a new camera view
        p = random_sample(p_k)
        
        # Render depth from new view
        I_prime_d = render_depth(f_nerf, p)
        
        # Compute invisible regions (Eq. 4)
        # Warp new view depth to reference view coordinates
        I_prime_d_warped = warp(I_prime_d, p, reference_view_pose)
        m_prime = indicator_function(I_prime_d_warped - I_ref_depth > epsilon)
        
        # Perform collaborative RGBD inpainting (Eq. 5)
        # Use Stable Diffusion for RGB and depth estimator for depth
        I_prime_rgb_inpainted = inpaint_rgb(I_prime_rgb, m_prime, StableDiffusion)
        I_prime_d_inpainted = inpaint_depth(I_prime_d, m_prime, DepthEstimator)
        I_prime_rgbd = (I_prime_rgb_inpainted, I_prime_d_inpainted)
        
        # Compute conflicted regions for all reference views
        final_conflict_mask = None
        for I_ref_view in I_ref:
            # Warp inpainted new view to reference view
            I_prime_d_warped_back = warp(I_prime_d_inpainted, p, I_ref_view.pose)
            # Eq. 6: Detect geometry conflict
            m_double_prime = indicator_function(I_ref_view.depth - I_prime_d_warped_back > epsilon)
            
            if final_conflict_mask is None:
                final_conflict_mask = m_double_prime
            else:
                # Logic AND operation across all reference views
                final_conflict_mask = logical_and(final_conflict_mask, m_double_prime)
        
        # Erase conflicted regions from supervision
        # The supervision mask for the new view is ~final_conflict_mask
        # (i.e., do not use pixels marked as conflicted)
        
        # Add new view to reference views
        I_ref.add(I_prime_rgbd)
        
        # Fine-tune NeRF
        # First 100 iterations: only new view
        fine_tune(f_nerf, I_prime_rgbd, iterations=100, mask=~final_conflict_mask)
        # Remaining 1900 iterations: all reference views
        fine_tune(f_nerf, I_ref, iterations=1900, mask=~final_conflict_mask)
        
    return f_nerf
```

**Edge Cases and Pitfalls:**
*   **Depth Estimation Error:** The initial depth estimation from a single panorama is critical. Errors here propagate to the trajectory generation and warping accuracy [S1].
*   **Geometry Conflict:** The method explicitly handles conflicts where inpainted regions occlude existing visible geometry. If the threshold $\epsilon$ is too small, valid geometry might be erased; if too large, conflicting geometry might remain [S1].
*   **Stable Diffusion Limitations:** The inpainting model is trained on perspective FOV images, which may not perfectly generalize to panoramic projections, potentially introducing artifacts [S1].

## Operating characteristics
*   **Time Complexity:** The method is iterative. Each iteration involves depth estimation, warping, inpainting (using Stable Diffusion, which is computationally expensive), and NeRF fine-tuning. Training a scene takes about 2 hours on a single Nvidia V100 GPU [S1].
*   **Memory Complexity:** Depends on the Instant-NGP hash grid size ($2^{19}$ entries) and the storage of reference views. The GPU memory usage is reported as 32G for the experiments [S1].
*   **Strengths:** Effectively reconstructs complete panoramic NeRFs from a single input by progressively filling in missing geometry and texture while maintaining consistency [S1].
*   **Limitations:** Relies on external models (depth estimator, Stable Diffusion) which may introduce errors. The process is slow due to the iterative nature and the use of diffusion models [S1].
*   **When to use:** When only a single panorama is available and a complete 3D representation is required.
*   **When not to use:** When multiple views are available (multi-view stereo is more efficient) or when real-time performance is critical.

## Background
The Progressive Inpainting-and-Erasing (PERF) method was introduced in the paper "PERF: Panoramic Neural Radiance Field From a Single Panorama" [S1]. It addresses the challenge of single-view panorama reconstruction, which is ill-posed due to missing information. The method builds on the Instant-NGP representation [S1] and integrates generative inpainting techniques to complete the scene. It is part of the broader family of NeRF editing and inpainting procedures [S1].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
