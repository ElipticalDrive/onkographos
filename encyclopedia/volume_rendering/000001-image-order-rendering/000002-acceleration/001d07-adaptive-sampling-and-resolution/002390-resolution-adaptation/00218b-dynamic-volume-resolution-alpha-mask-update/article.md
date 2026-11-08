# Dynamic Volume Resolution & Alpha Mask Update

Also called: Dynamic Resolution and Mask Update

## Overview
This method dynamically adjusts the voxel grid resolution and the active spatial bounding box of a neural volume representation during training to balance memory usage and rendering fidelity. It solves the problem of inefficiently allocating resources to empty space or overly dense regions by periodically shrinking the volume to its occupied core and upscaling the grid density at predefined intervals. It differs from [Dynamic Resolution Adaptation](2683-dynamic-resolution-adaptation.md) by explicitly managing the *training* state (optimizer, learning rate, bounding box) rather than just the inference-time sampling rate, and from [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md) by modifying the underlying neural field's parameter space rather than selecting precomputed textures.

## How it works
The method operates as a periodic update loop triggered by specific training iterations. It performs two coupled operations: updating the alpha mask (spatial extent) and upscaling the volume resolution (grid density).

**Inputs:**
*   `iteration`: Current training step index.
*   `tensorf`: The neural field object containing the voxel grid, bounding box (`aabb`), and optimizer.
*   `args`: Configuration object containing lists and hyperparameters.
*   `N_voxel_list`: A list of target voxel counts for upscaling, popped sequentially.

**Outputs:**
*   Updated `tensorf` with new resolution, bounding box, and optimizer state.
*   Updated ray sampling parameters (`nSamples`).

**Data Structures:**
*   `reso_cur`: A 3D tuple `(res_x, res_y, res_z)` representing the current voxel grid resolution.
*   `new_aabb`: A 2D tensor or array of shape `(2, 3)` representing the min and max coordinates of the new active bounding box.
*   `optimizer`: An Adam optimizer instance holding momentum states for the neural field parameters.

**Parameters:**
*   `update_AlphaMask_list`: List of integers. Iterations at which to update the alpha mask and bounding box [S1, unofficial].
*   `upsamp_list`: List of integers. Iterations at which to increase the voxel grid resolution [S1, unofficial].
*   `args.lr_upsample_reset`: Boolean. If `True`, resets learning rate to `args.lr_init` on upsample; if `False`, applies decay [S1, unofficial].
*   `args.lr_decay_target_ratio`: Float. The multiplier for learning rate decay when `lr_upsample_reset` is `False` [S1, unofficial].
*   `args.nSamples`: Integer. Maximum number of samples per ray before dynamic adjustment [S1, unofficial].
*   `args.step_ratio`: Float. Scaling factor for sample count relative to resolution [S1, unofficial].
*   `args.lr_init`, `args.lr_basis`: Floats. Initial learning rates for the main field and basis parameters [S1, unofficial].
*   `args.ndc_ray`: Boolean. If `True`, rays are normalized to [-1, 1] and bypass spatial bounding box culling [S1, unofficial].

**Steps:**

1.  **Alpha Mask Update Check:**
    If `iteration` is in `update_AlphaMask_list`:
    a.  Determine mask resolution `reso_mask`. If `reso_cur[0] * reso_cur[1] * reso_cur[2] < 256**3`, set `reso_mask = reso_cur`. Otherwise, `reso_mask` is typically a downsampled version (implementation detail: often `reso_cur // 4` or similar, but exact downsampling factor is not specified in sources; assume `reso_mask` is derived from `reso_cur` such that it fits memory) [S1, unofficial].
    b.  Compute the new active bounding box: `new_aabb = tensorf.updateAlphaMask(tuple(reso_mask))` [S1, unofficial]. This function typically involves thresholding the density field to create a binary occupancy grid and finding the min/max coordinates of occupied voxels.
    c.  Shrink the volume: `tensorf.shrink(new_aabb)` [S1, unofficial]. This reindexes the voxel grid to the new AABB, discarding empty space.
    d.  Update regularization: Set `L1_reg_weight = args.L1_weight_rest` [S1, unofficial].
    e.  Filter rays (Conditional): If `not args.ndc_ray` and this is the *second* alpha mask update iteration:
        i.  Filter rays outside the new bounding box: `allrays, allrgbs = tensorf.filtering_rays(allrays, allrgbs)` [S1, unofficial].
        ii. Reinitialize the sampler: `trainingSampler = SimpleSampler(allrgbs.shape[0], args.batch_size)` [S1, unofficial].

