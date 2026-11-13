# Progressive Inpainting-and-Erasing

Also called: Conflicted Geometry Detection and Erasing, Inpainting-and-Erasing Strategy, PERF, PERF Algorithm

## Overview
Progressive Inpainting-and-Erasing (PERF) is a training procedure for generating a panoramic Neural Radiance Field (NeRF) from a single input panorama by iteratively synthesizing novel views and refining the 3D geometry [S1]. It solves the problem of geometry conflicts that arise when inpainting invisible regions with external priors (e.g., diffusion models), which may contradict the existing learned structure [S1]. Unlike standard NeRF training that assumes consistent multi-view observations, PERF explicitly detects and erases conflicted regions from the supervision signal, ensuring that the inpainted content aligns with the established 3D geometry [S1].

## How it works
**Inputs and Outputs:**
- Input: A single input panorama image $I_{rgb}$ [S1].
- Output: A panoramic NeRF representation $f_{nerf}$ [S1].

**Data Structures:**
- **NeRF Representation:** Uses Instant-NGP [S1]. The hash table size is $2^{19}$ with 16 levels of feature vectors, each of channel size 2 [S1].
- **Reference Views Set ($I_{ref}$):** A set containing the initial RGBD panorama and subsequently added inpainted RGBD panoramas [S1].
- **Trajectory Set ($\{p_k\}$):** A set of novel camera poses generated based on the initial depth map [S1].

**Parameters:**
- $\epsilon$: A small threshold value for depth difference comparison [S1].
- $iters$: Total number of progressive sampling iterations [S1].
- Initial training iterations: 10,000 iterations for the single input panorama [S1].
- New view sampling: 32 new positions with approximately gradually increased distance from the original position [S1].
- Fine-tuning iterations per new view: 2,000 iterations total [S1].
    - First 100 iterations: Supervision from only the new reference view [S1].
    - Remaining 1,900 iterations: Supervision from all reference views [S1].
- Optimizer: Adam [S1].
- Learning rate: Initially $10^{-3}$, decayed to $10^{-4}$ with cosine annealing strategy for each learning stage of a newly sampled view [S1].
- Ray sampling: 2,048 rays sampled per iteration for supervision [S1].

**Formulas:**
1.  **Depth Estimation:** The initial depth map $I_d$ is estimated from $I_{rgb}$ using an external depth estimator (Eq. 2 in [S1]). The input becomes $I_{rgbd}$ [S1].
2.  **Single-View NeRF Training:** $f_{nerf}$ is trained using $I_d$ and $I_{rgb}$ (Eq. 3 in [S1]).
3.  **Invisible Region Mask ($m'$):** Used to detect regions not visible from a reference view but present in a new view.
    $$ m'_{ij} = \mathbb{I}(\text{warp}(v_{ij}) - v'_{ij} > \epsilon) $$
    where $v_{ij}$ is the reference depth, $v'_{ij}$ is the warped depth from the new view, $\text{warp}(\cdot)$ is the warping function, $\mathbb{I}(\cdot)$ is the indicator function, and $\epsilon$ is the small threshold [S1]. A value of 1 indicates an invisible region (conflict potential), and 0 indicates visible [S1].
