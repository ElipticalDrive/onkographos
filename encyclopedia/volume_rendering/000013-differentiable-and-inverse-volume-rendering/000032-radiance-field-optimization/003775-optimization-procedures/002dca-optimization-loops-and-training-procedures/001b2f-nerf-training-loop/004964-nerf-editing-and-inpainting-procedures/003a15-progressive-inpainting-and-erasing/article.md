# Progressive Inpainting-and-Erasing

Also called: Conflicted Geometry Detection and Erasing, Inpainting-and-Erasing Strategy, PERF, PERF Algorithm

## Overview
Progressive Inpainting-and-Erasing (PERF) is a training procedure for expanding a single-view Neural Radiance Field (NeRF) into a complete 3D representation by iteratively synthesizing novel views and refining the geometry. It solves the problem of geometry conflicts that arise when inpainting invisible regions with external generative models, which may violate the consistency constraints of the already-learned visible geometry [S1]. Unlike standard NeRF training which assumes multi-view consistency, PERF explicitly detects and erases conflicting regions from the supervision loss, allowing the model to converge to a coherent 3D structure from a single input [S1].

## How it works
The method operates on a pre-trained single-view NeRF $f_{\text{nerf}}$ initialized from a single input panorama. It uses a hash-grid based representation (Instant-NGP) and iteratively samples new camera poses, inpaints the resulting views, detects geometric inconsistencies, and fine-tunes the NeRF [S1].

### Inputs and Outputs
*   **Input:** A single RGB panorama image $I_{\text{rgb}}$.
*   **Output:** A panoramic NeRF $f_{\text{nerf}}$ representing the complete 3D scene.

### Data Structures
*   **NeRF Model ($f_{\text{nerf}}$):** An Instant-NGP implementation.
    *   **Hash Table:** Size $2^{19}$, 16 levels, feature vector channel size 2 [S1].
    *   **Parameters:** $\Theta$ (weights of the MLP and hash grid).
*   **Reference Views Set ($I_{\text{ref}}$):** A set of RGB-D images. Initially contains the estimated depth map of the input panorama.
*   **Trajectory Set ($\{p_k\}$):** A set of candidate novel camera poses generated based on the initial depth map, with positions gradually increasing in distance from the original view [S1].

### Formulas
The rendering equation remains the standard volume rendering integral [S1, 12372]:
$$
\hat{C}(r) = \sum_{i=1}^{N} T(t_i)(1 - \exp(-\sigma(t_i)\delta_i))c(t_i)
$$
where $T(t_i) = \exp(-\sum_{j=1}^{i-1} \sigma(t_j)\delta_j)$ [S1, 12372].

**Depth Estimation:**
The initial depth map $I_d$ is estimated from $I_{\text{rgb}}$ using an external depth estimator (Eq. 2 in [S1]). The initial RGB-D image is $I_{\text{rgbd}} = (I_{\text{rgb}}, I_d)$ [S1].

**Invisible Region Mask ($m'$):**
For a new view $p'$, depth $I'_d$ is rendered. This depth is warped to the reference view coordinate system. The mask $m'$ identifies pixels where the warped depth is significantly larger than the reference depth, indicating invisible regions in the reference view:
$$
m'_{ij} = \mathbb{I}(\text{warp}(v'_{ij}) - v_{ij} > \epsilon)
$$
where $v_{ij}$ and $v'_{ij}$ are the depth values at pixel $(i,j)$ in the reference and warped new view respectively, $\epsilon$ is a small threshold, and $\mathbb{I}(\cdot)$ is the indicator function [S1].

**Collaborative RGBD Inpainting:**
The invisible regions in the new view are inpainted using Stable Diffusion for RGB and a depth estimator for depth stitching (Eq. 5 in [S1]) [S1].

