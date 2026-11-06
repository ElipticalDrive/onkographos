# Dynamic Volume Resolution & Alpha Mask Update

Also called: Dynamic Resolution and Mask Update

## Overview
This method dynamically adjusts the spatial resolution of a volumetric representation and regenerates an alpha mask to prune empty space during training. It solves the problem of inefficient computation in sparse or evolving scenes by reducing the voxel grid size when the occupied volume is small and increasing it as detail is learned. It sets itself apart from static resolution methods by coupling grid upscaling with alpha mask culling and optimizer reinitialization, ensuring that computational resources are focused only on the active region of the scene [S1, unofficial].

## How it works
The method operates as a periodic update step within a training loop, triggered by specific iteration lists. It manages two coupled processes: alpha mask generation (pruning) and volume resolution upscaling (refinement).

### Inputs and Outputs
*   **Inputs**: Current training iteration, current voxel resolution `reso_cur`, current axis-aligned bounding box (AABB) of the volume, list of trigger iterations `update_AlphaMask_list` and `upsamp_list`, and hyperparameters (`args`).
*   **Outputs**: Updated voxel grid resolution, updated alpha mask, potentially resized bounding box, and reinitialized optimizer state.

### Data Structures
*   `reso_cur`: A tuple or array of three integers $(R_x, R_y, R_z)$ representing the current voxel grid dimensions.
*   `aabb`: A tensor or array defining the 3D bounding box of the active volume.
*   `update_AlphaMask_list`: A sorted list of integers representing training iterations where the alpha mask should be regenerated.
*   `upsamp_list`: A sorted list of integers representing training iterations where the volume resolution should be increased.
*   `N_voxel_list`: A list of target total voxel counts corresponding to `upsamp_list`.

### Parameters
*   `args.update_AlphaMask_list`: List of iterations for mask updates.
*   `args.upsamp_list`: List of iterations for resolution upscaling.
*   `args.lr_upsample_reset`: Boolean. If `True`, resets learning rate to initial value on upsample; if `False`, applies decay.
*   `args.lr_decay_target_ratio`: Float. Multiplier for LR decay if `lr_upsample_reset` is `False`.
*   `args.nSamples`: Maximum samples per ray before dynamic adjustment.
*   `args.step_ratio`: Float. Controls sample density relative to resolution.
*   `args.lr_init`: Initial learning rate for main parameters.
*   `args.lr_basis`: Initial learning rate for basis parameters.
*   `args.L1_weight_rest`: L1 regularization weight applied after shrinkage.
*   `args.ndc_ray`: Boolean. If `False`, enables ray filtering outside the bounding box.

### Algorithm Steps

1.  **Check for Alpha Mask Update**:
    If the current iteration is in `update_AlphaMask_list`:
    a.  **Cap Mask Resolution**: If the current voxel count $R_x R_y R_z < 256^3$, set `reso_mask = reso_cur`. Otherwise, `reso_mask` is typically derived from the current resolution or a predefined lower bound (implementation detail: often `reso_mask` is set to `reso_cur` to ensure consistency).
    b.  **Generate Mask**: Call `new_aabb = tensorf.updateAlphaMask(tuple(reso_mask))`. This function computes the alpha mask from the density field, identifies occupied voxels, and returns the new, tighter AABB enclosing the non-empty space.
    c.  **Shrink Volume**: If this is the first update, shrink the neural field's internal representation to `new_aabb` using `tensorf.shrink(new_aabb)`.
    d.  **Update Regularization**: Set the L1 regularization weight to `args.L1_weight_rest`.
    e.  **Filter Rays (Conditional)**: If `args.ndc_ray` is `False` and this is the second update iteration:
        i.  Filter rays that lie outside the new `new_aabb`: `allrays, allrgbs = tensorf.filtering_rays(allrays, allrgbs)`.
        ii. Reinitialize the ray sampler: `trainingSampler = SimpleSampler(allrgbs.shape[0], args.batch_size)`.

