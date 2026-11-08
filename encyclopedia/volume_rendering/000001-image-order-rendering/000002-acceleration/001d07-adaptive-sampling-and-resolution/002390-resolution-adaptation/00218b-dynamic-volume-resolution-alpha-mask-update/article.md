# Dynamic Volume Resolution & Alpha Mask Update

Also called: Dynamic Resolution and Mask Update

## Overview
This method dynamically adjusts the voxel grid resolution and the active spatial bounding box (AABB) of a volume representation during training to balance computational cost and geometric fidelity. It solves the problem of inefficiently training dense neural volumes by pruning empty space via an alpha mask and upscaling the grid only when necessary. It differs from static resolution methods by coupling spatial culling (alpha mask) with discrete resolution upscaling steps, and from continuous adaptive sampling methods by modifying the underlying domain discretization rather than just the ray sampling density [S1, unofficial].

## How it works
The method operates as a periodic update step within the training loop, triggered by specific iteration lists. It manages two coupled resources: the **Alpha Mask** (defining the active AABB) and the **Volume Grid Resolution** (defining the voxel density).

### Inputs and Outputs
*   **Inputs:** Current iteration number, current voxel resolution `reso_cur`, current AABB `tensorf.aabb`, list of training rays `allrays`, ground truth colors `allrgbs`, and hyperparameters (see Parameters).
*   **Outputs:** Updated voxel grid, updated AABB, updated ray sampler, and reinitialized optimizer state.

### Data Structures
*   **`reso_cur`**: A tuple `(res_x, res_y, res_z)` representing the current voxel grid dimensions.
*   **`tensorf.aabb`**: A 2x3 tensor or array defining the min/max coordinates of the current active bounding box.
*   **`update_AlphaMask_list`**: A sorted list of integer iteration numbers triggering mask updates.
*   **`upsamp_list`**: A sorted list of integer iteration numbers triggering resolution upscaling.
*   **`N_voxel_list`**: A list of target voxel counts (integers) corresponding to `upsamp_list`.

### Parameters
*   `update_AlphaMask_list`: List of iterations for mask updates [S1, unofficial].
*   `upsamp_list`: List of iterations for resolution upscaling [S1, unofficial].
*   `N_voxel_list`: List of target voxel counts for upscaling [S1, unofficial].
*   `args.lr_upsample_reset`: Boolean. If `True`, resets learning rate to initial on upsample; otherwise applies decay [S1, unofficial].
*   `args.lr_decay_target_ratio`: Float. Multiplier for LR decay if `lr_upsample_reset` is `False` [S1, unofficial].
*   `args.nSamples`: Maximum samples per ray before dynamic adjustment [S1, unofficial].
*   `args.step_ratio`: Float. Controls sample density relative to resolution [S1, unofficial].
*   `args.lr_init`, `args.lr_basis`: Initial learning rates for main and basis parameters [S1, unofficial].
*   `args.L1_weight_rest`: L1 regularization weight applied after initial shrinkage [S1, unofficial].
*   `args.ndc_ray`: Boolean. If `False`, enables ray filtering against the AABB [S1, unofficial].

