# Safe Raymarching Step Size

Also called: Conservative stepping, Grid Step Size, Ray Marching with SDF-guided Step Size, TensorBase Step Size Calculation

## Overview
Safe Raymarching Step Size is a method for determining the increment $\Delta t$ along a ray $r(t) = \mathbf{o} + t\mathbf{d}$ to ensure the marching process does not overshoot or penetrate implicit surfaces defined by Signed Distance Fields (SDFs) [S2, S3, unofficial]. It solves the problem of geometric artifacts caused by step sizes exceeding the local distance to the surface, particularly when the SDF is distorted by operations like domain repetition or boolean combinations [S1, unofficial]. It sets itself apart from standard sphere tracing by explicitly incorporating clamping, configurable aggressiveness ratios, and robust termination masks to handle edge cases such as immediate surface hits or boundary escapes [S2, unofficial].

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and normalized direction $\mathbf{d}$ [Parent].
- Scene bounding box (AABB) defined by $\mathbf{min}$ and $\mathbf{max}$ [S3, unofficial].
- Grid resolution $N_{grid}$ (voxel count per axis) [S3, unofficial].
- SDF evaluation function $f(\mathbf{p})$ [S2, unofficial].
- Parameters:
  - `clamp_dist`: Float threshold for clamping SDF values. Typical value: $0.1$ [S2, unofficial].
  - `ray_marching_ratio`: Float multiplier for step size aggressiveness. Typical value: $1.5$ [S2, unofficial].
  - `threshold`: Float threshold for surface convergence. Typical value: $5 \times 10^{-5}$ [S2, unofficial].
  - `march_step`: Integer maximum number of iterations [S2, unofficial].

**Outputs:**
- List of sampled points $\mathbf{p}_i$ [S2, unofficial].
- List of accumulated depths $t_i$ [S2, unofficial].
- List of SDF values $f_i$ [S2, unofficial].
- Boolean mask `valid_mask_render` indicating valid rays [S2, unofficial].

**Data Structures:**
- `marching_zdepth`: Tensor of shape $(N_{rays})$, initialized to $0$ [S2, unofficial].
- `sdf_list`, `points_list`, `marching_zdepth_list`: Lists to accumulate results [S2, unofficial].
- `valid_mask_max_marching_zdepth`: Boolean tensor [S2, unofficial].
- `valid_mask_ray_marching`: Boolean tensor [S2, unofficial].
- `valid_mask_first_query`: Boolean tensor [S2, unofficial] [G].

**Steps:**

1.  **Calculate Grid Step Size (Grid-based variant only):**
    If using a grid-based approach [S3, unofficial]:
    - Calculate inverse AABB size: $\mathbf{inv\_size} = \frac{1}{\mathbf{max} - \mathbf{min}}$.
    - Calculate unit size per grid cell: $\mathbf{unit\_size} = \mathbf{inv\_size} / N_{grid}$.
    - Calculate diagonal length of AABB: $L_{diag} = \|\mathbf{max} - \mathbf{min}\|$.
    - Calculate step size: $\Delta t_{grid} = \text{mean}(\mathbf{unit\_size}) \times \text{step\_ratio}$ [S3, unofficial].
    - Calculate number of samples: $N_{samples} = \lceil L_{diag} / \Delta t_{grid} \rceil + 1$ [S3, unofficial].
    - *Note: In SDF-based variants, $\Delta t$ is derived from the SDF value itself, not the grid [S2, unofficial].*

2.  **Initialize Marching State:**
    - Set `marching_zdepth` = $0$ for all rays [S2, unofficial].
    - Initialize empty lists `sdf_list`, `points_list`, `marching_zdepth_list` [S2, unofficial].

