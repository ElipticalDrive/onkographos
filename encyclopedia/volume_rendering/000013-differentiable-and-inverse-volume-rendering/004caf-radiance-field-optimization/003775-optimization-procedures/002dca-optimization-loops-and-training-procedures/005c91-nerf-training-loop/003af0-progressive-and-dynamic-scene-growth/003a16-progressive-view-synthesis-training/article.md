# Progressive View Synthesis Training

Also called: PERF training loop, collaborative RGBD inpainting and progressive inpainting-and-erasing

## Overview
Progressive View Synthesis Training (PERF) is a training strategy for Panoramic Neural Radiance Fields (NeRF) that synthesizes novel views from a single input panorama by progressively adding reference views to improve geometry and semantic coherence in occluded regions [S1]. It sets itself apart from sibling methods like [Dynamic Growth Parameters](2968-dynamic-growth-parameters.md) by explicitly integrating a collaborative RGBD inpainting and progressive inpainting-and-erasing mechanism to resolve geometry conflicts and semantic inconsistencies that arise when expanding the view set [S1]. Unlike [Incremental Registration](11341-incremental-registration.md) which focuses on pose alignment, PERF focuses on content completion and geometric regularization during the view expansion process [S1].

## How it works
The method trains a NeRF representation (specifically using Instant-NGP) by starting with a single panorama and iteratively introducing new synthetic reference views. Each iteration involves inpainting missing content, erasing conflicting geometry, and re-optimizing the network with a staged learning rate and supervision schedule.

### Inputs and Outputs
*   **Input:** A single input RGB-D panorama (color and depth maps).
*   **Output:** A trained NeRF model capable of rendering novel views (RGB and depth) from arbitrary camera poses sampled around the scene.

### Data Structures
The implementation relies on the Instant-NGP representation via the NeRFAcc framework [S1].
*   **Hash Grid:** A hash table of size $2^{19}$ with 16 levels of feature vectors. Each feature vector has a channel size of 2 [S1].
*   **Ray Buffer:** At each iteration, 2048 rays are randomly sampled. Each ray is associated with a ground-truth or inpainted color value and a depth value for supervision [S1].

### Parameters
*   **Initial Training Iterations:** 10,000 iterations on the single input panorama [S1].
*   **Progressive Views:** 32 new positions are sampled with gradually increased distance from the original reference view position [S1].
*   **Iterations per New View:** 2,000 iterations [S1].
*   **Supervision Schedule:**
    *   First 100 iterations: Supervision using only the new reference view [S1].
    *   Remaining 1,900 iterations: Supervision using all reference views (original + newly added) [S1].
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** Initial $10^{-3}$, decaying to $10^{-4}$ using cosine annealing for each new view stage [S1].
*   **Distortion Loss Weight ($\lambda_{dist}$):** 1 [S1].
*   **Inpainting Model:** Pre-trained Stable Diffusion 2 [S1].

### Numbered Steps

1.  **Initial Training:** Train the NeRF model using the single input panorama for 10,000 iterations [S1].
2.  **View Sampling:** Progressively sample 32 new camera positions. These positions are distributed with approximately gradually increased distance from the original reference panorama view position [S1].
3.  **Iterative Expansion Loop:** For each newly added reference view position:
    a. **Collaborative RGBD Inpainting:** Generate a synthetic reference view for the new position. Since the inpainting model (Stable Diffusion 2) is trained on perspective FOV and cannot handle panoramic distortion directly, the following projection strategy is used [S1]:
        i. Identify masked/occluded regions in the target panoramic view.
        ii. Project these masked regions to perspective views.
        iii. Apply inpainting using the pre-trained Stable Diffusion 2 model on the perspective views.
        iv. Project the inpainted regions back to the panoramic coordinate system.
    b. **Progressive Inpainting-and-Erasing:** Apply an erasing strategy to remove conflicting geometry from previous views that contradicts the new inpainted content. This prevents wrong redundant geometry and degraded RGB quality [S1].
    c. **NeRF Re-optimization:** Continue training the NeRF for 2,000 iterations [S1].
        i. **Phase 1 (Iterations 1–100):** Use only the new reference view (color and depth) for supervision.
        ii. **Phase 2 (Iterations 101–2000):** Use all accumulated reference views (original + all newly added) for supervision.
        iii. At each iteration, randomly sample 2048 rays. The supervision targets are the ground-truth or inpainted colors and depth values for these rays [S1].
        iv. Optimize NeRF parameters using Adam with the learning rate schedule ($10^{-3} \to 10^{-4}$ via cosine annealing) [S1].
        v. Apply distortion loss with weight $\lambda_{dist} = 1$ to encourage volume densities to converge to valid surfaces, avoiding "foggy effects" or "floating" densities [S1].
