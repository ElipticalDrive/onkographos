# Variance-based Step Size Adaptation

Also called: Choosing dt through Variance Bricking, Variance Bricking

## Overview
Variance-based Step Size Adaptation is a preprocessing technique that determines the adaptive ray traversal step size ($\Delta t$) for volume rendering by calculating the statistical variance of voxel data within spatial bricks [S1, unofficial]. It solves the problem of selecting an appropriate step size for sparse or local traversal without the computational cost of calculating iso-surface normals at runtime [S1, unofficial]. Unlike fixed-step methods, it adjusts $\Delta t$ based on local data complexity: high variance (frequent normal changes) yields smaller steps, while low variance allows larger steps [S1, unofficial].

## How it works
**Inputs:**
*   Volume dataset $V$ stored in bricks.
*   Voxel dimensions $dx, dy, dz$.
*   Precomputed variance map $v_{ijk}$ for each brick $(i,j,k)$.
*   Constants $\kappa_{min}, \kappa_{max}$.

**Outputs:**
*   Adaptive step size $\Delta t_{ijk}$ for each brick.

**Data Structures:**
*   **Brick:** A sub-volume of the dataset. The source uses block dimensions such as $4 \times 4 \times 4$, $8 \times 8 \times 8$, etc. [S1, unofficial].
*   **Variance Map:** A 3D array where each element corresponds to a brick, storing the precomputed variance $v$.

**Steps:**

1.  **Preprocessing (CPU):** For each brick in the volume, calculate the variance $v$ of the voxel values contained within it.
    $$ v = \frac{1}{n} \sum_{i=1}^{n} (x_i - \bar{x})^2 $$
    where $n$ is the number of voxels in the brick, $x_i$ is the value of the $i$-th voxel, and $\bar{x}$ is the mean of the voxel values in the brick [S1, unofficial].

2.  **Mapping Variance to Coefficient:** Map the variance $v$ of a brick to a coefficient $\kappa$ using a linear function. This function takes the brick's variance $v$, the global minimum variance $v_{min}$, and the global maximum variance $v_{max}$ across all bricks [S1, unofficial]:
    $$ \kappa = \kappa_{min} + \frac{(\kappa_{max} - \kappa_{min})(v - v_{min})}{v_{max} - v_{min}} $$
    Note: The source formula is written as $\kappa_{min} \frac{v - v_{min}}{v_{max} - v_{min}} (\kappa_{max} - \kappa_{min})$? No, the source text says:
    $$ f(v, v_{max}, v_{min}) = \kappa_{min} \frac{v - v_{min}}{v_{max} - v_{min}} (\kappa_{max} - \kappa_{min}) $$
    Wait, let's look at the source text carefully:
    `f (v, vmax , vmin ) = κmin (v − vmin ) / (vmax − vmin) * (κmax − κmin)`?
    The source text is:
    `f (v, vmax , vmin ) = κmin (v − vmin ) κmax − κmin vmax − vmin`
    This is ambiguous. However, standard linear interpolation between $\kappa_{min}$ (at $v_{min}$) and $\kappa_{max}$ (at $v_{max}$) is:
    $$ \kappa = \kappa_{min} + (\kappa_{max} - \kappa_{min}) \frac{v - v_{min}}{v_{max} - v_{min}} $$
    The source text snippet `κmin (v − vmin ) κmax − κmin vmax − vmin` likely implies:
    $$ \kappa = \kappa_{min} + \frac{(\kappa_{max} - \kappa_{min})(v - v_{min})}{v_{max} - v_{min}} $$
    Let's re-read the snippet: `f (v, vmax , vmin ) = κmin (v − vmin ) κmax − κmin vmax − vmin`.
    Actually, looking at the layout, it might be:
    $$ \kappa = \kappa_{min} \frac{v - v_{min}}{v_{max} - v_{min}} + \dots $$
    No, the most logical interpretation of "linear mapping" with bounds $\kappa_{min}$ and $\kappa_{max}$ is the standard interpolation. The source text is poorly formatted. Let's look at the context: "Areas with larger variance require a larger value of $\kappa$."
    If $v = v_{min}$, $\kappa$ should be $\kappa_{min}$.
    If $v = v_{max}$, $\kappa$ should be $\kappa_{max}$.
    The formula provided in the source is:
    `f (v, vmax , vmin ) = κmin (v − vmin ) κmax − κmin vmax − vmin`
    This looks like a typo for:
    $$ \kappa = \kappa_{min} + \frac{(\kappa_{max} - \kappa_{min})(v - v_{min})}{v_{max} - v_{min}} $$
    Or potentially:
    $$ \kappa = \kappa_{min} \left( 1 - \frac{v - v_{min}}{v_{max} - v_{min}} \right) + \kappa_{max} \left( \frac{v - v_{min}}{v_{max} - v_{min}} \right) $$
    Given the text "Areas with larger variance require a larger value of $\kappa$", and $\kappa_{max} > \kappa_{min}$, the coefficient increases with variance.
    The source explicitly lists $\kappa_{max}=0.3$ and $\kappa_{min}=0.1$.
    We will use the standard linear interpolation formula which satisfies the boundary conditions and the monotonicity requirement [S1, unofficial].