3.  **Iterate for `march_step` iterations:**
    For each iteration $i$:
    a.  **Sample Points:**
        - Compute current depth: $t_{current} = t_{init} + \text{marching\_zdepth}$ [S2, unofficial].
        - Compute positions: $\mathbf{p} = \mathbf{o} + t_{current} \mathbf{d}$ [S2, unofficial].
    b.  **Evaluate SDF:**
        - Compute $f_{raw} = f(\mathbf{p})$ [S2, unofficial].
        - *If using neural decoder, detach gradients if `no_grad` is True* [S2, unofficial].
    c.  **Clamp SDF:**
        - Compute $f_{clamped} = \text{clamp}(f_{raw}, -\text{clamp\_dist}, \text{clamp\_dist})$ [S2, unofficial].
    d.  **Update Depth (Step):**
        - Compute step increment: $\Delta t = f_{clamped} \times \text{ray\_marching\_ratio}$ [S2, unofficial].
        - Update `marching_zdepth`: $\text{marching\_zdepth} \leftarrow \text{marching\_zdepth} + \Delta t$ [S2, unofficial].
        - *Safety Note:* To mitigate artifacts from distorted SDFs, the step size should be conservative (i.e., `ray_marching_ratio` $\le 1.0$ or adjusted based on local distortion) [S1, unofficial].
    e.  **Store Data:**
        - Append $\mathbf{p}$ to `points_list` [S2, unofficial].
        - Append $t_{current}$ to `marching_zdepth_list` [S2, unofficial].
        - Append $f_{raw}$ to `sdf_list` [S2, unofficial].

4.  **Compute Termination Masks:**
    a.  **Max Bound Check:**
        - Compute `maxbound_zdepth` using camera position and valid rays [S2, unofficial].
        - `valid_mask_max_marching_zdepth` = $(\text{final\_depth} + t_{init} < \text{maxbound\_zdepth})$ [S2, unofficial].
    b.  **Surface Hit Check:**
        - Compute min SDF across all steps: $f_{min} = \min_i (f_i)$ [S2, unofficial].
        - `valid_mask_ray_marching` = $(f_{min} \le \text{threshold})$ [S2, unofficial].
    c.  **First Query Check (Edge Case):**
        - If the first SDF query $f_0 \le \text{threshold}$, the ray starts inside or on the surface.
        - `valid_mask_first_query` = $(f_0 > \text{threshold})$ [S2, unofficial].
        - *Pitfall:* This mask filters out immediate hits to prevent incorrect rendering logic, requiring special handling for rays starting inside geometry [S2, unofficial].
    d.  **Combine Masks:**
        - `valid_mask_render` = `valid_mask_max_marching_zdepth` AND `valid_mask_ray_marching` AND `valid_mask_first_query` [S2, unofficial].