4.  **Completion:** Repeat Step 3 for all 32 sampled positions.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
from torch import optim
from nerfacc_framework import InstantNGP, render_rays, sample_rays
from inpainting_model import StableDiffusionInpaint

def progressive_view_synthesis_training(single_panorama_rgb, single_panorama_depth):
    """
    Trains a NeRF from a single panorama using progressive view synthesis.
    
    Args:
        single_panorama_rgb: Tensor, RGB panorama image.
        single_panorama_depth: Tensor, Depth panorama image.
        
    Returns:
        trained_nerf: The trained Instant-NGP model.
    """
    # 1. Initialize NeRF (Instant-NGP)
    # Hash table size 2^19, 16 levels, channel size 2 [S1]
    nerf = InstantNGP(hash_size=2**19, num_levels=16, channel_size=2)
    
    # 2. Initial Training on Single Panorama
    optimizer = optim.Adam(nerf.parameters(), lr=1e-3)
    for iteration in range(10000):
        rays = sample_rays(2048) # Randomly sample 2048 rays [S1]
        # Render colors and depths
        rendered_colors, rendered_depths = render_rays(nerf, rays)
        
        # Supervision with ground truth
        gt_colors, gt_depths = get_gt_for_rays(rays, single_panorama_rgb, single_panorama_depth)
        
        loss = compute_loss(rendered_colors, gt_colors) + \
               compute_loss(rendered_depths, gt_depths) + \
               1.0 * compute_distortion_loss(nerf, rays) # lambda_dist = 1 [S1]
        
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

    # 3. Progressive View Sampling
    # Sample 32 new positions with gradually increased distance [S1]
    new_view_positions = sample_positions(32, initial_pose=single_panorama_pose)
    
    # Store all reference views for supervision
    all_reference_views = [single_panorama_rgb]
    all_reference_depths = [single_panorama_depth]
    
    inpainter = StableDiffusionInpaint() # Pre-trained Stable Diffusion 2 [S1]

    for view_idx, new_pos in enumerate(new_view_positions):
        # Learning rate schedule for this stage [S1]
        scheduler = CosineAnnealingLR(optimizer, T_max=2000, eta_min=1e-4)
        
        # Generate synthetic reference view for the new position
        # This involves collaborative RGBD inpainting and erasing [S1]
        synthetic_rgb, synthetic_depth = generate_synthetic_view(
            nerf, new_pos, inpainter, all_reference_views, all_reference_depths
        )
        
        # Add to reference set
        all_reference_views.append(synthetic_rgb)
        all_reference_depths.append(synthetic_depth)
        
        # Re-optimization for 2000 iterations [S1]
        for iter_in_stage in range(2000):
            rays = sample_rays(2048) # Sample 2048 rays [S1]
            
            # Determine supervision source based on iteration count [S1]
            if iter_in_stage < 100:
                # First 100 iterations: only new view
                target_rgb = synthetic_rgb
                target_depth = synthetic_depth
            else:
                # Remaining 1900 iterations: all views
                # Note: Rays must be mapped to the appropriate view in the set
                # For simplicity, assume ray sampling covers the union of views
                # or specific rays are assigned to specific views.
                # The source says "supervise NeRF with all the reference views" [S1]
                # Implementation detail: Aggregate losses from all views for the sampled rays
                target_rgb, target_depth = get_multi_view_targets(rays, all_reference_views, all_reference_depths)
            
            rendered_colors, rendered_depths = render_rays(nerf, rays)
            
            loss = compute_loss(rendered_colors, target_rgb) + \
                   compute_loss(rendered_depths, target_depth) + \
                   1.0 * compute_distortion_loss(nerf, rays)
            
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
            scheduler.step()

    return nerf

