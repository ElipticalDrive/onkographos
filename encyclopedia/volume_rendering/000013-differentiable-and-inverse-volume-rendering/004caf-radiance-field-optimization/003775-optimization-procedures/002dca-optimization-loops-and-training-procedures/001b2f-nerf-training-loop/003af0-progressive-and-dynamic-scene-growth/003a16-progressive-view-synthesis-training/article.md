# Progressive View Synthesis Training

Also called: PERF training loop, collaborative RGBD inpainting and progressive inpainting-and-erasing

## Overview
Progressive View Synthesis Training (PERF) is a training strategy for Panoramic Neural Radiance Fields (NeRF) that improves geometry and semantic coherence in occluded regions by progressively adding reference views to a single input panorama [S1]. It sets itself apart from sibling methods like Dynamic Growth Parameters by focusing on view synthesis expansion rather than spatial density field expansion, and from Online Incremental NeRF Optimization by using a pre-defined progressive sampling schedule rather than active agent acquisition [8666-progressive-view-synthesis-training.md]. Unlike alternatives such as DS-NeRF or DietNeRF, PERF explicitly addresses the "foggy" artifacts and semantic incoherence common in single-panorama synthesis by integrating collaborative RGBD inpainting and an inpainting-and-erasing strategy [S1].

## How it works
**Inputs:** A single input RGB-D panorama (color and depth maps).
**Outputs:** A trained NeRF model capable of synthesizing novel views from arbitrary panoramic poses.

**Data Structures:**
*   **NeRF Model:** Implemented via Instant-NGP [S1].
    *   **Hash Table:** Size $2^{19}$ with 16 levels of feature vectors, each of channel size 2 [S1].
*   **Reference Views:** A list of camera poses. Initially contains the single input panorama. Grows by 32 positions [S1].
*   **Inpainting Model:** Pre-trained Stable Diffusion 2 [S1].

**Parameters:**
*   Initial training iterations: 10,000 [S1].
*   Number of progressively sampled new positions: 32 [S1].
*   Training iterations per new view: 2,000 [S1].
*   Supervision iterations using only the new view: 100 [S1].
*   Supervision iterations using all views: 1,900 [S1].
*   Initial learning rate: $10^{-3}$ [S1].
*   Decayed learning rate: $10^{-4}$ [S1].
*   Number of rays sampled per iteration: 2,048 [S1].
*   Distortion loss weight ($\lambda_{dist}$): 1 [S1].
*   Optimizer: Adam [S1].
*   Learning rate schedule: Cosine annealing from $10^{-3}$ to $10^{-4}$ for each learning stage of a newly sampled view [S1].

**Steps:**

1.  **Initial Training:** Train the NeRF model using only the single input panorama for 10,000 iterations [S1].
2.  **Progressive View Sampling:** Sample 32 new camera positions with gradually increased distance from the original reference panorama view position [S1].
3.  **Iterative View Addition:** For each newly added reference view, perform the following sub-steps:
    a.  **Collaborative RGBD Inpainting:**
        i.  Identify masked regions (occluded areas) in the new view relative to the current NeRF prediction or ground truth if available.
        ii. Project these masked regions from the panoramic coordinate system to perspective views [S1].
        iii. Apply the pre-trained Stable Diffusion 2 inpainting model to the perspective views [S1].
        iv. Project the inpainted regions back to the panoramic coordinate system [S1].
    b.  **Progressive Inpainting-and-Erasing:** This strategy avoids geometry conflicts arising from different views, which lead to wrong redundant geometry and degraded RGB quality [S1]. (Specific algorithmic details of the erasing mechanism are not provided in the sources, but it is applied after inpainting) [S1].
    c.  **NeRF Re-optimization:** Continue NeRF training for 2,000 iterations [S1].
        i.  **Phase 1 (First 100 iterations):** Use only the new reference view for supervision [S1].
        ii. **Phase 2 (Remaining 1,900 iterations):** Supervise NeRF with all reference views (the original panorama plus all previously added views) [S1].
        iii. At each iteration, randomly sample 2,048 rays with ground-truth or inpainted colors and depth values [S1].
        iv. Apply the distortion loss with weight $\lambda_{dist} = 1$ to encourage volume densities to converge to a valid surface instead of "floating" everywhere [S1].
        v.  Reset the learning rate to $10^{-3}$ and decay to $10^{-4}$ using cosine annealing for this new stage [S1].