**Conflicted Region Mask ($m''$):**
To detect geometry conflicts, the depth of the *inpainted* new view is warped back to the reference view. The mask $m''$ identifies pixels where the reference depth is significantly larger than the warped inpainted depth, indicating that the inpainted content violates the existing geometry:
$$
m''_{ij} = \mathbb{I}(v_{ij} - \text{warp}(v'_{ij}) > \epsilon)
$$
where $v_{ij}$ is the reference depth and $v'_{ij}$ is the warped depth from the inpainted new view [S1].

**Final Conflict Mask:**
The final conflict mask is computed by performing a logic AND operation across the individual conflict masks $m''$ from all views in $I_{\text{ref}}$ [S1].

### Parameters
*   **$\epsilon$:** A small value used as the threshold for depth difference in mask computation [S1].
*   **Hash Table Size:** $2^{19}$ with 16 levels of feature vectors of channel size 2 [S1].
*   **Initial Training Iterations:** 10,000 iterations for the single input panorama [S1].
*   **New View Sampling:** 32 new positions are sampled with approximately gradually increased distance from the original position [S1].
*   **Fine-tuning Iterations:** 2,000 iterations per new view.
    *   First 100 iterations: Supervision only from the new reference view [S1].
    *   Remaining 1,900 iterations: Supervision from all reference views in $I_{\text{ref}}$ [S1].
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** Initially $10^{-3}$, decayed to $10^{-4}$ with cosine annealing for each learning stage of a newly sampled view [S1].
*   **Ray Sampling:** 2,048 rays sampled per iteration for supervision [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_inpainting_and_erasing(I_rgb):
    """
    Implements the PERF algorithm [S1].
    """
    # 1. Initial Depth Estimation and Single-View Training
    I_d = estimate_depth(I_rgb)  # Eq. (2) in [S1]
    I_rgbd = (I_rgb, I_d)
    
    # Initialize NeRF with Instant-NGP (Hash table 2^19, 16 levels, channel 2) [S1]
    f_nerf = initialize_instant_ngp()
    
    # Train single-view NeRF
    train_neRF(f_nerf, [I_rgbd], iterations=10000, optimizer="Adam", lr=1e-3)
    
    # 2. Generate Trajectories
    # Generate 32 new positions with gradually increased distance [S1]
    trajectories = generate_trajectories(I_d, num_views=32)
    
    # 3. Initialize Reference Views
    I_ref = {I_rgbd}
    
    # 4. Progressive Loop
    for n in range(1, 33):  # iters = 32 based on [S1]
        # Randomly sample a new camera view p from trajectories
        p = random_sample(trajectories)
        
        # Render initial depth from current NeRF for the new view
        I_d_prime = render_depth(f_nerf, p)
        
        # Compute invisible regions according to Eq. (4) [S1]
        # Warp depth from new view to reference view coordinates
        I_d_prime_warped = warp_depth(I_d_prime, p, reference_view=p) # Simplified: warp to ref
        # Note: In practice, warping is done between specific view pairs.
        # Eq 4: m' = I(warp(v') - v > epsilon)
        m_invisible = compute_invisible_mask(I_d_prime_warped, I_d, epsilon=epsilon)
        
        # Perform collaborative RGBD inpainting using Stable Diffusion and depth estimator [S1]
        # Eq. (5) in [S1]
        I_rgb_inpainted, I_d_inpainted = collaborative_rgb_d_inpainting(
            I_rgb=p.rgb, 
            I_d=I_d_prime, 
            mask=m_invisible
        )
        I_new_rgbd = (I_rgb_inpainted, I_d_inpainted)
        
        # Compute conflicted regions for all reference views [S1]
        # Final mask is logic AND of individual masks
        m_conflict_final = None
        
        for I_ref_view in I_ref:
            # Warp depth from inpainted new view to reference view
            I_d_new_warped_to_ref = warp_depth(I_d_inpainted, p, reference_view=I_ref_view)
            # Eq. (6): m'' = I(v - warp(v') > epsilon)
            m_conflict = compute_conflict_mask(I_ref_view.depth, I_d_new_warped_to_ref, epsilon=epsilon)
            
            if m_conflict_final is None:
                m_conflict_final = m_conflict
            else:
                # Logic AND operation across all reference views [S1]
                m_conflict_final = logical_and(m_conflict_final, m_conflict)
        
        # Erase conflicted regions from supervision
        # The loss is computed only on pixels where m_conflict_final == 0
        # Add the new view to reference views
        I_ref.add(I_new_rgbd)
        
        # Fine-tune NeRF with updated I_ref [S1]
        # First 100 iterations: only new view
        train_neRF(f_nerf, [I_new_rgbd], iterations=100, optimizer="Adam", lr=1e-3)
        # Remaining 1900 iterations: all reference views
        train_neRF(f_nerf, list(I_ref), iterations=1900, optimizer="Adam", lr=1e-3)
        
    return f_nerf

def compute_invisible_mask(I_d_prime_warped, I_d_ref, epsilon):
    """
    Computes mask m' indicating invisible regions in the reference view.
    Eq. (4): m' = I(warp(v') - v > epsilon)
    """
    diff = I_d_prime_warped - I_d_ref
    return (diff > epsilon).astype(float)

def compute_conflict_mask(I_d_ref, I_d_new_warped, epsilon):
    """
    Computes mask m'' indicating conflicted regions.
    Eq. (6): m'' = I(v - warp(v') > epsilon)
    """
    diff = I_d_ref - I_d_new_warped
    return (diff > epsilon).astype(float)
```

### Edge Cases and Pitfalls
*   **Depth Estimation Error:** The initial depth map $I_d$ is critical. Errors here propagate to trajectory generation and warping accuracy [S1].
*   **Warping Accuracy:** The warping function must accurately project depth maps between views. Misalignment can lead to false positives in conflict detection [S1].
*   **Inpainting Artifacts:** Stable Diffusion may generate plausible but geometrically inconsistent content. The erasing mechanism is essential to prevent these artifacts from corrupting the NeRF geometry [S1].
*   **Convergence:** The method relies on iterative fine-tuning. Insufficient iterations per view may lead to incomplete learning, while excessive iterations may overfit to the inpainted views [S1].

## Subtopics

### Collaborative RGBD Inpainting
A sub-procedure that uses Stable Diffusion for RGB inpainting and a depth estimator for depth stitching to fill invisible regions. It sets apart from simple RGB inpainting by maintaining depth consistency. Choose when geometry-aware completion is needed. [S1]

### Single-View NeRF Training
The initialization step using a single panorama and estimated depth. It sets apart from multi-view NeRF training by requiring depth estimation and lacking multi-view constraints initially. Choose when only a single view is available. [S1]

## Operating characteristics
*   **Time Complexity:** The method is iterative. For $N$ novel views, each requiring fine-tuning for 2,000 iterations, the total training time is proportional to $N \times 2000$ iterations. A typical scene takes about 2 hours on a single Nvidia V100 GPU [S1].
*   **Memory Complexity:** Depends on the Instant-NGP hash table size ($2^{19}$) and the number of reference views stored. The hash table uses 16 levels with channel size 2 [S1].
*   **Scaling:** The method scales with the number of sampled views (32 in the source). Performance may degrade if the scene has large unobserved regions or complex geometry not captured by the initial depth estimate [S1].
*   **Strengths:** Effectively handles geometry conflicts from generative inpainting. Can reconstruct 3D scenes from a single panorama.
*   **Limitations:** Relies on external models (Stable Diffusion, depth estimator) which may introduce artifacts. Computationally expensive due to iterative training and rendering.
*   **When to use:** When reconstructing a 3D scene from a single panoramic image is required, and multi-view data is unavailable.
*   **When not to use:** When high-fidelity multi-view data is available, or when real-time performance is critical.

## Background
The Progressive Inpainting-and-Erasing (PERF) method was introduced in the paper "PERF: Panoramic Neural Radiance Field From a Single Panorama" [S1]. It addresses the challenge of extending NeRFs from a single view by combining generative inpainting with explicit geometry conflict detection. The method builds on the Instant-NGP representation [S1] and uses Stable Diffusion for inpainting [S1].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