def generate_synthetic_view(nerf, camera_pose, inpainter, ref_rgbs, ref_depths):
    """
    Generates a synthetic view using inpainting and erasing.
    1. Project occluded regions to perspective.
    2. Inpaint using Stable Diffusion.
    3. Project back to panorama.
    4. Erase conflicting geometry.
    """
    # Render initial guess from NeRF
    initial_render = render_view(nerf, camera_pose)
    
    # Identify masked/occluded regions (e.g., using depth consistency or visibility)
    mask = identify_occluded_regions(initial_render, ref_rgbs, ref_depths)
    
    # Project masked regions to perspective views
    perspective_patches = project_to_perspective(mask, initial_render)
    
    # Inpaint perspective patches
    inpainted_patches = inpainter.inpaint(perspective_patches)
    
    # Project back to panorama
    inpainted_panorama = project_back_to_perspective(inpainted_patches, mask)
    
    # Apply progressive inpainting-and-erasing to resolve conflicts
    final_rgb, final_depth = apply_erasing_strategy(inpainted_panorama, initial_render)
    
    return final_rgb, final_depth

def compute_distortion_loss(nerf, rays):
    """
    Encourages volume densities to converge to a valid surface.
    Uses the distortion loss from [4] with weight 1 [S1].
    """
    # Implementation of distortion loss as defined in the Instant-NGP paper [4]
    # This prevents "foggy effects" and "floating" densities [S1]
    return 0.0 # Placeholder for specific distortion loss formula from [4]
```

### Edge Cases and Pitfalls
*   **Panoramic Distortion:** Inpainting models trained on perspective FOV (like Stable Diffusion 2) cannot be directly applied to panoramas due to distortion. The method must project masked regions to perspective views, inpaint, and project back [S1].
*   **Foggy Effects/Floating Densities:** NeRF can produce invalid surfaces ("foggy effects") if not regularized. This is mitigated by using distortion loss with weight $\lambda_{dist} = 1$ [S1].
*   **Geometry Conflicts:** Without the inpainting-and-erasing strategy, geometry conflicts arise from different views, leading to wrong redundant geometry and degraded RGB quality [S1].

## Operating characteristics
*   **Time Complexity:** Training a scene takes approximately 2 hours on a single Nvidia V100 GPU with 32G memory [S1]. The complexity is dominated by the iterative inpainting and re-optimization steps for each of the 32 views.
*   **Memory Complexity:** Depends on the Instant-NGP hash grid size ($2^{19}$ entries) and the storage of reference views. The V100 with 32G memory is sufficient for the reported scenes [S1].
*   **Performance:** On the Replica dataset, PERF achieves the best performance in PSNR, SSIM, and LPIPS compared to DS-NeRF, DietNeRF, SinNeRF, 360FusionNeRF, and Omni-NeRF [S1]. Specifically, it shows significant improvement in masked PSNR, indicating better inpainting of invisible regions [S1].
*   **Strengths:** Superior semantic coherence in occluded regions compared to methods using CLIP or DINO regularization (DietNeRF, SinNeRF) [S1]. Better visual quality than Omni-NeRF which relies on pseudo ground truth from point clouds [S1].
*   **Limitations:** Relies on a heavy pre-trained inpainting model (Stable Diffusion 2), which may limit real-time applicability. The method is designed for single panorama input, limiting its direct use with multi-view sparse inputs without adaptation [S1].
*   **When to Use:** Single panorama novel view synthesis, 3D virtual tours, VR games/conferences, filmmaking, text-to-3D, and 3D scene stylization [S1].
*   **When Not to Use:** When real-time training is required, or when high-quality perspective multi-view inputs are available (other methods may be more efficient).

## Background
*   **Origin:** Proposed in the paper "PERF: Panoramic Neural Radiance Field From a Single Panorama" [S1].
*   **History:** Developed to address the limitations of existing NeRF methods in handling single panorama inputs, specifically the lack of semantic coherence and geometric validity in occluded regions. It builds on Instant-NGP [40] and integrates Stable Diffusion 2 [61] for content completion.
*   **Usage:** Used for single panorama novel view synthesis and extended to applications like text-to-3D and 3D scene stylization [S1]. User studies showed ~81% preference for PERF results over DS-NeRF, SinNeRF, and DietNeRF [S1].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
