# Dynamic Volume Resolution & Alpha Mask Update

Also called: Dynamic Resolution and Mask Update

## Overview
This method dynamically adjusts the voxel grid resolution and prunes empty space in a neural volume representation (such as a Neural Radiance Field) during training to balance computational cost and geometric accuracy. It distinguishes itself from static resolution methods by coupling periodic resolution upscaling with alpha mask regeneration, ensuring that the active volume data structure shrinks to fit the occupied space [S1, unofficial]. Unlike sibling methods that rely on spatial metrics for adaptive sampling, this approach follows a pre-defined schedule of iterations to trigger structural changes [S1, unofficial].

## How it works
The method operates on a dense voxel grid or neural field representation. It uses two primary lists of iteration indices to trigger updates: `update_AlphaMask_list` for pruning empty space and `upsamp_list` for increasing resolution.

### Inputs and Outputs
*   **Inputs:**
    *   `iteration`: Current training step index.
    *   `reso_cur`: Current 3D resolution vector $(R_x, R_y, R_z)$.
    *   `tensorf`: The neural field object containing the volume grid and parameters.
    *   `args`: Configuration object containing hyperparameters (see Parameters).
    *   `N_voxel_list`: A queue of target voxel counts for upscaling.
    *   `allrays`, `allrgbs`: Training data (rays and target colors).
*   **Outputs:**
    *   Updated `tensorf` with modified grid resolution and parameters.
    *   Updated `optimizer` with new learning rates and parameter groups.
    *   Filtered `allrays`, `allrgbs` if bounding box culling is active.

### Data Structures
*   **Voxel Grid:** A 3D tensor of shape $(R_x, R_y, R_z, C)$, where $C$ is the channel dimension (e.g., density and color features).
*   **Alpha Mask:** A binary or float tensor indicating occupied voxels, derived from the density field.
*   **Optimizer State:** Adam optimizer state, including momentum buffers for gradients.

### Parameters
*   `update_AlphaMask_list`: List of integers. Iterations at which to regenerate the alpha mask.
*   `upsamp_list`: List of integers. Iterations at which to increase the volume resolution.
*   `args.lr_upsample_reset`: Boolean. If `True`, resets the learning rate to `args.lr_init` upon upscaling. If `False`, applies decay.
*   `args.lr_decay_target_ratio`: Float. The multiplier for learning rate decay when `lr_upsample_reset` is `False`.
*   `args.nSamples`: Integer. Maximum number of samples per ray.
*   `args.step_ratio`: Float. Ratio controlling sample density relative to resolution.
*   `args.lr_init`: Float. Initial learning rate for main parameters.
*   `args.lr_basis`: Float. Initial learning rate for basis parameters.
*   `args.L1_weight_rest`: Float. L1 regularization weight applied after the first mask update.
*   `args.ndc_ray`: Boolean. If `False`, rays are filtered against the bounding box.

### Steps
1.  **Alpha Mask Update Check:**
    *   If `iteration` is in `update_AlphaMask_list`:
        *   Determine mask resolution `reso_mask`. If `reso_cur[0] * reso_cur[1] * reso_cur[2] < 256^3`, set `reso_mask = reso_cur`. Otherwise, use a predefined lower resolution (implementation detail: [G] typically a fixed lower bound or scaled version).
        *   Call `tensorf.updateAlphaMask(tuple(reso_mask))` to generate a new occupancy mask.
        *   On the first update:
            *   Shrink the volume to the new Axis-Aligned Bounding Box (AABB): `tensorf.shrink(new_aabb)`.
            *   Update regularization: Set `L1_reg_weight = args.L1_weight_rest`.
        *   If `args.ndc_ray` is `False` and this is the second update iteration:
            *   Filter rays outside the new bounding box: `allrays, allrgbs = tensorf.filtering_rays(allrays, allrgbs)`.
            *   Reinitialize the sampler: `trainingSampler = SimpleSampler(allrgbs.shape[0], args.batch_size)`.

