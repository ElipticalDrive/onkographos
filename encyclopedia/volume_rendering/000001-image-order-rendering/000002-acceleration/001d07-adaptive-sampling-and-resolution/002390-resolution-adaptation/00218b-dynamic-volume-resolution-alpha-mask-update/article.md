# Dynamic Volume Resolution & Alpha Mask Update

Also called: Dynamic Resolution and Mask Update

## Overview
This method dynamically adjusts the voxel grid resolution and the active spatial bounding box (AABB) of a neural volume representation during training. It solves the problem of inefficient memory usage and slow convergence by initially training in a low-resolution, coarse space and progressively refining the grid density and spatial extent as the model learns [S1, unofficial]. It differs from static resolution methods by coupling resolution upscaling with alpha mask updates, ensuring that the volume only occupies the space containing visible geometry, thereby pruning empty background space [S1, unofficial].

## How it works
The method operates as a periodic control loop triggered at specific training iterations. It manages two distinct resources: the spatial resolution of the voxel grid and the spatial extent of the volume (defined by its AABB).

### Inputs and Outputs
*   **Inputs:**
    *   `iteration`: Current training step index.
    *   `update_AlphaMask_list`: List of integers specifying iterations where the alpha mask and AABB should be updated.
    *   `upsamp_list`: List of integers specifying iterations where the voxel resolution should be increased.
    *   `args`: Configuration object containing hyperparameters (see Parameters).
    *   `tensorf`: The neural volume object holding the grid, optimizer, and AABB.
    *   `allrays`, `allrgbs`: The dataset of rays and corresponding ground truth colors.
*   **Outputs:**
    *   Updated `tensorf` with new resolution, new AABB, and reinitialized optimizer.
    *   Filtered `allrays` and `allrgbs` if rays fall outside the new bounding box.

### Data Structures
*   **AABB (Axis-Aligned Bounding Box):** Defined by two 3D vectors, `min` and `max`, representing the corners of the volume.
*   **Voxel Grid:** A 3D tensor of density and color features. Its dimensions are determined by `reso_cur`.
*   **Optimizer State:** Typically Adam or RAdam, storing momentum buffers for each parameter.

### Parameters
*   `update_AlphaMask_list`: List of iteration numbers.
*   `upsamp_list`: List of iteration numbers.
*   `args.lr_upsample_reset`: Boolean. If `True`, resets learning rate to `args.lr_init` on upsample. If `False`, applies decay.
*   `args.lr_decay_target_ratio`: Float. Multiplier for LR decay if `lr_upsample_reset` is `False`.
*   `args.nSamples`: Integer. Maximum samples per ray.
*   `args.step_ratio`: Float. Ratio controlling sample density relative to resolution.
*   `args.lr_init`: Float. Initial learning rate for main parameters.
*   `args.lr_basis`: Float. Initial learning rate for basis parameters.
*   `args.ndc_ray`: Boolean. If `True`, rays are in Normalized Device Coordinates (NDC), bypassing spatial culling.
*   `args.L1_weight_rest`: Float. L1 regularization weight used after the first shrink.

### Numbered Steps
1.  **Check for Alpha Mask Update:**
    *   If `iteration` is in `update_AlphaMask_list`:
        1.  Determine mask resolution `reso_mask`. If `reso_cur[0] * reso_cur[1] * reso_cur[2] < 256^3`, set `reso_mask = reso_cur`. Otherwise, cap `reso_mask` to a maximum resolution (implementation detail: typically the current resolution or a fixed cap, but the source implies capping if the volume is small) [S1, unofficial].
        2.  Update the alpha mask: `new_aabb = tensorf.updateAlphaMask(tuple(reso_mask))` [S1, unofficial]. This function computes a binary occupancy grid from the density field, likely using thresholding and multi-scale max pooling to smooth boundaries [S3, unofficial].
        3.  **Shrink Volume:** If this is the first update, shrink the volume to the new `new_aabb`: `tensorf.shrink(new_aabb)` [S1, unofficial].
        4.  **Update Regularization:** Switch the L1 regularization weight: `L1_reg_weight = args.L1_weight_rest` [S1, unofficial].
        5.  **Filter Rays (Conditional):** If `args.ndc_ray` is `False` AND this is the second update iteration:
            *   Filter rays outside the new bounding box: `allrays, allrgbs = tensorf.filtering_rays(allrays, allrgbs)` [S1, unofficial].
            *   Reinitialize the sampler: `trainingSampler = SimpleSampler(allrgbs.shape[0], args.batch_size)` [S1, unofficial].
            *   *Note:* NDC rays bypass spatial bounding box culling, so filtering is skipped for them [S1, unofficial].