2.  **Resolution Upscaling Check:**
    If `iteration` is in `upsamp_list`:
    a.  Pop the next target voxel count: `n_voxels = N_voxel_list.pop(0)` [S1, unofficial].
    b.  Convert voxel count to resolution: `reso_cur = N_to_reso(n_voxels, tensorf.aabb)` [S1, unofficial]. This function calculates the new `(res_x, res_y, res_z)` such that `res_x * res_y * res_z ≈ n_voxels` while preserving the aspect ratio of `tensorf.aabb`.
    c.  Calculate new samples per ray: `nSamples = min(args.nSamples, cal_n_samples(reso_cur, args.step_ratio))` [S1, unofficial]. `cal_n_samples` typically scales with the total number of voxels or the maximum dimension.
    d.  Upsample the grid: `tensorf.upsample_volume_grid(reso_cur)` [S1, unofficial]. This interpolates the existing neural field parameters to the new, higher-resolution grid.
    e.  Calculate Learning Rate Scale:
        i.  If `args.lr_upsample_reset` is `True`: `lr_scale = 1` [S1, unofficial].
        ii. Else: `lr_scale = args.lr_decay_target_ratio ** (iteration / args.n_iters)` [S1, unofficial].
    f.  Update Optimizer:
        i.  Get new parameter groups with scaled learning rates: `grad_vars = tensorf.get_optparam_groups(args.lr_init * lr_scale, args.lr_basis * lr_scale)` [S1, unofficial].
        ii. Reinitialize Adam optimizer: `optimizer = torch.optim.Adam(grad_vars, betas=(0.9, 0.99))` [S1, unofficial]. Note: This discards previous momentum states [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_resolution_and_mask_update_step(
    iteration, 
    tensorf, 
    args, 
    N_voxel_list, 
    update_AlphaMask_list, 
    upsamp_list, 
    allrays=None, 
    allrgbs=None, 
    trainingSampler=None
):
    """
    Performs periodic updates to the neural volume's resolution and bounding box.
    
    Args:
        iteration: Current training step.
        tensorf: Neural field object with methods: updateAlphaMask, shrink, filtering_rays, 
                 upsample_volume_grid, get_optparam_groups, aabb.
        args: Configuration object.
        N_voxel_list: List of target voxel counts for upscaling.
        update_AlphaMask_list: List of iterations for mask updates.
        upsamp_list: List of iterations for resolution upscaling.
        allrays: Tensor of ray origins and directions.
        allrgbs: Tensor of ground truth colors.
        trainingSampler: Current ray sampler.
        
    Returns:
        Updated tensorf, allrays, allrgbs, trainingSampler, nSamples.
    """
    nSamples = args.nSamples
    
    # 1. Alpha Mask Update
    if iteration in update_AlphaMask_list:
        # Determine mask resolution
        current_voxel_count = tensorf.reso_cur[0] * tensorf.reso_cur[1] * tensorf.reso_cur[2]
        if current_voxel_count < 256**3:
            reso_mask = tensorf.reso_cur
        else:
            # Implementation detail: Downsample reso_cur. 
            # Source does not specify exact downsampling factor, assume a standard reduction.
            # [G] Common practice is dividing by 4 or 8, but we must not guess.
            # We assume a helper function exists or it is passed externally.
            # For completeness, we note that reso_mask is derived from reso_cur.
            reso_mask = tuple(r // 4 for r in tensorf.reso_cur) # Placeholder for unspecified logic [G]
            
        # Update alpha mask and get new AABB
        new_aabb = tensorf.updateAlphaMask(tuple(reso_mask))
        
        # Shrink volume to new AABB
        tensorf.shrink(new_aabb)
        
        # Update L1 weight
        tensorf.L1_reg_weight = args.L1_weight_rest
        
        # Filter rays if not using NDC and it's the second update
        if not args.ndc_ray and tensorf.alpha_mask_update_count == 1: # Assuming counter tracks updates
            allrays, allrgbs = tensorf.filtering_rays(allrays, allrgbs)
            trainingSampler = SimpleSampler(allrgbs.shape[0], args.batch_size)
            tensorf.alpha_mask_update_count += 1
        elif not args.ndc_ray:
            tensorf.alpha_mask_update_count += 1

    # 2. Resolution Upscaling
    if iteration in upsamp_list:
        if not N_voxel_list:
            raise ValueError("No more voxel counts in N_voxel_list for upscaling")
            
        n_voxels = N_voxel_list.pop(0)
        
        # Convert voxel count to resolution
        reso_cur = N_to_reso(n_voxels, tensorf.aabb)
        tensorf.reso_cur = reso_cur
        
        # Calculate samples per ray
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
        # Reinitialize Adam, discarding momentum
        tensorf.optimizer = torch.optim.Adam(grad_vars, betas=(0.9, 0.99))
        
    return nSamples, trainingSampler, allrays, allrgbs

def N_to_reso(n_voxels, aabb):
    """
    Converts a target voxel count to a 3D resolution tuple based on AABB aspect ratio.
    """
    # [G] Standard implementation: calculate aspect ratios, distribute voxels proportionally.
    # Since source does not provide this function, we describe it.
    # 1. Get dimensions of aabb: (x_max-x_min, y_max-y_min, z_max-z_min)
    # 2. Calculate aspect ratios.
    # 3. Solve for res_x, res_y, res_z such that res_x*res_y*res_z = n_voxels 
    #    and res_x/res_y = aspect_x/aspect_y, etc.
    # 4. Round to integers.
    # This function is assumed to be implemented in tensorf or utils.
    pass

def cal_n_samples(reso, step_ratio):
    """
    Calculates number of samples based on resolution and step ratio.
    """
    # [G] Typically: max_samples = max(reso) * step_ratio or similar.
    # Source does not specify formula.
    pass

class SimpleSampler:
    def __init__(self, num_rays, batch_size):
        self.num_rays = num_rays
        self.batch_size = batch_size
        self.indices = torch.randperm(num_rays)
        self.current_idx = 0
        
    def __iter__(self):
        return self
    
    def __next__(self):
        if self.current_idx >= self.num_rays:
            raise StopIteration
        batch_indices = self.indices[self.current_idx:self.current_idx + self.batch_size]
        self.current_idx += self.batch_size
        return batch_indices
```

## Subtopics

### [Alpha Mask Generation](3048-alpha-mask-generation.md)
Alpha Mask Generation creates a binary occupancy grid from a dense density field to prune empty space and enable adaptive resolution. It differs from sibling methods by explicitly combining density thresholding with multi-scale max pooling to smooth the mask boundaries before updating the active bounding box.

## Operating characteristics
*   **Time Complexity:** The alpha mask update is $O(V)$ where $V$ is the number of voxels in the mask resolution. The upscaling is $O(V_{new})$ for interpolation and optimizer reinitialization. Ray filtering is $O(R)$ where $R$ is the number of rays.
*   **Memory Complexity:** Reduces memory usage by shrinking the bounding box to occupied space. Upscaling increases memory usage to $O(V_{new})$.
*   **Performance:** Improves training efficiency by focusing computation on relevant geometry. However, frequent updates can cause overhead.
*   **Strengths:** Effective for scenes with large empty spaces. Prevents overfitting to noise in empty regions.
*   **Limitations:** Requires careful tuning of `update_AlphaMask_list` and `upsamp_list`. Reinitializing the optimizer can cause training instability if not intended [S1, unofficial]. NDC rays bypass spatial culling, potentially wasting computation if not handled [S1, unofficial].
*   **When to use:** When training neural radiance fields on scenes with significant empty space or when memory is constrained.
*   **When not to use:** When using NDC rays without ray filtering, or when training stability is critical and optimizer reinitialization is undesirable.

## Background
This method originates from the training pipeline of Collaborative Neural Radiance Fields for Novel View Synthesis [S1, unofficial]. It is designed to handle the dynamic nature of neural volume representations, where the optimal resolution and spatial extent change as the model learns. The approach combines techniques from adaptive sampling and spatial pruning to optimize the trade-off between rendering quality and computational cost.

## Sources
[S1] Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