5.  **Return Results:**
    - Concatenate lists along dimension 0 to form tensors of shape $(\text{march\_step}, N_{rays})$ [S2, unofficial].
    - Return `sdf_list`, `marching_zdepth_list`, `points_list`, `valid_mask_render` [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def safe_raymarching_step_size(o, d, f, aabb_min, aabb_max, grid_size, 
                               clamp_dist=0.1, ray_marching_ratio=1.5, 
                               threshold=5e-5, march_step=100):
    """
    o: (N, 3) ray origins
    d: (N, 3) normalized ray directions
    f: callable SDF function f(p) -> (N,)
    aabb_min, aabb_max: (3,) bounding box corners
    grid_size: int, resolution per axis
    """
    N = o.shape[0]
    marching_zdepth = torch.zeros(N)
    sdf_list = []
    points_list = []
    zdepth_list = []
    
    # Precompute max bound for escape check
    # Assuming maxbound is derived from AABB intersection or camera bounds
    # Simplified: use AABB diagonal as max possible depth for simplicity if not provided
    diag_len = torch.norm(aabb_max - aabb_min)
    maxbound_zdepth = diag_len 
    
    for i in range(march_step):
        # 1. Sample Points
        current_depth = marching_zdepth
        points = o + current_depth.unsqueeze(1) * d
        
        # 2. Evaluate SDF
        sdf_raw = f(points)
        
        # 3. Clamp SDF
        sdf_clamped = torch.clamp(sdf_raw, -clamp_dist, clamp_dist)
        
        # 4. Update Depth
        step = sdf_clamped * ray_marching_ratio
        marching_zdepth = marching_zdepth + step
        
        # 5. Store Data
        points_list.append(points)
        zdepth_list.append(current_depth)
        sdf_list.append(sdf_raw)
        
        # Early exit optimization (optional, based on sibling methods)
        # If all rays have hit or escaped, break
        
    # 6. Compute Masks
    # Max bound check
    final_depth = marching_zdepth
    valid_max_bound = final_depth < maxbound_zdepth
    
    # Surface hit check: min SDF across all steps <= threshold
    sdf_tensor = torch.stack(sdf_list, dim=0) # (march_step, N)
    min_sdf = torch.min(sdf_tensor, dim=0).values
    valid_hit = min_sdf <= threshold
    
    # First query check: first SDF > threshold (to filter out immediate hits)
    first_sdf = sdf_tensor[0]
    valid_first = first_sdf > threshold
    
    # Combine
    valid_mask = valid_max_bound & valid_hit & valid_first
    
    # Concatenate lists
    points_tensor = torch.stack(points_list, dim=0)
    zdepth_tensor = torch.stack(zdepth_list, dim=0)
    
    return sdf_tensor, zdepth_tensor, points_tensor, valid_mask
```

## Subtopics

### [Distance Accuracy vs Performance Trade-off](1546-distance-accuracy-vs-performance-trade-off.md)
Analyzes the inverse relationship between SDF evaluation accuracy and step count; underestimating distance forces smaller steps, increasing iterations.

### [Adaptive Lipschitz Constant (ALC)](1883-adaptive-lipschitz-constant-alc.md)
Uses a nested BVH to compute tighter, adaptive Lipschitz constants for step size, avoiding overly conservative global bounds.

### [Gradient-based Sphere Tracing with Lipschitz Constraints](1936-gradient-based-sphere-tracing-with-lipschitz-constraints.md)
Uses local SDF gradients to compute larger, safer step sizes, accounting for curvature.

### [Lipschitz Step Size Ray Marching](3729-lipschitz-step-size-ray-marching.md)
Uses a pre-computed global Lipschitz constant $\lambda$ to guarantee no penetration, simple but potentially less efficient.

## Operating characteristics
- **Time Complexity:** $O(N_{rays} \times N_{steps})$, where $N_{steps}$ depends on the step size accuracy and scene complexity [S2, unofficial].
- **Memory Complexity:** $O(N_{rays} \times N_{steps})$ to store intermediate SDF and position data for backpropagation or rendering [S2, unofficial].
- **Performance:** Parallel implementations (`ray_marching_trivial`) process all rays in a single loop, while recursive variants (`ray_marching_recursive`) only process unfinished rays, optimizing for early hits [S2, unofficial].
- **Strengths:** Robust against overshooting in distorted SDFs; handles edge cases like immediate surface hits via masks [S1, S2, unofficial].
- **Limitations:** Conservative step sizes (low `ray_marching_ratio`) increase iteration counts; clamping may reduce accuracy near the surface [S2, unofficial].
- **When to Use:** When rendering distorted SDFs or neural implicit surfaces where standard sphere tracing may overshoot [S1, S2, unofficial].
- **When Not to Use:** When high performance is critical and the SDF is well-behaved (Lipschitz constant $\lambda=1$), standard sphere tracing is more efficient [S1, unofficial].

## Background
Safe Raymarching Step Size originated from the need to render distorted Signed Distance Fields, such as those created by domain repetition or boolean operations, which can cause artifacts if the step size exceeds the local distance [S1, unofficial]. The method was formalized in implementations like the DIST-Renderer, which introduced clamping and ratio-based step control to stabilize differentiable rendering [S2, unofficial]. It is also referenced in collaborative neural radiance field implementations for novel view synthesis, where grid-based step sizes are calculated from AABB dimensions [S3, unofficial].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S3] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