2.  **Check for Resolution Upsampling:**
    *   If `iteration` is in `upsamp_list`:
        1.  Pop the next target voxel count: `n_voxels = N_voxel_list.pop(0)` [S1, unofficial].
        2.  Convert voxel count to resolution: `reso_cur = N_to_reso(n_voxels, tensorf.aabb)` [S1, unofficial]. This function maps the number of voxels to 3D grid dimensions (e.g., $N^{1/3} \times N^{1/3} \times N^{1/3}$) while preserving the aspect ratio of the AABB.
        3.  **Update Samples per Ray:** Calculate the new number of samples per ray: `nSamples = min(args.nSamples, cal_n_samples(reso_cur, args.step_ratio))` [S1, unofficial]. This ensures sample density scales with resolution but does not exceed the global maximum.
        4.  **Upsample Grid:** Upsample the volume grid to the new resolution: `tensorf.upsample_volume_grid(reso_cur)` [S1, unofficial]. This typically involves trilinear interpolation of the existing grid into the new, finer grid.
        5.  **Calculate Learning Rate Scale:**
            *   If `args.lr_upsample_reset` is `True`: `lr_scale = 1` [S1, unofficial].
            *   Else: `lr_scale = args.lr_decay_target_ratio ** (iteration / args.n_iters)` [S1, unofficial].
        6.  **Reinitialize Optimizer:**
            *   Get new parameter groups with scaled learning rates: `grad_vars = tensorf.get_optparam_groups(args.lr_init * lr_scale, args.lr_basis * lr_scale)` [S1, unofficial].
            *   Reinitialize the optimizer: `optimizer = torch.optim.Adam(grad_vars, betas=(0.9, 0.99))` [S1, unofficial].
            *   *Pitfall:* This discards all momentum states from previous steps, which may cause training instability if not intended [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def step_dynamic_resolution_and_mask(
    iteration, 
    update_AlphaMask_list, 
    upsamp_list, 
    args, 
    tensorf, 
    allrays, 
    allrgbs,
    N_voxel_list
):
    """
    Performs dynamic resolution and alpha mask updates for a neural volume.
    
    Args:
        iteration: Current training step.
        update_AlphaMask_list: List of iterations for alpha mask updates.
        upsamp_list: List of iterations for resolution upscaling.
        args: Configuration object.
        tensorf: Neural volume object with methods: updateAlphaMask, shrink, 
                 filtering_rays, upsample_volume_grid, get_optparam_groups, aabb.
        allrays: List of rays.
        allrgbs: List of ground truth colors.
        N_voxel_list: List of target voxel counts for upscaling.
        
    Returns:
        updated_allrays, updated_allrgbs, updated_tensorf, updated_optimizer
    """
    
    # --- Alpha Mask Update ---
    if iteration in update_AlphaMask_list:
        # Determine mask resolution
        current_voxel_count = tensorf.reso_cur[0] * tensorf.reso_cur[1] * tensorf.reso_cur[2]
        if current_voxel_count < 256**3:
            reso_mask = tensorf.reso_cur
        else:
            # Cap mask resolution to current resolution or a defined max
            # Source implies capping if volume is large, but exact cap logic is 
            # implementation-specific. We assume reso_mask is derived from reso_cur.
            reso_mask = tensorf.reso_cur 
            
        # Update alpha mask and get new AABB
        new_aabb = tensorf.updateAlphaMask(tuple(reso_mask))
        
        # Shrink volume to new AABB (first update)
        # Note: Source says "On first update", implying a flag or check.
        # We assume tensorf tracks if it has been shrunk.
        if not tensorf.has_been_shrunk:
            tensorf.shrink(new_aabb)
            tensorf.has_been_shrunk = True
            tensorf.L1_reg_weight = args.L1_weight_rest
            
        # Filter rays if not using NDC and it's the second update
        # Note: Source says "second update iteration". We need to track update count.
        if not args.ndc_ray and tensorf.alpha_update_count == 1:
            allrays, allrgbs = tensorf.filtering_rays(allrays, allrgbs)
            # Reinitialize sampler
            trainingSampler = SimpleSampler(len(allrgbs), args.batch_size)
            
        tensorf.alpha_update_count += 1

    # --- Resolution Upsampling ---
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
        grad_vars = tensorf.get_optparam_groups(
            args.lr_init * lr_scale, 
            args.lr_basis * lr_scale
        )
        optimizer = torch.optim.Adam(grad_vars, betas=(0.9, 0.99))
        
        # Update internal state
        tensorf.reso_cur = reso_cur
        tensorf.nSamples = nSamples
        
    return allrays, allrgbs, tensorf, optimizer

def N_to_reso(n_voxels, aabb):
    """
    Converts a total number of voxels to a 3D resolution tuple,
    preserving the aspect ratio of the AABB.
    """
    # Calculate side length assuming cubic voxels
    side_length = round(n_voxels ** (1/3))
    
    # Get AABB dimensions
    min_corner = aabb.min
    max_corner = aabb.max
    dims = max_corner - min_corner
    
    # Calculate resolution for each axis based on aspect ratio
    # This is a simplified implementation; actual implementation may vary
    # to ensure integer resolutions and exact voxel count.
    # A common approach is to scale the base side length by the aspect ratio.
    aspect_x = dims[0] / max(dims)
    aspect_y = dims[1] / max(dims)
    aspect_z = dims[2] / max(dims)
    
    reso_x = max(1, round(side_length * aspect_x))
    reso_y = max(1, round(side_length * aspect_y))
    reso_z = max(1, round(side_length * aspect_z))
    
    return (reso_x, reso_y, reso_z)

def cal_n_samples(reso, step_ratio):
    """
    Calculates the number of samples per ray based on resolution and step ratio.
    """
    # Assuming uniform sampling along the ray length
    # The number of samples is proportional to the grid resolution
    # and the step ratio.
    # This is a placeholder for the actual formula, which depends on 
    # the specific ray marching implementation.
    # A typical formula might be:
    #   ray_length = ...
    #   step_size = ray_length / reso
    #   n_samples = ray_length / (step_size * step_ratio)
    # For now, we return a simplified value.
    return int(reso[0] * step_ratio) # Simplified

class SimpleSampler:
    def __init__(self, num_rays, batch_size):
        self.num_rays = num_rays
        self.batch_size = batch_size
        
    def sample(self):
        # Placeholder for sampling logic
        pass
```

### Edge Cases and Pitfalls
*   **Optimizer Reinitialization:** Reinitializing the Adam optimizer discards momentum states. This can lead to training instability or slower convergence if the learning rate schedule is not adjusted accordingly [S1, unofficial].
*   **NDC Rays:** If `args.ndc_ray` is `True`, rays are not filtered by the AABB. This means the volume may be trained on empty space if the AABB is shrunk, leading to wasted computation [S1, unofficial].
*   **Ray Filtering Timing:** Ray filtering is only applied on the *second* alpha mask update. This may leave rays in empty space during the first update, potentially affecting the initial mask quality [S1, unofficial].
*   **Resolution Capping:** The mask resolution is capped if the volume is small (`< 256^3`). This prevents excessive detail in the mask for small volumes, but the exact cap logic is implementation-specific [S1, unofficial].

## Subtopics

### [Alpha Mask Generation](3048-alpha-mask-generation.md)
Alpha Mask Generation creates a binary occupancy grid from a dense density field to prune empty space and enable adaptive resolution. It differs from sibling methods by explicitly combining density thresholding with multi-scale max pooling to smooth the mask boundaries before updating the active bounding box.

## Operating characteristics
*   **Time Complexity:** The upscaling operation involves interpolating the entire volume grid, which is $O(N)$ where $N$ is the number of voxels. The alpha mask update involves computing the mask for the current resolution, also $O(N)$. Ray filtering is $O(R)$ where $R$ is the number of rays.
*   **Memory Complexity:** The memory usage scales with the voxel grid resolution. Upscaling increases memory usage, while shrinking the AABB reduces it by pruning empty space.
*   **Performance:** This method reduces training time and memory usage by focusing computation on the relevant spatial region. It allows for higher effective resolution in the final model by starting with a coarse grid [S1, unofficial].
*   **Strengths:** Efficient memory usage, ability to handle large scenes by pruning empty space, progressive refinement of geometry.
*   **Limitations:** Requires careful tuning of update schedules (`update_AlphaMask_list`, `upsamp_list`). Optimizer reinitialization can cause instability. NDC rays bypass spatial culling, reducing efficiency in some cases.

## Background
This method is used in neural volume rendering frameworks, such as those based on Neural Radiance Fields (NeRF) and their variants, to manage the trade-off between resolution and computational cost. It is particularly useful for large-scale scenes where a fixed high-resolution grid would be infeasible. The approach of dynamically adjusting resolution and bounding box is a common technique in adaptive volume rendering [S1, unofficial].

## Sources
[S1] Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