2.  **Resolution Upsampling Check:**
    *   If `iteration` is in `upsamp_list`:
        *   Pop the next target voxel count: `n_voxels = N_voxel_list.pop(0)`.
        *   Convert voxel count to 3D resolution: `reso_cur = N_to_reso(n_voxels, tensorf.aabb)`.
        *   Update samples per ray: `nSamples = min(args.nSamples, cal_n_samples(reso_cur, args.step_ratio))`.
        *   Upsample the volume grid: `tensorf.upsample_volume_grid(reso_cur)`.
        *   Calculate Learning Rate Scale (`lr_scale`):
            *   If `args.lr_upsample_reset` is `True`: `lr_scale = 1`.
            *   Else: `lr_scale = args.lr_decay_target_ratio ** (iteration / args.n_iters)`.
        *   Get new parameter groups with scaled learning rates:
            *   `grad_vars = tensorf.get_optparam_groups(args.lr_init * lr_scale, args.lr_basis * lr_scale)`.
        *   Reinitialize the Adam optimizer:
            *   `optimizer = torch.optim.Adam(grad_vars, betas=(0.9, 0.99))`.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_resolution_and_mask_update_step(
    iteration, 
    reso_cur, 
    tensorf, 
    args, 
    N_voxel_list, 
    update_AlphaMask_list, 
    upsamp_list, 
    allrays, 
    allrgbs, 
    trainingSampler, 
    optimizer
):
    """
    Performs dynamic resolution upscaling and alpha mask updates based on iteration schedule.
    """
    
    # --- Alpha Mask Update ---
    if iteration in update_AlphaMask_list:
        # Cap mask resolution if current volume is small
        if reso_cur[0] * reso_cur[1] * reso_cur[2] < 256**3:
            reso_mask = reso_cur
        else:
            # Implementation detail: [G] Use a fixed lower resolution or scaled resolution
            # For example, if reso_cur is (512,512,512), reso_mask might be (256,256,256)
            # The source implies a specific logic not fully detailed, but ensures mask is not too large.
            # We assume a standard downsampling or fixed cap here.
            reso_mask = tuple(max(1, r // 2) for r in reso_cur) 
        
        # Update the alpha mask in the tensorf
        new_aabb = tensorf.updateAlphaMask(tuple(reso_mask))
        
        # Handle first update specifics
        if iteration == update_AlphaMask_list[0]:
            tensorf.shrink(new_aabb)
            # Update L1 weight in the model or loss function
            args.L1_weight_rest = args.L1_weight_rest # Assuming this is set elsewhere or passed
            
        # Filter rays if not using NDC and it's the second update
        if not args.ndc_ray and iteration == update_AlphaMask_list[1]:
            allrays, allrgbs = tensorf.filtering_rays(allrays, allrgbs)
            trainingSampler = SimpleSampler(allrgbs.shape[0], args.batch_size)

    # --- Resolution Upsampling ---
    if iteration in upsamp_list:
        # Pop next target voxel count
        if not N_voxel_list:
            return # No more upscaling steps
            
        n_voxels = N_voxel_list.pop(0)
        
        # Convert voxel count to 3D resolution
        reso_cur = N_to_reso(n_voxels, tensorf.aabb)
        
        # Update samples per ray
        nSamples = min(args.nSamples, cal_n_samples(reso_cur, args.step_ratio))
        
        # Upsample the volume grid
        tensorf.upsample_volume_grid(reso_cur)
        
        # Calculate LR scale
        if args.lr_upsample_reset:
            lr_scale = 1.0
        else:
            lr_scale = args.lr_decay_target_ratio ** (iteration / args.n_iters)
            
        # Get new parameter groups with scaled LR
        grad_vars = tensorf.get_optparam_groups(
            args.lr_init * lr_scale, 
            args.lr_basis * lr_scale
        )
        
        # Reinitialize optimizer (discards momentum)
        optimizer = torch.optim.Adam(grad_vars, betas=(0.9, 0.99))

    return allrays, allrgbs, trainingSampler, optimizer, reso_cur
```

### Edge Cases and Pitfalls
*   **Ray Filtering:** Ray filtering against the bounding box is only applied if `args.ndc_ray` is `False`. If `True`, rays are in normalized device coordinates and bypass spatial culling, potentially wasting computation on empty space [S1, unofficial].
*   **Optimizer State Loss:** The optimizer is completely reinitialized on upscaling, discarding momentum states. This may cause training instability if the learning rate schedule is not adjusted accordingly [S1, unofficial].
*   **Mask Resolution Cap:** If the current volume resolution is very low ($< 256^3$ voxels), the mask resolution is capped to the current resolution to prevent excessive pruning [S1, unofficial].

## Subtopics

### [Alpha Mask Generation](3048-alpha-mask-generation.md)
Prunes empty space by converting density to a binary mask using 3D max pooling for robust boundary detection.

## Operating characteristics
*   **Time Complexity:** Upsampling involves interpolating the volume grid, which is $O(R_x R_y R_z)$. Alpha mask generation involves pooling and thresholding, also $O(R_x R_y R_z)$. Ray filtering is $O(N_{rays})$.
*   **Memory Complexity:** The voxel grid memory scales with $R_x R_y R_z$. Shrinking the AABB reduces memory usage by removing empty voxels.
*   **Performance:** Reduces computational load by focusing resources on occupied space and increasing resolution only when necessary.
*   **Strengths:** Efficient memory usage, adaptive detail.
*   **Limitations:** Discrete changes in resolution can cause visual popping if not smoothed. Reinitializing the optimizer may require careful tuning of learning rates.

## Background
This method is used in training Neural Radiance Fields (NeRF) and related neural volume representations to manage the trade-off between training time and reconstruction quality. It is part of the training pipeline for collaborative neural radiance fields.

## Sources
[S1] Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