4.  **Completion:** After all 32 views are added and optimized, the model is ready for novel view synthesis [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
from torch import optim
from nerf_acc import InstantNGP # Conceptual wrapper for Instant-NGP implementation
from stable_diffusion import InpaintModel # Conceptual wrapper for Stable Diffusion 2

def project_to_perspective(panoramic_mask, camera_pose):
    """
    Projects masked regions from panoramic coordinates to perspective view coordinates.
    Implementation detail: Uses camera projection matrices to map 3D points or 2D masks.
    """
    # Not given in the sources. Placeholder for geometric projection logic.
    pass

def project_back_to_panorama(perspective_inpainting, camera_pose):
    """
    Projects inpainted regions from perspective view coordinates back to panoramic coordinates.
    """
    # Not given in the sources. Placeholder for inverse geometric projection logic.
    pass

def train_perf(single_input_panorama):
    """
    Trains a NeRF model using the Progressive View Synthesis Training (PERF) method.
    
    Args:
        single_input_panorama: Tuple of (RGB_image, Depth_image) for the initial view.
        
    Returns:
        trained_nerf_model: The optimized NeRF model.
    """
    # 1. Initialize NeRF with Instant-NGP
    # Hash table size: 2^19, 16 levels, channel size 2
    nerf_model = InstantNGP(hash_size=2**19, num_levels=16, channel_size=2)
    
    # 2. Initial Training
    # Train with single input panorama for 10,000 iterations
    optimizer = optim.Adam(nerf_model.parameters(), lr=1e-3)
    for i in range(10000):
        rays = sample_rays(2048, single_input_panorama)
        loss = compute_loss(nerf_model, rays, single_input_panorama)
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
    # 3. Progressive View Sampling
    # Sample 32 new positions with gradually increased distance from original position
    new_view_positions = sample_progressive_positions(32, initial_pose=single_input_panorama.pose)
    
    reference_views = [single_input_panorama]
    
    for new_view_pos in new_view_positions:
        # Create a placeholder for the new view's ground truth if it exists, 
        # or prepare for inpainting if it's a novel view synthesis target.
        # Note: The source implies "reference views" are added. In single-panorama 
        # synthesis, these are often synthesized or inpainted targets.
        
        # 3a. Collaborative RGBD Inpainting
        # Identify masked regions (occluded areas)
        masked_regions = identify_occlusions(nerf_model, new_view_pos)
        
        # Project to perspective
        perspective_masks = project_to_perspective(masked_regions, new_view_pos)
        
        # Inpaint using Stable Diffusion 2
        perspective_inpaintings = []
        for mask in perspective_masks:
            inpainted = InpaintModel.inpaint(mask, new_view_pos)
            perspective_inpaintings.append(inpainted)
            
        # Project back to panorama
        inpainted_panorama = project_back_to_panorama(perspective_inpaintings, new_view_pos)
        
        # 3b. Progressive Inpainting-and-Erasing
        # This step avoids geometry conflicts. Specific algorithm not detailed in source.
        # [G] Assuming this modifies the reference view data or NeRF state to 
        # resolve conflicts before optimization.
        inpainted_panorama = apply_inpainting_and_erasing(inpainted_panorama, nerf_model, new_view_pos)
        
        # Add to reference views
        reference_views.append(inpainted_panorama)
        
        # 3c. NeRF Re-optimization
        # Reset optimizer for new stage
        optimizer = optim.Adam(nerf_model.parameters(), lr=1e-3)
        
        # Phase 1: 100 iterations with only the new view
        for i in range(100):
            rays = sample_rays(2048, inpainted_panorama)
            loss = compute_loss(nerf_model, rays, inpainted_panorama, lambda_dist=1.0)
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
        # Phase 2: 1,900 iterations with all reference views
        for i in range(1900):
            # Randomly sample rays from all reference views
            rays = sample_rays_from_all_views(2048, reference_views)
            loss = compute_loss(nerf_model, rays, reference_views, lambda_dist=1.0)
            
            # Cosine annealing decay from 1e-3 to 1e-4
            current_lr = cosine_annealing(step=i, total_steps=1900, max_lr=1e-3, min_lr=1e-4)
            for param_group in optimizer.param_groups:
                param_group['lr'] = current_lr
                
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
    return nerf_model

def sample_rays(num_rays, view):
    """
    Randomly samples rays from a view.
    """
    # Not given in the sources. Placeholder for ray sampling logic.
    pass

def sample_rays_from_all_views(num_rays, views):
    """
    Randomly samples rays from a list of views.
    """
    # Not given in the sources. Placeholder for ray sampling logic.
    pass

def compute_loss(nerf_model, rays, views, lambda_dist=1.0):
    """
    Computes the training loss including color, depth, and distortion loss.
    """
    # Render colors and depths from NeRF
    rendered_colors, rendered_depths = nerf_model.render(rays)
    
    # Get ground truth or inpainted colors and depths
    gt_colors, gt_depths = views.get_ground_truth(rays)
    
    # Color and Depth Loss
    color_loss = torch.mean((rendered_colors - gt_colors) ** 2)
    depth_loss = torch.mean((rendered_depths - gt_depths) ** 2)
    
    # Distortion Loss
    # Encourages volume densities to converge to a valid surface
    distortion_loss = compute_distortion_loss(nerf_model, rays)
    
    total_loss = color_loss + depth_loss + lambda_dist * distortion_loss
    return total_loss

def compute_distortion_loss(nerf_model, rays):
    """
    Computes the distortion loss as defined in [4] (referenced in S1).
    """
    # Not given in the sources. Placeholder for distortion loss calculation.
    pass

def identify_occlusions(nerf_model, camera_pose):
    """
    Identifies masked regions in the new view that are occluded or unobserved.
    """
    # Not given in the sources. Placeholder for occlusion detection.
    pass

def sample_progressive_positions(num_positions, initial_pose):
    """
    Samples 32 new positions with gradually increased distance from the initial pose.
    """
    # Not given in the sources. Placeholder for position sampling logic.
    pass

def cosine_annealing(step, total_steps, max_lr, min_lr):
    """
    Cosine annealing learning rate schedule.
    """
    import math
    return min_lr + 0.5 * (max_lr - min_lr) * (1 + math.cos(math.pi * step / total_steps))
```

## Subtopics

### Dynamic Growth Parameters
Adds new branches or parameters where the model exhibits high uncertainty or reconstruction error. Choose when spatial detail needs to be added adaptively based on error metrics rather than view coverage. [2968-dynamic-growth-parameters.md]

### Online Incremental NeRF Optimization
Adds new views as they are captured by an active agent. Choose when the scene is dynamic or when the camera path is determined by an online exploration strategy. [12066-online-incremental-nerf-optimization.md]

## Operating characteristics
*   **Time Complexity:** Training a scene takes approximately 2 hours on a single Nvidia V100 GPU with 32G memory [S1]. The complexity is dominated by the iterative re-optimization steps for each of the 32 added views, with 2,000 iterations per view.
*   **Memory Complexity:** The Instant-NGP hash table size is $2^{19}$ with 16 levels of feature vectors of channel size 2 [S1]. The memory usage scales with the hash table size and the number of reference views stored for supervision.
*   **Measured Performance:** On the Replica dataset, PERF achieves a PSNR of 23.49, SSIM of 0.838, and LPIPS of 0.244, outperforming DS-NeRF, DietNeRF, SinNeRF, 360FusionNeRF, and Omni-NeRF [S1]. It achieves a masked PSNR of 12.63, demonstrating effective inpainting of occluded regions [S1].
*   **Strengths:**
    *   Effectively synthesizes occluded regions with semantic coherence using Stable Diffusion inpainting [S1].
    *   Avoids "foggy" artifacts and "floating" densities through distortion loss and inpainting-and-erasing [S1].
    *   User study shows ~81% preference for PERF results over other methods [S1].
*   **Limitations:**
    *   Relies on a pre-defined progressive sampling schedule, which may not be optimal for all scene geometries.
    *   Requires a pre-trained inpainting model (Stable Diffusion 2), which adds computational overhead and dependency [S1].
    *   Inpainting models trained on perspective FOV cannot be directly applied to panoramas due to distortion, requiring projection steps [S1].
*   **When to Use:** Single panorama novel view synthesis, 3D virtual tours, VR games/conferences, filmmaking, text-to-3D, and 3D scene stylization [S1].
*   **When Not to Use:** When real-time online updates are required (use Online Incremental NeRF Optimization) or when spatial detail needs to be added based on local error metrics (use Dynamic Growth Parameters).

## Background
Progressive View Synthesis Training (PERF) was introduced in the paper "PERF: Panoramic Neural Radiance Field From a Single Panorama" [S1]. It addresses the limitations of existing NeRF methods in handling single-panorama inputs, where occluded regions often result in foggy artifacts or semantic incoherence. The method builds on the principles of iterative expansion shared by the Progressive and Dynamic Scene Growth family, specifically focusing on view synthesis needs to resolve occlusions and expand the field of view [8666-progressive-view-synthesis-training.md]. It was implemented using the NeRFAcc framework and Instant-NGP representation [S1].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