3.  **Calculate Step Size:** Compute the step size $\Delta t$ for the brick using the coefficient $\kappa$ and the voxel dimensions:
    $$ \Delta t_{ijk} = \frac{\min(dx, dy, dz)}{\kappa_{ijk}} $$
    where $dx, dy, dz$ are the physical lengths of a single voxel in the x, y, and z directions [S1, unofficial].

**Parameters:**
*   $\kappa_{max}$: Maximum kappa value. Typical value: 0.3 [S1, unofficial].
*   $\kappa_{min}$: Minimum kappa value. Typical value: 0.1 [S1, unofficial].
*   Brick Size: Experimentally determined. The source shows performance tests for block sizes like $4 \times 4$, $8 \times 8$, $16 \times 16$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_variance(volume_data, brick_dims):
    """
    Preprocess volume data to calculate variance for each brick.
    volume_data: 3D array of voxel values.
    brick_dims: tuple (bx, by, bz) defining the brick size.
    Returns: 3D array of variances, one per brick.
    """
    vol_shape = volume_data.shape
    num_bricks_x = (vol_shape[0] + brick_dims[0] - 1) // brick_dims[0]
    num_bricks_y = (vol_shape[1] + brick_dims[1] - 1) // brick_dims[1]
    num_bricks_z = (vol_shape[2] + brick_dims[2] - 1) // brick_dims[2]
    
    variance_map = np.zeros((num_bricks_x, num_bricks_y, num_bricks_z))
    
    for i in range(num_bricks_x):
        for j in range(num_bricks_y):
            for k in range(num_bricks_z):
                # Extract brick data, handling boundaries
                x_start = i * brick_dims[0]
                y_start = j * brick_dims[1]
                z_start = k * brick_dims[2]
                
                x_end = min(x_start + brick_dims[0], vol_shape[0])
                y_end = min(y_start + brick_dims[1], vol_shape[1])
                z_end = min(z_start + brick_dims[2], vol_shape[2])
                
                brick_data = volume_data[x_start:x_end, y_start:y_end, z_start:z_end]
                
                # Calculate variance
                n = brick_data.size
                if n == 0:
                    variance_map[i, j, k] = 0
                    continue
                    
                mean_val = np.mean(brick_data)
                variance = np.mean((brick_data - mean_val) ** 2)
                variance_map[i, j, k] = variance
                
    return variance_map

def compute_step_sizes(variance_map, voxel_dims, kappa_min, kappa_max):
    """
    Compute adaptive step sizes based on variance map.
    variance_map: 3D array of variances.
    voxel_dims: tuple (dx, dy, dz).
    kappa_min, kappa_max: constants.
    Returns: 3D array of step sizes.
    """
    v_min = np.min(variance_map)
    v_max = np.max(variance_map)
    
    # Avoid division by zero if all variances are identical
    if v_max == v_min:
        kappa = (kappa_min + kappa_max) / 2.0
        step_sizes = np.full_like(variance_map, np.min(voxel_dims) / kappa)
        return step_sizes
        
    # Linear mapping from variance to kappa
    # kappa = kappa_min + (kappa_max - kappa_min) * (v - v_min) / (v_max - v_min)
    normalized_v = (variance_map - v_min) / (v_max - v_min)
    kappa = kappa_min + (kappa_max - kappa_min) * normalized_v
    
    # Calculate step size
    min_voxel_dim = np.min(voxel_dims)
    step_sizes = min_voxel_dim / kappa
    
    return step_sizes