4.  **Collaborative RGBD Inpainting:** Used to fill invisible regions. RGB is inpainted using Stable Diffusion, and depth is stitched using a depth estimator (Eq. 5 in [S1]) [S1].
5.  **Conflicted Region Mask ($m''$):** Used to detect geometry conflicts between the inpainted new view and existing reference views.
    $$ m''_{ij} = \mathbb{I}(v_{ij} - \text{warp}(v'_{ij}) > \epsilon) $$
    where $v_{ij}$ and $v'_{ij}$ are pixel values of the depth maps $I_d$ and $I'_d$ respectively [S1]. A value of 1 indicates inconsistent/conflicted geometry, and 0 indicates consistent [S1].
6.  **Final Conflict Mask:** Computed by performing a logic AND operation across the individual masks $m''$ from all reference views in $I_{ref}$ [S1].

**Steps:**
1.  Estimate the initial depth map $I_d$ of the input single panorama $I_{rgb}$ to obtain $I_{rgbd}$ [S1].
2.  Train a single-view NeRF $f_{nerf}$ using $I_d$ and $I_{rgb}$ [S1].
3.  Generate a trajectory set $\{p_k\}$ of novel views based on the depth map $I_d$ [S1].
4.  Initialize the set of reference views $I_{ref} = \{I_{rgbd}\}$ [S1].
5.  Iterate $n$ from 1 to $iters$:
    a.  Randomly sample a new camera view $p$ from $\{p_k\}$ [S1].
    b.  Compute the invisible regions according to Eq. (4) [S1].
    c.  Perform collaborative RGBD inpainting using Stable Diffusion for RGB and a depth estimator for depth stitching (Eq. 5) [S1].
    d.  For all $I_{ref} \in I_{ref}$:
        i.  Compute the conflicted regions according to Eq. (6) [S1].
        ii. Erase these conflicted regions from the supervision of the NeRF training [S1].
    e.  Add the new inpainted view to the reference views $I_{ref}$ [S1].
    f.  Fine-tune $f_{nerf}$ with the updated $I_{ref}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_inpainting_and_erasing(I_rgb):
    # 1. Estimate depth and initialize NeRF
    I_d = estimate_depth(I_rgb) # Eq. 2 in [S1]
    I_rgbd = combine(I_rgb, I_d)
    f_nerf = train_single_view_neRF(I_rgbd) # Eq. 3 in [S1]
    
    # 2. Generate trajectories
    p_k = generate_trajectories(I_d)
    
    # 3. Initialize reference views
    I_ref = {I_rgbd}
    
    # 4. Progressive loop
    for n in range(1, iters + 1):
        # Sample new view
        p = random_sample(p_k)
        
        # Render depth from current NeRF for the new view p
        # Note: The source implies warping between reference and new view depths.
        # We assume we can render depth from f_nerf for any pose.
        v_new = render_depth(f_nerf, p)
        
        # Compute invisible regions (Eq. 4)
        # Warp new view depth to reference view coordinates
        # For simplicity, assume we pick one reference view or aggregate.
        # The algorithm description says "Compute the invisible regions according to Eq. 4"
        # before inpainting. This likely uses the current NeRF's geometry.
        # Let's assume we warp v_new to the coordinate system of the reference views
        # to find what is occluded.
        # However, Eq 4 uses v_ij (reference) and v'_ij (warped new).
        # In the context of generating a NEW view, we need to know what parts of the
        # new view are "invisible" relative to the existing reference views to inpaint them.
        # The source says: "Compute the invisible regions according to Eq. 4 and perform 
        # collaborative RGBD inpainting".
        
        # Let's assume we have a reference depth map v_ref from I_ref (e.g., the first one or aggregated)
        # And we warp the new view's depth v_new to the reference view.
        # If warp(v_new) > v_ref + epsilon, then the new view sees something behind the reference surface -> Invisible in reference?
        # Actually, Eq 4: m'_ij = I(warp(v_ij) - v'_ij > epsilon).
        # v_ij is reference depth. v'_ij is warped new view depth.
        # If warped new depth is significantly smaller than reference depth, it means the new view sees something closer (visible).
        # If warped new depth is larger, it means the new view sees behind the reference surface (invisible in reference).
        # So m' marks invisible regions in the new view relative to reference.
        
        v_ref = get_depth_from_reference(I_ref) # Representative reference depth
        v_new_warped = warp(v_new, pose_ref, pose_new)
        m_invisible = indicator(warp(v_ref, pose_new, pose_ref) - v_new > epsilon) # Logic check on Eq 4 direction
        
        # Perform collaborative RGBD inpainting (Eq. 5)
        # Inpaint RGB using Stable Diffusion for regions in m_invisible
        # Inpaint/Estimate Depth using depth estimator for regions in m_invisible
        I_new_rgbd = collaborative_rgb_inpainting(I_new_rgb, I_new_depth, m_invisible)
        
        # Compute conflicted regions (Eq. 6)
        # This step ensures the inpainted geometry doesn't conflict with existing references.
        # We compare the inpainted new view against all existing reference views.
        m_conflict_final = None
        for I_ref_view in I_ref:
            v_ref_view = get_depth(I_ref_view)
            v_new_warped_to_ref = warp(I_new_rgbd.depth, pose_new, pose_ref_view)
            m_conflict = indicator(v_ref_view - v_new_warped_to_ref > epsilon) # Eq. 6
            if m_conflict_final is None:
                m_conflict_final = m_conflict
            else:
                m_conflict_final = logical_and(m_conflict_final, m_conflict)
        
        # Erase conflicted regions from supervision
        # The loss for these pixels is ignored or masked out during fine-tuning.
        I_new_rgbd.masked_conflict(m_conflict_final)
        
        # Add to reference views
        I_ref.add(I_new_rgbd)
        
        # Fine-tune NeRF
        # First 100 iters: only new view
        # Next 1900 iters: all reference views
        fine_tune_neRF(f_nerf, I_ref, iterations=2000, lr_schedule=cosine_anneal(1e-3, 1e-4))
        
    return f_nerf
```

**Edge Cases and Pitfalls:**
- **Precision:** The threshold $\epsilon$ must be carefully chosen to allow small errors while detecting significant geometry conflicts [S1].
- **Boundary Conditions:** The warping function must handle occlusions and depth discontinuities correctly to avoid false conflict detection [S1].
- **Degenerate Input:** The method relies on a reasonable initial depth estimate; poor depth estimation may lead to incorrect trajectory generation and conflict detection [S1].

## Subtopics

### Collaborative RGBD Inpainting
A sub-method used within PERF to fill invisible regions. It uses Stable Diffusion for RGB and a depth estimator for depth, guided by the invisible region mask. It is chosen when external priors are needed to complete missing geometry [S1].

### Single-View NeRF Training
The initialization step where a NeRF is trained from a single RGBD image. It is chosen to establish an initial 3D geometry before progressive expansion [S1].

## Operating characteristics
- **Time Complexity:** Training a scene takes about 2 hours on a single Nvidia V100 GPU with 32G memory [S1]. The complexity is dominated by the iterative fine-tuning and the external model inference (Stable Diffusion, depth estimator) for each of the 32 new views [S1].
- **Memory Complexity:** Uses Instant-NGP with a hash table size of $2^{19}$ and 16 levels of feature vectors of channel size 2 [S1].
- **Scaling:** The method scales to 32 new views with gradually increased distance from the original position [S1].
- **Strengths:** Effectively avoids geometry conflicts between visible and invisible regions, allowing for consistent panoramic NeRF generation from a single input [S1].
- **Limitations:** Relies on external models (Stable Diffusion, depth estimator) which may introduce artifacts or inconsistencies if not properly constrained by the erasing mechanism [S1].
- **When to use:** When generating a panoramic NeRF from a single input panorama is required, and multi-view data is unavailable [S1].
- **When not to use:** When high-fidelity multi-view data is available, as standard NeRF training would be more efficient and accurate [S1].

## Background
- **Origin:** Proposed in the paper "PERF: Panoramic Neural Radiance Field From a Single Panorama" [S1].
- **History:** The method builds on the concept of Neural Radiance Fields [S1] and addresses the specific challenge of single-view panorama generation, which is difficult due to the lack of multi-view consistency [S1].
- **Usage:** Evaluated on the Replica dataset and in-the-wild data for qualitative evaluation [S1].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
