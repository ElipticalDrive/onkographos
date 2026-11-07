# Dynamic Volume Resolution & Alpha Mask Update

Also called: Dynamic Resolution and Mask Update

## Overview
This method dynamically adjusts the spatial resolution of a voxel-based volume representation and regenerates an alpha mask to prune empty space during training. It solves the problem of inefficient memory usage and slow convergence in sparse or complex scenes by coarsening the grid where possible and refining it when necessary. It differs from [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md) by modifying the underlying grid resolution globally rather than selecting pre-computed levels, and from [Dynamic Resolution Adaptation](2683-dynamic-resolution-adaptation.md) by coupling resolution changes with explicit alpha mask regeneration and optimizer reinitialization.

## How it works
The method operates as a scheduled update loop triggered at specific iterations. It manages two primary resources: the voxel grid resolution ($reso$) and the alpha mask (a binary or soft mask indicating occupied space).

### Inputs and Outputs
*   **Inputs:**
    *   `iteration`: Current training step.
    *   `tensorf`: The neural field object containing the volume grid, bounding box (`aabb`), and parameters.
    *   `args`: Configuration object containing hyperparameters (see Parameters).
    *   `update_AlphaMask_list`: List of iterations for alpha mask updates.
    *   `upsamp_list`: List of iterations for resolution upscaling.
    *   `N_voxel_list`: List of target voxel counts for upscaling.
*   **Outputs:**
    *   Updated `tensorf` with new resolution and/or alpha mask.
    *   Updated optimizer state (potentially reinitialized).
    *   Updated sampling parameters (`nSamples`).

### Data Structures
*   **Voxel Grid:** A 3D tensor of shape $(reso[0], reso[1], reso[2])$.
*   **Alpha Mask:** A binary or soft tensor of shape $(reso\_mask[0], reso\_mask[1], reso\_mask[2])$ indicating occupied voxels.
*   **Bounding Box (AABB):** Axis-aligned bounding box defining the spatial extent of the volume.

### Parameters
*   `update_AlphaMask_list`: List of integers. Iterations at which to update the alpha mask.
*   `upsamp_list`: List of integers. Iterations at which to increase volume resolution.
*   `args.lr_upsample_reset`: Boolean. If `True`, resets learning rate to initial value on upsample; if `False`, applies decay.
*   `args.lr_decay_target_ratio`: Float. Multiplier for LR decay during upscaling if `lr_upsample_reset` is `False`.
*   `args.nSamples`: Integer. Maximum samples per ray before dynamic adjustment.
*   `args.step_ratio`: Float. Controls sample density relative to resolution during upscaling.
*   `args.lr_init`, `args.lr_basis`: Floats. Initial learning rates for main and basis parameters.
*   `args.ndc_ray`: Boolean. If `False`, enables ray filtering outside the bounding box.

### Steps
1.  **Alpha Mask Update Check:**
    *   If `iteration` is in `update_AlphaMask_list`:
        *   Determine mask resolution: If `reso_cur[0] * reso_cur[1] * reso_cur[2] < 256**3`, set `reso_mask = reso_cur`. Otherwise, cap `reso_mask` to a maximum resolution (implementation detail: typically `256**3` or similar limit to prevent memory overflow).
        *   Update alpha mask: Call `tensorf.updateAlphaMask(tuple(reso_mask))`.
        *   Shrink volume: On the first update, call `tensorf.shrink(new_aabb)` to adjust the bounding box to the new occupied region.
        *   Switch L1 weight: Set `L1_reg_weight = args.L1_weight_rest`.
        *   Filter rays (if applicable): If `args.ndc_ray` is `False` and this is the second update iteration, call `tensorf.filtering_rays(allrays, allrgbs)` to remove rays outside the new bounding box. Reinitialize sampler: `trainingSampler = SimpleSampler(allrgbs.shape[0], args.batch_size)`.