def ray_march_with_variance(ray_origin, ray_direction, volume_data, step_sizes_map, brick_dims, voxel_dims, max_dist, epsilon):
    """
    Ray marching loop using precomputed variance-based step sizes.
    """
    t = 0.0
    max_steps = 1000 # Arbitrary limit for safety
    min_voxel_dim = np.min(voxel_dims)
    
    # Precompute brick dimensions in world space
    brick_size_x = brick_dims[0] * voxel_dims[0]
    brick_size_y = brick_dims[1] * voxel_dims[1]
    brick_size_z = brick_dims[2] * voxel_dims[2]
    
    for _ in range(max_steps):
        if t > max_dist:
            return False, t
            
        p = ray_origin + ray_direction * t
        
        # Determine which brick the current point is in
        # Assuming origin aligned with grid
        bx = int(p[0] / brick_size_x)
        by = int(p[1] / brick_size_y)
        bz = int(p[2] / brick_size_z)
        
        # Check bounds
        if not (0 <= bx < step_sizes_map.shape[0] and 
                0 <= by < step_sizes_map.shape[1] and 
                0 <= bz < step_sizes_map.shape[2]):
            break
            
        # Get step size for this brick
        dt = step_sizes_map[bx, by, bz]
        
        # Safety clamp (optional, based on parent principles)
        # dt = max(dt, min_voxel_dim / 10.0) # Example guard
        
        # Sample at current and next position
        # Note: The source uses sparse traversal which samples at t_prev and t_curr
        # and checks if iso-surface is between them.
        
        # For simplicity, we just advance. In a full implementation, 
        # one would check for surface intersection between t and t+dt.
        
        # Advance
        t += dt
        
    return False, t
```

**Edge Cases and Pitfalls:**
*   **Uniform Data:** If $v_{max} = v_{min}$, the denominator in the mapping function becomes zero. The implementation must handle this by using a default $\kappa$ or a small epsilon [S1, unofficial].
*   **Brick Boundaries:** The step size is constant within a brick. If a ray crosses a brick boundary, the step size changes abruptly. This may require clamping the step size to ensure the ray does not overshoot the brick boundary significantly, or the traversal logic must account for brick boundaries explicitly (e.g., using sparse traversal which checks for surface intersection between samples) [S1, unofficial].
*   **Precision:** The variance calculation involves squaring differences, which can lead to precision issues for large datasets with large value ranges. Using double precision for variance calculation is recommended [G].

## Subtopics

### Sparse Traversal
A traversal method that uses the adaptive step size to skip over empty or uniform regions, checking for surface intersections only at sample points. It is chosen when performance is critical and some visual artifacts from skipping are acceptable. [Link](2312-adaptive-step-size-ray-marching.md)

### Local Traversal
A variant of sparse traversal that uses frame-to-frame coherence (prediction buffer) to further accelerate traversal. It is chosen for interactive applications with rotating datasets. [Link](2442-dynamic-sampling-step-adjustment.md)

## Operating characteristics
*   **Time Complexity:** Preprocessing is $O(N)$ where $N$ is the number of voxels, performed once on the CPU. Ray traversal is $O(M)$ where $M$ is the number of steps, but the number of steps is reduced in high-variance regions compared to fixed-step methods that must use the smallest step size required by the most complex region.
*   **Memory Complexity:** Requires storing the variance map, which is $O(N_{bricks})$ where $N_{bricks}$ is the number of bricks. This is significantly smaller than the volume data itself.
*   **Measured Performance:** The source reports performance improvements on NVIDIA CUDA hardware. For example, on the "Bucky" dataset, a $4 \times 4$ brick size achieved 31 FPS, while $8 \times 8$ achieved 61 FPS. On the "Engine" dataset, $8 \times 8$ achieved 31 FPS. Performance scales with brick size up to a point, after which it plateaus or decreases slightly due to cache effects [S1, unofficial].
*   **Strengths:** Eliminates the need for expensive runtime normal calculations. Enhances interactivity for rotating datasets. Effectively utilizes GPU memory hierarchy.
*   **Limitations:** Introduces a slight decrease in image correctness during rotation, though described as negligible. Requires CPU preprocessing. The choice of brick size affects performance and quality.
*   **When to Use:** Interactive volume rendering of large datasets where iso-surface normals change frequently, and precomputation is acceptable.
*   **When Not to Use:** Real-time applications where preprocessing time is prohibitive, or datasets with very low variance where fixed-step methods are sufficient.

## Background
The method was introduced by Braley et al. in 2010 in the paper "GPU Accelerated Voxel Traversal using the Prediction Buffer" [S1, unofficial]. It was developed to enhance the performance of sparse and local traversal techniques for isosurface volume rendering on GPUs. The authors noted that future work would focus on refining the mapping function from variance to step size [S1, unofficial].

## Sources
[S1] GPU Accelerated Voxel Traversal using the Prediction Buffer, https://people.cs.vt.edu/yongcao/publication/pdf/Braley10.pdf, fetched 2026-10-25 (unofficial)