### Algorithm Steps
The update logic is executed per training step. The following pseudocode combines the mask update and resolution upscaling logic. Note that these events are typically mutually exclusive or ordered within the same step depending on implementation, but conceptually they are distinct checks.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_dynamic_resolution_and_mask(
    iteration, 
    reso_cur, 
    tensorf, 
    allrays, 
    allrgbs, 
    args, 
    update_AlphaMask_list, 
    upsamp_list, 
    N_voxel_list
):
    """
    Performs dynamic resolution and alpha mask updates based on iteration triggers.
    tensorf: The neural volume field object containing updateAlphaMask, shrink, 
             upsample_volume_grid, filtering_rays, get_optparam_groups, and optimizer.
    """
    
    # --- Part 1: Alpha Mask Update ---
    if iteration in update_AlphaMask_list:
        # Cap mask resolution to prevent excessive memory usage for very fine grids
        if reso_cur[0] * reso_cur[1] * reso_cur[2] < 256**3:
            reso_mask = reso_cur
        else:
            # Not given in the sources: How to downsample reso_cur if > 256^3?
            # Assumption: Use a fixed lower bound or downsample by factor.
            # For implementation safety, we assume reso_mask is derived from reso_cur.
            # [G] Common practice is to cap mask resolution to a manageable size, 
            # e.g., 128 or 256. Here we assume a fallback if not explicitly defined.
            reso_mask = (128, 128, 128) # [G] Placeholder for missing logic

        # Update the alpha mask and get the new active bounding box
        new_aabb = tensorf.updateAlphaMask(tuple(reso_mask))
        
        # Shrink the volume to the new AABB
        tensorf.shrink(new_aabb)
        
        # Switch L1 weight to the rest value
        args.L1_weight = args.L1_weight_rest
        
        # If not using NDC rays, filter rays outside the new bounding box
        if not args.ndc_ray:
            # Check if this is the second update iteration to apply filtering
            # [G] The source implies a check for "second update iteration". 
            # We assume a counter or flag tracks this.
            if is_second_mask_update(): 
                allrays, allrgbs = tensorf.filtering_rays(allrays, allrgbs)
                # Reinitialize sampler with new ray count
                trainingSampler = SimpleSampler(allrgbs.shape[0], args.batch_size)
    
    # --- Part 2: Resolution Upscaling ---
    if iteration in upsamp_list:
        # Pop the next target voxel count
        if N_voxel_list:
            n_voxels = N_voxel_list.pop(0)
        else:
            return # No more upscaling steps

        # Convert voxel count to resolution based on current AABB
        reso_cur = N_to_reso(n_voxels, tensorf.aabb)
        
        # Update samples per ray based on new resolution and step ratio
        # cal_n_samples is a helper function defined elsewhere
        nSamples = min(args.nSamples, cal_n_samples(reso_cur, args.step_ratio))
        
        # Upsample the volume grid to the new resolution
        tensorf.upsample_volume_grid(reso_cur)
        
        # Calculate Learning Rate Scale
        if args.lr_upsample_reset:
            lr_scale = 1.0
        else:
            # Apply decay based on iteration progress
            lr_scale = args.lr_decay_target_ratio ** (iteration / args.n_iters)
        
        # Get new parameter groups with scaled learning rates
        grad_vars = tensorf.get_optparam_groups(
            args.lr_init * lr_scale, 
            args.lr_basis * lr_scale
        )
        
        # Reinitialize Adam optimizer (discards momentum states)
        # [G] Standard Adam initialization with betas=(0.9, 0.99) as per source
        optimizer = torch.optim.Adam(grad_vars, betas=(0.9, 0.99))
```

### Helper Functions (Implied/Required)
*   `N_to_reso(n_voxels, aabb)`: Calculates `(res_x, res_y, res_z)` such that `res_x * res_y * res_z ≈ n_voxels` while preserving the aspect ratio of `aabb`.
*   `cal_n_samples(reso, step_ratio)`: Computes the number of samples per ray, likely proportional to the grid resolution and scaled by `step_ratio`.
*   `is_second_mask_update()`: Returns `True` if the current mask update is the second one performed in the training session.

### Edge Cases and Pitfalls
1.  **Ray Filtering Condition:** Ray filtering (`filtering_rays`) is only applied if `args.ndc_ray` is `False`. NDC rays bypass spatial bounding box culling, which may cause wasted computation on empty space if the mask shrinks but rays are not filtered [S1, unofficial].
2.  **Optimizer State Loss:** The optimizer is completely reinitialized on upsample, discarding momentum states. This may cause training instability if not intended or if the learning rate scale is not adjusted carefully [S1, unofficial].
3.  **Mask Resolution Cap:** If the current voxel count exceeds $256^3$, the mask resolution logic is not explicitly defined in the sources. Implementers must define a fallback (e.g., fixed resolution or downsampled resolution) to avoid errors in `updateAlphaMask` [S1, unofficial].

## Subtopics

### [Alpha Mask Generation](3048-alpha-mask-generation.md)
Alpha Mask Generation creates a binary occupancy grid from a dense density field to prune empty space and enable adaptive resolution. It differs from sibling methods by explicitly combining density thresholding with multi-scale max pooling to smooth the mask boundaries before updating the active bounding box.

## Operating characteristics
*   **Time Complexity:** The mask update is $O(V_{mask})$ where $V_{mask}$ is the mask resolution. The upscaling is $O(V_{new})$ where $V_{new}$ is the new voxel count. Ray filtering is $O(N_{rays})$.
*   **Memory Complexity:** Memory scales with the voxel grid size. The mask adds $O(V_{mask})$ memory. Upscaling increases memory usage significantly, requiring sufficient VRAM.
*   **Strengths:** Reduces training time by ignoring empty space; allows high-resolution details to be learned only where necessary.
*   **Limitations:** Discrete upscaling steps can cause visual popping if not smoothed; reinitializing the optimizer can destabilize training; NDC ray modes are incompatible with spatial culling.
*   **When to use:** When training neural volumes with large empty backgrounds or sparse geometry.
*   **When not to use:** When using NDC rays without custom filtering; when VRAM is extremely limited and upscaling steps exceed capacity.

## Background
This method is derived from implementations in Collaborative Neural Radiance Fields for Novel View Synthesis, specifically the `CoNGP` repository [S1, unofficial]. It represents a practical engineering approach to dynamic resolution in neural volume rendering, combining alpha masking (common in 3D Gaussian Splatting and NeRF variants) with discrete grid upscaling.

## Sources
[S1] Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