2.  **Resolution Upsampling Check:**
    *   If `iteration` is in `upsamp_list`:
        *   Pop next voxel count: `n_voxels = N_voxel_list.pop(0)`.
        *   Convert to resolution: `reso_cur = N_to_reso(n_voxels, tensorf.aabb)`.
        *   Update samples per ray: `nSamples = min(args.nSamples, cal_n_samples(reso_cur, args.step_ratio))`.
        *   Upsample volume: Call `tensorf.upsample_volume_grid(reso_cur)`.
        *   Calculate LR scale:
            *   If `args.lr_upsample_reset` is `True`, `lr_scale = 1`.
            *   Else, `lr_scale = args.lr_decay_target_ratio ** (iteration / args.n_iters)`.
        *   Reinitialize optimizer:
            *   Get new parameter groups: `grad_vars = tensorf.get_optparam_groups(args.lr_init * lr_scale, args.lr_basis * lr_scale)`.
            *   Create new optimizer: `optimizer = torch.optim.Adam(grad_vars, betas=(0.9, 0.99))`.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def step_dynamic_resolution(tensorf, args, iteration, update_AlphaMask_list, upsamp_list, N_voxel_list, allrays, allrgbs):
    """
    Performs dynamic resolution and alpha mask updates based on scheduled iterations.
    """
    # 1. Alpha Mask Update
    if iteration in update_AlphaMask_list:
        # Determine mask resolution, capping if total voxels < 256^3
        if tensorf.reso[0] * tensorf.reso[1] * tensorf.reso[2] < 256**3:
            reso_mask = tensorf.reso
        else:
            # Implementation detail: cap mask resolution to prevent memory issues
            # Assuming a max resolution limit, e.g., 256x256x256
            reso_mask = (256, 256, 256) 
        
        # Update alpha mask
        new_aabb = tensorf.updateAlphaMask(tuple(reso_mask))
        
        # Shrink volume to new AABB on first update
        if tensorf.is_first_alpha_update:
            tensorf.shrink(new_aabb)
            tensorf.is_first_alpha_update = False
            # Switch L1 weight
            args.L1_reg_weight = args.L1_weight_rest
        
        # Filter rays if not using NDC and it's the second update
        if not args.ndc_ray and tensorf.alpha_update_count == 2:
            allrays, allrgbs = tensorf.filtering_rays(allrays, allrgbs)
            # Reinitialize sampler
            trainingSampler = SimpleSampler(allrgbs.shape[0], args.batch_size)
    
    # 2. Resolution Upsampling
    if iteration in upsamp_list:
        # Pop next voxel count
        if not N_voxel_list:
            raise ValueError("No more voxel counts in N_voxel_list")
        n_voxels = N_voxel_list.pop(0)
        
        # Convert voxel count to resolution
        reso_cur = N_to_reso(n_voxels, tensorf.aabb)
        
        # Update samples per ray
        nSamples = min(args.nSamples, cal_n_samples(reso_cur, args.step_ratio))
        
        # Upsample volume grid
        tensorf.upsample_volume_grid(reso_cur)
        
        # Calculate LR scale
        if args.lr_upsample_reset:
            lr_scale = 1.0
        else:
            lr_scale = args.lr_decay_target_ratio ** (iteration / args.n_iters)
        
        # Reinitialize optimizer
        grad_vars = tensorf.get_optparam_groups(args.lr_init * lr_scale, args.lr_basis * lr_scale)
        optimizer = torch.optim.Adam(grad_vars, betas=(0.9, 0.99))
        
        return allrays, allrgbs, trainingSampler, optimizer
```

### Edge Cases and Pitfalls
*   **Ray Filtering:** Ray filtering (`filtering_rays`) is only applied if `args.ndc_ray` is `False`. NDC rays bypass spatial bounding box culling, which may cause wasted computation on empty space if not handled [S1, unofficial].
*   **Optimizer Reinitialization:** The optimizer is completely reinitialized on upsample, discarding momentum states. This may cause training instability if not intended [S1, unofficial].
*   **Memory Limits:** Alpha mask resolution is capped to prevent memory overflow when the volume resolution is high.

## Subtopics

### [Alpha Mask Generation](3048-alpha-mask-generation.md)
Generates an alpha mask from the dense density field to prune empty space, using a threshold and max pooling for smoothing.

## Operating characteristics
*   **Time Complexity:** The upsample operation is $O(N)$ where $N$ is the number of new voxels. The alpha mask update is $O(M)$ where $M$ is the number of voxels in the mask. Ray filtering is $O(R)$ where $R$ is the number of rays.
*   **Memory Complexity:** Memory usage scales with the voxel grid resolution. Upsampling increases memory usage, while alpha mask pruning can reduce it by allowing the volume to shrink.
*   **Strengths:** Efficiently manages memory in sparse scenes. Allows for high-resolution details in complex regions while keeping empty space coarse.
*   **Limitations:** Requires careful scheduling of updates to avoid visual popping or training instability. Reinitializing the optimizer can disrupt convergence.
*   **When to use:** When training neural radiance fields or voxel-based representations on scenes with significant empty space or varying complexity.
*   **When not to use:** When real-time inference is required and dynamic resolution changes are not supported by the renderer. When the scene is uniformly dense and resolution changes offer little benefit.

## Background
This method is derived from implementations in Collaborative Neural Radiance Fields for Novel View Synthesis [S1, unofficial]. It addresses the challenges of training large-scale neural radiance fields by dynamically adjusting the representation resolution and pruning empty space.

## Sources
[S1] Collaborative Neural Radiance Fields for Novel View Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
