# Progressive View Synthesis Training

Also called: PERF training loop, collaborative RGBD inpainting and progressive inpainting-and-erasing

## Overview
Progressive View Synthesis Training (PERF) is a training strategy for Panoramic Neural Radiance Fields (NeRF) that synthesizes novel views from a single input panorama by iteratively expanding the set of reference views [S1]. It solves the problem of synthesizing occluded regions in panoramic scenes, where standard NeRF methods suffer from "foggy effects" or "floating" densities due to lack of semantic and geometric constraints [S1]. Unlike sibling methods that rely on semantic regularization via CLIP or DINO features [S1], PERF distinguishes itself by using collaborative RGBD inpainting and a progressive inpainting-and-erasing strategy to generate high-quality pseudo-ground truth for occluded areas, ensuring semantic coherence and valid surface convergence [S1].

## How it works
The method operates in an iterative loop that alternates between sampling new camera poses, generating inpainted reference views for those poses, and retraining the NeRF representation with the expanded dataset.

### Inputs and Outputs
*   **Input:** A single input RGB-D panorama (color image and depth map).
*   **Output:** A trained NeRF model capable of rendering novel views (RGB and depth) from arbitrary camera poses within the scene.

### Data Structures
The implementation uses the Instant-NGP representation via the NeRFAcc framework [S1].
*   **Hash Grid:** A hash table of size $2^{19}$ with 16 levels of feature vectors, each having a channel size of 2 [S1].
*   **Ray Buffer:** At each iteration, 2048 rays are sampled. Each ray is associated with a ground-truth or inpainted color value and a depth value [S1].

### Parameters
*   **Initial Training Iterations:** 10,000 [S1].
*   **Number of New Views:** 32 positions are progressively sampled [S1].
*   **Sampling Distance:** New positions are sampled with gradually increased distance from the original reference panorama view position [S1].
*   **Training Iterations per New View:** 2,000 [S1].
*   **Supervision Split:**
    *   First 100 iterations: Supervision uses only the new reference view [S1].
    *   Remaining 1,900 iterations: Supervision uses all accumulated reference views [S1].
*   **Optimizer:** Adam [S1].
*   **Learning Rate Schedule:**
    *   Initial learning rate: $10^{-3}$ [S1].
    *   Decayed learning rate: $10^{-4}$ [S1].
    *   Strategy: Cosine annealing for each learning stage of a newly sampled view [S1].
*   **Rays per Iteration:** 2,048 [S1].
*   **Distortion Loss Weight ($\lambda_{dist}$):** 1 [S1].
*   **Inpainting Model:** Pre-trained Stable Diffusion 2 [S1].

### Steps
1.  **Initial Training:** Train the NeRF model using the single input panorama for 10,000 iterations [S1].
2.  **View Sampling:** Progressively sample 32 new camera positions. The distance of these positions from the original reference view increases gradually [S1].
3.  **Collaborative RGBD Inpainting:** For each newly added reference view position:
    *   Render the current NeRF to identify occluded or unreliable regions.
    *   Project the masked regions of the panorama to perspective views to avoid distortion artifacts inherent in applying perspective-trained inpainting models directly to panoramas [S1].
    *   Apply the pre-trained Stable Diffusion 2 model to inpaint the color and depth information in these perspective views [S1].
    *   Project the inpainted regions back to the panorama format [S1].
4.  **Progressive Inpainting-and-Erasing:** Apply the inpainting-and-erasing strategy to resolve geometry conflicts that arise from different views, preventing wrong redundant geometry and degraded RGB quality [S1].
5.  **Incremental Training:** Continue NeRF training for 2,000 iterations with the new view:
    *   **Phase 1 (Iterations 1–100):** Use only the new reference view (and its inpainted data) for supervision [S1].
    *   **Phase 2 (Iterations 101–2000):** Supervise the NeRF using all accumulated reference views (original + newly added) [S1].