2.  **Check for Resolution Upscaling**:
    If the current iteration is in `upsamp_list`:
    a.  **Get Target Voxel Count**: Pop the next target voxel count from the list: `n_voxels = N_voxel_list.pop(0)`.
    b.  **Calculate New Resolution**: Convert `n_voxels` to a 3D resolution tuple `reso_cur` while preserving the aspect ratio of the `aabb`. The function `N_to_reso(n_voxels, tensorf.aabb)` performs this mapping.
    c.  **Adjust Samples per Ray**: Calculate the new number of samples per ray:
        $$ nSamples = \min(\text{args.nSamples}, \text{cal\_n\_samples}(\text{reso\_cur}, \text{args.step\_ratio})) $$
        where `cal_n_samples` typically scales the sample count based on the new resolution and step ratio to maintain visual quality.
    d.  **Upsample Grid**: Call `tensorf.upsample_volume_grid(reso_cur)`. This increases the resolution of the underlying voxel grid, usually by interpolation or zero-padding, to match `reso_cur`.
    e.  **Calculate Learning Rate Scale**:
        $$ \text{lr\_scale} = \begin{cases} 1 & \text{if args.lr\_upsample\_reset is True} \\ \text{args.lr\_decay\_target\_ratio}^{(\text{iteration} / \text{args.n\_iters})} & \text{otherwise} \end{cases} $$
    f.  **Reinitialize Optimizer**:
        i.  Get new parameter groups with scaled learning rates:
            $$ \text{grad\_vars} = \text{tensorf.get\_optparam\_groups}(\text{args.lr\_init} \times \text{lr\_scale}, \text{args.lr\_basis} \times \text{lr\_scale}) $$
        ii. Create a new Adam optimizer:
            $$ \text{optimizer} = \text{torch.optim.Adam}(\text{grad\_vars}, \text{betas}=(0.9, 0.99)) $$
        Note: This discards previous momentum states [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_resolution_and_mask_update_step(
    iteration, 
    reso_cur, 
    aabb, 
    update_AlphaMask_list, 
    upsamp_list, 
    N_voxel_list, 
    args, 
    tensorf, 
    allrays, 
    allrgbs, 
    trainingSampler
):
    """
    Performs dynamic resolution upscaling and alpha mask updates.
    
    Args:
        iteration: Current training iteration number.
        reso_cur: Current voxel resolution (tuple of 3 ints).
        aabb: Current axis-aligned bounding box.
        update_AlphaMask_list: List of iterations for mask updates.
        upsamp_list: List of iterations for resolution upscaling.
        N_voxel_list: List of target total voxel counts.
        args: Configuration object containing hyperparameters.
        tensorf: The neural field object with methods updateAlphaMask, shrink, 
                 filtering_rays, upsample_volume_grid, get_optparam_groups.
        allrays: Tensor of ray origins and directions.
        allrgbs: Tensor of target RGB values.
        trainingSampler: Current ray sampler object.
        
    Returns:
        Updated reso_cur, allrays, allrgbs, trainingSampler, optimizer.
    """
    
    # --- Alpha Mask Update ---
    if iteration in update_AlphaMask_list:
        # Cap mask resolution if current volume is small
        if reso_cur[0] * reso_cur[1] * reso_cur[2] < 256**3:
            reso_mask = reso_cur
        else:
            # Implementation detail: reso_mask is set to current resolution
            # to ensure the mask covers the active area adequately.
            reso_mask = reso_cur 
            
        # Generate new alpha mask and get tightened AABB
        new_aabb = tensorf.updateAlphaMask(tuple(reso_mask))
        
        # Shrink the volume to the new bounding box
        tensorf.shrink(new_aabb)
        
        # Update L1 regularization weight
        args.L1_weight_rest = args.L1_weight_rest # Assuming this is set elsewhere or passed
        
        # Filter rays if not using NDC and it's the second update
        if not args.ndc_ray and iteration == update_AlphaMask_list[1]: # Assuming second update index
            allrays, allrgbs = tensorf.filtering_rays(allrays, allrgbs)
            # Reinitialize sampler with new ray count
            trainingSampler = SimpleSampler(allrgbs.shape[0], args.batch_size)
            
    # --- Resolution Upscaling ---
    if iteration in upsamp_list:
        # Pop the next target voxel count
        if N_voxel_list:
            n_voxels = N_voxel_list.pop(0)
        else:
            n_voxels = None
            
        if n_voxels is not None:
            # Convert voxel count to resolution
            reso_cur = N_to_reso(n_voxels, aabb)
            
            # Calculate samples per ray
            nSamples = min(args.nSamples, cal_n_samples(reso_cur, args.step_ratio))
            
            # Upsample the volume grid
            tensorf.upsample_volume_grid(reso_cur)
            
            # Calculate learning rate scale
            if args.lr_upsample_reset:
                lr_scale = 1.0
            else:
                lr_scale = (args.lr_decay_target_ratio ** (iteration / args.n_iters))
                
            # Get new parameter groups with scaled LR
            grad_vars = tensorf.get_optparam_groups(
                args.lr_init * lr_scale, 
                args.lr_basis * lr_scale
            )
            
            # Reinitialize Adam optimizer
            optimizer = torch.optim.Adam(grad_vars, betas=(0.9, 0.99))
            
    return reso_cur, allrays, allrgbs, trainingSampler, optimizer

def N_to_reso(n_voxels, aabb):
    """
    Converts a total number of voxels to a 3D resolution tuple
    preserving the aspect ratio of the AABB.
    """
    # Calculate aspect ratios
    # Assuming aabb is [min_x, min_y, min_z, max_x, max_y, max_z]
    dx = aabb[3] - aabb[0]
    dy = aabb[4] - aabb[1]
    dz = aabb[5] - aabb[2]
    
    # Total volume in world units
    total_vol = dx * dy * dz
    
    # Voxel volume
    voxel_vol = total_vol / n_voxels
    
    # Side lengths in world units
    side_x = (voxel_vol * (dx/total_vol) * n_voxels) ** (1/3) # Simplified logic
    # More robust: scale base resolution
    # Let base resolution be 1,1,1. Scale factors are dx, dy, dz.
    # We want n_x * n_y * n_z = n_voxels
    # n_x / n_y = dx / dy, etc.
    
    # Approximate resolution
    base_res = int(round(n_voxels ** (1/3)))
    
    # Scale by aspect ratio
    rx = int(round(base_res * (dx / (total_vol ** (1/3)))))
    ry = int(round(base_res * (dy / (total_vol ** (1/3)))))
    rz = int(round(base_res * (dz / (total_vol ** (1/3)))))
    
    # Ensure minimum resolution
    rx = max(1, rx)
    ry = max(1, ry)
    rz = max(1, rz)
    
    return (rx, ry, rz)

def cal_n_samples(reso, step_ratio):
    """
    Calculates the number of samples per ray based on resolution and step ratio.
    """
    # Typical implementation: samples = resolution * step_ratio
    # Or based on max dimension
    max_res = max(reso)
    return int(max_res * step_ratio)

class SimpleSampler:
    def __init__(self, num_rays, batch_size):
        self.num_rays = num_rays
        self.batch_size = batch_size
        
    def __call__(self):
        # Placeholder for sampling logic
        pass
```

### Edge Cases and Pitfalls
*   **Optimizer Instability**: Reinitializing the Adam optimizer discards momentum states. This can cause training instability, especially if upscaling happens too frequently or too early [S1, unofficial].
*   **NDC Rays**: Ray filtering (`filtering_rays`) is only applied if `args.ndc_ray` is `False`. NDC rays bypass spatial bounding box culling, so filtering them is unnecessary or incorrect [S1, unofficial].
*   **Resolution Capping**: If the current voxel count is less than $256^3$, the mask resolution is capped to the current resolution to prevent excessive pruning or artifacts [S1, unofficial].

## Subtopics

### [Alpha Mask Generation](3048-alpha-mask-generation.md)
Generates an alpha mask from the dense density field to prune empty space, using a threshold and max pooling for smoothing.

## Operating characteristics
*   **Time Complexity**: The alpha mask update involves a forward pass through the network to compute densities, which is $O(V)$ where $V$ is the number of voxels. The resolution upscaling involves grid interpolation, also $O(V_{new})$. The ray filtering is $O(R)$ where $R$ is the number of rays.
*   **Memory Complexity**: The memory footprint scales with the voxel grid size $R_x R_y R_z$. Upscaling increases memory usage, while alpha mask pruning may allow for a smaller effective bounding box, potentially reducing memory if the grid is sparse.
*   **Performance**: This method is most effective in scenes with significant empty space or evolving structures. It reduces computation in empty regions and increases detail in occupied regions. However, the overhead of mask generation and grid upscaling must be balanced against the savings in ray marching.
*   **Strengths**: Efficiently handles sparse scenes; adapts to changing scene complexity; reduces memory usage in early training stages.
*   **Limitations**: Requires careful tuning of trigger iterations; optimizer reinitialization can destabilize training; NDC ray support is limited.

## Background
This method is associated with Collaborative Neural Radiance Fields for Novel View Synthesis [S1, unofficial]. It was developed to address the inefficiencies of static voxel grids in neural rendering, particularly in collaborative or distributed settings where scene content may vary. The approach combines dynamic resolution adjustment with alpha mask culling to optimize resource allocation during training.

## Sources
[S1] Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
