# Safe Raymarching Step Size

Also called: Conservative stepping, Grid Step Size, Ray Marching with SDF-guided Step Size, TensorBase Step Size Calculation

## Overview
Safe Raymarching Step Size is a method for determining the step length in distance-field ray marching that guarantees the ray never overshoots or penetrates the implicit surface, even when the Signed Distance Function (SDF) is distorted or approximated [S1, S2, unofficial]. It solves the problem of artifacts caused by domain repetition or boolean operations, where the estimated distance may exceed the actual distance to the geometry [S1, unofficial]. It sets itself apart from standard sphere tracing by explicitly clamping SDF values and applying a conservative ratio to the step increment [S2, unofficial].

## How it works
**Inputs:**
- Camera position $\mathbf{c}$ and ray directions $\mathbf{r}$ for all pixels [S2, unofficial].
- SDF decoder or evaluator $f(\mathbf{p})$ [S2, unofficial].
- Axis-Aligned Bounding Box (AABB) of the scene, defined by $\mathbf{aabb}_{min}$ and $\mathbf{aabb}_{max}$ [S3, unofficial].
- Parameters:
  - `clamp_dist`: Float threshold for clamping SDF values. Typical value: $0.1$ [S2, unofficial].
  - `ray_marching_ratio`: Float multiplier applied to the clamped SDF to determine step size. Typical value: $1.5$ [S2, unofficial].
  - `threshold`: Float SDF value threshold for surface intersection. Typical value: $5 \times 10^{-5}$ [S2, unofficial].
  - `march_step`: Integer maximum number of iterations [S2, unofficial].

**Outputs:**
- `sdf_list`: List of lists containing SDF values at each step for $N$ rays and $M$ steps [S2, unofficial].
- `marching_zdepth_list`: List of lists containing accumulated depth [S2, unofficial].
- `points_list`: List of lists containing 3D points sampled along rays [S2, unofficial].
- `valid_mask_render`: Boolean list indicating valid rays that have not exceeded bounds or hit the surface incorrectly [S2, unofficial].

**Data Structures:**
- Lists are used to accumulate data per step. `marching_zdepth` is initialized to zeros for all valid rays [S2, unofficial].

**Algorithm:**

1.  **Initialization:**
    - Initialize `marching_zdepth` to a list of zeros with length $N$, where $N$ is the number of valid rays [S2, unofficial].
    - Initialize empty lists: `sdf_list`, `marching_zdepth_list`, `points_list` [S2, unofficial].
    - Calculate `maxbound_zdepth` using a helper function `get_maxbound_zdepth(cam_pos, valid_cam_rays)` [S2, unofficial]. This represents the maximum distance a ray can travel before exiting the scene bounds [S2, unofficial].

2.  **Iterative Marching Loop:**
    For $i$ from $0$ to `march_step` - 1:
    
    a.  **Point Generation:**
        - Calculate current ray origins: $\mathbf{p}_i = \mathbf{c} + (\text{init\_zdepth} + \text{marching\_zdepth}) \odot \mathbf{r}$ [S2, unofficial].
        - Store $\mathbf{p}_i$ in `points_list` [S2, unofficial].
        
    b.  **SDF Evaluation:**
        - Evaluate SDF: $d_i = f(\mathbf{p}_i)$ [S2, unofficial].
        - If gradients are not required, detach them [S2, unofficial].
        - Store $d_i$ in `sdf_list` [S2, unofficial].
        
    c.  **Clamping and Step Calculation:**
        - Clamp SDF values: $d_{clamped} = \max(-\text{clamp\_dist}, \min(d_i, \text{clamp\_dist}))$ [S2, unofficial].
        - Calculate step size $\Delta s = d_{clamped} \times \text{ray\_marching\_ratio}$ [S2, unofficial].
        - Note: Sources indicate `ray_marching_ratio` is typically $> 1.0$ (e.g., $1.5$) to ensure safety against distortion artifacts [S2, S1, unofficial].
        - Not given in the sources. (Grid-based step calculation was mentioned in notes but not provided in the source text for [S3, unofficial].)
            
    d.  **Depth Update:**
        - Update `marching_zdepth`: $\text{marching\_zdepth} \leftarrow \text{marching\_zdepth} + \Delta s$ [S2, unofficial].
        - Store updated `marching_zdepth` in `marching_zdepth_list` [S2, unofficial].
        
    e.  **Termination Check (Optional Optimization):**
        - If using recursive marching (`ray_marching_recursive`), only process rays where `valid_mask_render` is still true [S2, unofficial].
        - If using trivial marching (`ray_marching_trivial`), process all rays in parallel [S2, unofficial].