6.  **Ray Sampling and Loss:** At each iteration, randomly sample 2,048 rays. Compute the loss using the ground-truth or inpainted colors and depth values. Include the distortion loss with weight $\lambda_{dist} = 1$ to encourage volume densities to converge to a valid surface rather than "floating" everywhere [S1].
7.  **Repeat:** Repeat steps 2–6 for all 32 new views.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_view_synthesis_training(input_panorama_rgb, input_panorama_depth):
    """
    Implements the PERF training loop for a single panorama.
    """
    # Initialize NeRF with Instant-NGP representation
    # Hash table size: 2^19, 16 levels, channel size 2
    nerf = InstantNGPNeRF()
    
    # Initialize optimizer (Adam)
    optimizer = Adam(nerf.parameters(), lr=1e-3)
    
    # Step 1: Initial Training
    # Train for 10,000 iterations using only the single input panorama
    for i in range(10000):
        rays = sample_rays(2048, input_panorama_rgb, input_panorama_depth)
        colors, depths = nerf.render(rays)
        loss = compute_loss(colors, depths, rays.gt_color, rays.gt_depth, lambda_dist=1.0)
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
    # Step 2: Progressive View Sampling
    # Sample 32 new positions with gradually increased distance from original view
    new_view_positions = sample_positions(count=32, base_position=input_panorama.camera_pose, distance_schedule='gradual_increase')
    
    for view_pos in new_view_positions:
        # Step 3: Collaborative RGBD Inpainting
        # Render current NeRF to find occluded/masked regions
        rendered_rgb, rendered_depth, mask = nerf.render_view(view_pos)
        
        # Project masked regions to perspective views to handle distortion
        perspective_patches = project_to_perspective(mask, rendered_rgb, rendered_depth)
        
        # Inpaint using Stable Diffusion 2
        inpainted_patches = stable_diffusion_2_inpaint(perspective_patches)
        
        # Project back to panorama
        inpainted_rgb, inpainted_depth = project_back_to_perspective(inpainted_patches, mask)
        
        # Step 4: Progressive Inpainting-and-Erasing
        # Apply strategy to avoid geometry conflicts
        apply_inpainting_and_erasing_strategy(inpainted_rgb, inpainted_depth)
        
        # Create new reference view data
        new_view_data = {
            'camera_pose': view_pos,
            'rgb': inpainted_rgb,
            'depth': inpainted_depth
        }
        
        # Add to accumulated reference views
        reference_views.append(new_view_data)
        
        # Step 5: Incremental Training
        # Train for 2,000 iterations per new view
        # Learning rate schedule: Cosine annealing from 1e-3 to 1e-4
        scheduler = CosineAnnealingLR(optimizer, T_max=2000, eta_min=1e-4)
        
        for j in range(2000):
            # Sample rays from all accumulated reference views
            # Use ground-truth or inpainted colors and depths
            rays = sample_rays_from_views(2048, reference_views)
            
            colors, depths = nerf.render(rays)
            
            # Compute loss with distortion regularization
            loss = compute_loss(colors, depths, rays.gt_color, rays.gt_depth, lambda_dist=1.0)
            
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
            scheduler.step()
            
    return nerf

def compute_loss(rendered_colors, rendered_depths, gt_colors, gt_depths, lambda_dist):
    """
    Computes the training loss including color, depth, and distortion terms.
    """
    # Color loss (e.g., L1 or L2)
    color_loss = mean_squared_error(rendered_colors, gt_colors)
    
    # Depth loss (e.g., L1)
    depth_loss = mean_absolute_error(rendered_depths, gt_depths)
    
    # Distortion loss to prevent floating densities
    # Weight lambda_dist = 1
    distortion_loss = compute_distortion_loss(rendered_depths, lambda_dist)
    
    total_loss = color_loss + depth_loss + distortion_loss
    return total_loss
```

### Edge Cases and Pitfalls
*   **Panorama Distortion:** Inpainting models trained on perspective FOV cannot be directly applied to panoramas due to distortion. The method must project masked regions to perspective views, inpaint, and project back [S1].
*   **Foggy Effects/Floating Densities:** NeRF can produce invalid surfaces ("foggy effects") if not regularized. This is mitigated by using distortion loss with weight $\lambda_{dist} = 1$ [S1].
*   **Geometry Conflicts:** Without the inpainting-and-erasing strategy, geometry conflicts arise from different views, leading to wrong redundant geometry and degraded RGB quality [S1].

## Operating characteristics
*   **Time Complexity:** Training a scene takes approximately 2 hours on a single Nvidia V100 GPU with 32G memory [S1].
*   **Memory Complexity:** The hash table size is $2^{19}$ with 16 levels of feature vectors of channel size 2 [S1].
*   **Performance:** On the Replica dataset, PERF achieves a PSNR of 23.49, SSIM of 0.838, and LPIPS of 0.244, outperforming DS-NeRF, DietNeRF, SinNeRF, 360FusionNeRF, and Omni-NeRF [S1].
*   **Strengths:** Superior performance in synthesizing occluded regions with semantic coherence; avoids foggy artifacts and floating densities; better visual quality compared to semantic regularization methods [S1].
*   **Limitations:** Relies on a pre-trained inpainting model (Stable Diffusion 2), which may introduce artifacts if the inpainting is inaccurate; requires careful handling of panorama distortion [S1].
*   **When to Use:** Single panorama novel view synthesis, 3D virtual tours, VR games/conferences, filmmaking, text-to-3D, and 3D scene stylization [S1].
*   **When Not to Use:** When high-quality multi-view input is available (other methods may be more efficient); when computational resources are extremely limited (2 hours on V100 is non-trivial); when semantic coherence is less critical than geometric accuracy from direct observations.

## Background
*   **Origin:** Proposed in the paper "PERF: Panoramic Neural Radiance Field From a Single Panorama" [S1].
*   **History:** Developed to address the limitations of existing NeRF methods in handling single-panorama inputs, particularly in occluded regions where standard NeRF fails to infer content semantically [S1].
*   **Usage:** Used in applications such as 3D virtual tours, VR games/conferences, filmmaking, text-to-3D, and 3D scene stylization [S1].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