3.  **Post-Processing and Masking:**
    - Concatenate lists along dimension 0 to form tensors of shape $(M, N)$ [S2, unofficial].
    - Compute `valid_mask_max_marching_zdepth`: Check if $\text{marching\_zdepth} + \text{init\_zdepth} < \text{maxbound\_zdepth}$ [S2, unofficial].
    - Compute `valid_mask_ray_marching`: Check if $\min_i(|d_i|) \le \text{threshold}$ for any step $i$ [S2, unofficial].
    - Compute `valid_mask_first_query`: Check if $d_0 > \text{threshold}$ [S2, unofficial]. This handles the corner case where the first query is already below the threshold, preventing incorrect filtering [S2, unofficial].
    - Combine masks: `valid_mask_render` = `valid_mask_max_marching_zdepth` AND `valid_mask_ray_marching` AND `valid_mask_first_query` [S2, unofficial].

4.  **Return:**
    - Return `sdf_list`, `marching_zdepth_list`, `points_list`, and `valid_mask_render` [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def safe_raymarching_step_size(cam_pos, cam_rays, sdf_decoder, aabb_min, aabb_max, 
                               clamp_dist=0.1, ray_marching_ratio=1.5, 
                               threshold=5e-5, march_step=100):
    """
    Implements Safe Raymarching Step Size.
    
    Args:
        cam_pos: List of tuples (x, y, z), camera positions.
        cam_rays: List of tuples (dx, dy, dz), normalized ray directions.
        sdf_decoder: Function evaluating SDF at point p.
        aabb_min: Tuple (x, y, z), min corner of bounding box.
        aabb_max: Tuple (x, y, z), max corner of bounding box.
        clamp_dist: float, SDF clamping threshold.
        ray_marching_ratio: float, multiplier for step size.
        threshold: float, intersection threshold.
        march_step: int, max iterations.
        
    Returns:
        sdf_list, marching_zdepth_list, points_list, valid_mask_render
    """
    N = len(cam_rays)
    
    # Initialize
    marching_zdepth = [0.0] * N
    sdf_list = []
    marching_zdepth_list = []
    points_list = []
    
    # Calculate max bound depth
    maxbound_zdepth = get_maxbound_zdepth(cam_pos, cam_rays, aabb_min, aabb_max)
    
    for i in range(march_step):
        # 1. Generate points
        points = []
        for j in range(N):
            px = cam_pos[j][0] + marching_zdepth[j] * cam_rays[j][0]
            py = cam_pos[j][1] + marching_zdepth[j] * cam_rays[j][1]
            pz = cam_pos[j][2] + marching_zdepth[j] * cam_rays[j][2]
            points.append((px, py, pz))
        points_list.append(points)
        
        # 2. Evaluate SDF
        sdf_values = [sdf_decoder(p) for p in points]
        # Detach gradients if necessary (simulated by returning float)
        sdf_values = [float(v) for v in sdf_values]
        sdf_list.append(sdf_values)
        
        # 3. Clamp SDF
        sdf_clamped = [max(-clamp_dist, min(v, clamp_dist)) for v in sdf_values]
        
        # 4. Calculate Step Size
        step_size = [v * ray_marching_ratio for v in sdf_clamped]
        
        # 5. Update Depth
        marching_zdepth = [z + s for z, s in zip(marching_zdepth, step_size)]
        marching_zdepth_list.append(marching_zdepth)
        
    # 6. Post-processing
    # Lists are already accumulated per step
    
    # Compute masks
    final_zdepth = marching_zdepth
    valid_mask_max = [z < maxbound_zdepth[j] for j, z in enumerate(final_zdepth)]
    
    # min abs SDF <= threshold
    min_abs_sdf = [min(abs(sdf_list[k][j]) for k in range(march_step)) for j in range(N)]
    valid_mask_hit = [m <= threshold for m in min_abs_sdf]
    
    # valid_mask_first_query: first SDF > threshold
    first_sdf = sdf_list[0]
    valid_mask_first = [s > threshold for s in first_sdf]
    
    valid_mask_render = [valid_mask_max[j] and valid_mask_hit[j] and valid_mask_first[j] for j in range(N)]
    
    return sdf_list, marching_zdepth_list, points_list, valid_mask_render

def get_maxbound_zdepth(cam_pos, cam_rays, aabb_min, aabb_max):
    """
    Calculates the distance from cam_pos to the intersection of the ray with the AABB.
    Returns a list of floats.
    """
    maxbound = []
    for j in range(len(cam_rays)):
        inv_dir = [1.0 / cam_rays[j][k] if cam_rays[j][k] != 0 else float('inf') for k in range(3)]
        t1 = [(aabb_min[k] - cam_pos[j][k]) * inv_dir[k] for k in range(3)]
        t2 = [(aabb_max[k] - cam_pos[j][k]) * inv_dir[k] for k in range(3)]
        
        tmin = max(min(t1[k], t2[k]) for k in range(3))
        tmax = min(max(t1[k], t2[k]) for k in range(3))
        
        if tmax < 0:
            maxbound.append(float('inf'))
        else:
            maxbound.append(tmin if tmin >= 0 else float('inf'))
    return maxbound
```

**Edge Cases and Pitfalls:**
- **Distortion Artifacts:** Domain repetition or boolean operations can distort the SDF, causing the estimated distance to be larger than the actual distance. This leads to the ray skipping over geometry. The solution is to step shorter than the SDF returns, achieved by `ray_marching_ratio` > 1.0 or clamping [S1, unofficial].
- **Immediate Hit:** If the first SDF query is already below the threshold, `valid_mask_first_query` filters it out to prevent incorrect rendering logic [S2, unofficial].
- **Boundary Exceedance:** Rays must not exceed `maxbound_zdepth`. This is checked against the final accumulated depth [S2, unofficial].
- **Precision:** Clamping SDF values prevents extremely small or negative values from causing erratic step sizes [S2, unofficial].

## Subtopics

### [Distance Accuracy vs Performance Trade-off](1546-distance-accuracy-vs-performance-trade-off.md)
Analyzes the inverse relationship between SDF evaluation accuracy and computational cost. Underestimating distance forces smaller steps, increasing iterations. An exact SDF may outperform a fast, inaccurate bound if the inaccuracy causes disproportionate step reduction.

### [Adaptive Lipschitz Constant (ALC)](1883-adaptive-lipschitz-constant-alc.md)
Uses a nested BVH to compute tighter, adaptive Lipschitz constants for each step, solving the problem of overly conservative step sizes in large scenes. Unlike Safe Raymarching which may use a single global bound, ALC leverages geometric containment.

### [Gradient-based Sphere Tracing with Lipschitz Constraints](1936-gradient-based-sphere-tracing-with-lipschitz-constraints.md)
Uses local SDF gradients to compute larger, safer step sizes, particularly near silhouettes. Combines first and second Lipschitz constraints to account for curvature, differing from Safe Raymarching by leveraging gradient magnitude rather than fixed ratios.

### [Lipschitz Step Size Ray Marching](3729-lipschitz-step-size-ray-marching.md)
A robust algorithm using a pre-computed global Lipschitz constant $\lambda$ to guarantee no surface penetration. Simpler and numerically stable but potentially less efficient for complex scenes with varying feature scales compared to adaptive methods.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Parallelizable across rays using tensor operations [S2, unofficial].
- **Strengths:** Guarantees no overshoot even with distorted SDFs [S1, unofficial]. Simple to implement with fixed parameters [S2, unofficial].
- **Limitations:** The `ray_marching_ratio` must be tuned to balance safety and speed [S2, unofficial].
- **When to Use:** When rendering distorted SDFs (e.g., from boolean operations or domain repetition) [S1, unofficial]. When using neural SDF decoders where gradient information might be noisy or unavailable [S2, unofficial].
- **When Not to Use:** Not given in the sources.

## Background
Safe Raymarching Step Size originates from techniques used in ray marching distance fields to mitigate artifacts caused by SDF distortion [S1, unofficial]. The concept of stepping shorter than the SDF returns was proposed to handle cases where domain repetition or boolean operations cause the distance field to be inaccurate [S1, unofficial]. The implementation details, including clamping, ratio multipliers, and tensor-based parallel processing, are derived from differentiable rendering frameworks like DIST-Renderer [S2, unofficial] and Collaborative Neural Radiance Fields [S3, unofficial].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S3] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
