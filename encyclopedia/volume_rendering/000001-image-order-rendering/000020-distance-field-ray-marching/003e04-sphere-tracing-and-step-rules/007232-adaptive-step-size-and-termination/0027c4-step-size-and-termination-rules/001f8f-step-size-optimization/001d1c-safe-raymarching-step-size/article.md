# Safe Raymarching Step Size

Also called: Conservative stepping, Distance-Aided Ray Marching, Grid Step Size, Ray Marching with SDF-guided Step Size, TensorBase Step Size Calculation

## Overview
Safe Raymarching Step Size is a technique for determining the step length $\Delta t$ along a ray in volume rendering or implicit surface rendering by using the local distance to the nearest geometry as an upper bound [S3, unofficial]. It solves the problem of "popping" artifacts or missed intersections that occur when fixed step sizes are too large to resolve fine details or when distorted distance fields overestimate the distance to the surface [S1, unofficial]. Unlike fixed-step methods, it adapts the step size to the local distance to the nearest object, ensuring the ray never penetrates the surface or skips over significant volumetric features [S3, unofficial]. It is distinguished from [Adaptive Step Size Control](9843-adaptive-step-size-control.md) by its reliance on the geometric distance field value itself as the primary driver of step size, rather than error estimates or density gradients alone [S2, S3, unofficial].

## How it works
**Inputs:**
*   Ray origin $\mathbf{o}$ and normalized direction $\mathbf{d}$ [G].
*   Scene function $f(\mathbf{p})$ returning the Signed Distance Field (SDF) value or distance bound at point $\mathbf{p}$ [S2, S3, unofficial].
*   Maximum marching distance $t_{max}$ (or `maxbound_zdepth`) [S2, unofficial].
*   SDF threshold $\epsilon$ (e.g., $5 \times 10^{-5}$) for surface hit detection [S2, unofficial].
*   Clamping distance $d_{clamp}$ (e.g., $0.1$) to limit extreme step sizes [S2, unofficial].
*   Ray marching ratio $\lambda$ (e.g., $1.5$) to scale the step size for safety or aggressiveness [S2, unofficial].
*   Grid parameters (optional, for grid-based SDFs): Axis-Aligned Bounding Box (AABB) defined by `min` and `max`, and grid resolution `gridSize` [S4, unofficial].

**Outputs:**
*   List of sampled points, SDF values, and accumulated depths [S2, unofficial].
*   A validity mask indicating which rays successfully hit the surface or remained within bounds [S2, unofficial].

**Data Structures:**
*   `marching_zdepth`: List of floats storing the accumulated distance along each ray, initialized to $0.0$ [S2, unofficial].
*   `sdf_list`: List of lists, each inner list of length $N$, storing the SDF value at each step [S2, unofficial].
*   `points_list`: List of lists, each inner list of length $N$, storing the 3D coordinates of samples [S2, unofficial].
*   `valid_mask_render`: Boolean list of length $N$ indicating valid rays [S2, unofficial].

**Step-by-Step Algorithm:**

1.  **Initialization:**
    *   Initialize `marching_zdepth` to zeros for all valid rays [S2, unofficial].
    *   Initialize empty lists `sdf_list`, `points_list`, and `marching_zdepth_list` [S2, unofficial].
    *   Compute `maxbound_zdepth` based on the camera position and ray directions, representing the maximum valid distance within the scene bounds [S2, unofficial].

2.  **Marching Loop:**
    For each step $i$ from $0$ to `march_step - 1`:
    *   **Sample Points:** Calculate the current 3D position for each ray:
        $$ \mathbf{p}_i = \mathbf{o} + \mathbf{d} \times (\text{init\_zdepth} + \text{marching\_zdepth}) $$
        where `init_zdepth` is the initial offset (often $0.0$) [S2, unofficial].
    *   **Evaluate SDF:** Query the scene function $f(\mathbf{p}_i)$ to get the distance to the nearest object [S2, unofficial]. If using a neural decoder, detach gradients if differentiability is not required for this step [S2, unofficial].
    *   **Clamp SDF:** To prevent instability from distorted fields or extreme values, clamp the SDF value:
        $$ f_{clamped} = \max(-d_{clamp}, \min(d_{clamp}, f(\mathbf{p}_i))) $$
        Typical $d_{clamp}$ is $0.1$ [S2, unofficial].
    *   **Compute Step Size:**
        *   *Grid-based:* If using a voxel grid, calculate the unit size per grid cell as $\frac{\text{AABB\_size}}{\text{gridSize}}$. The step size is the mean unit size multiplied by a `step_ratio` [S4, unofficial].
        *   *SDF-based:* Calculate the step size as the product of the clamped SDF and the ray marching ratio:
            $$ \Delta t = f_{clamped} \times \lambda $$
            Typical $\lambda$ is $1.5$ [S2, unofficial]. Note: Some implementations use a fixed step derived from grid resolution [S4, unofficial].
    *   **Update Depth:**
        $$ \text{marching\_zdepth} \leftarrow \text{marching\_zdepth} + \Delta t $$
    *   **Store Data:** Append $f(\mathbf{p}_i)$ to `sdf_list`, $\mathbf{p}_i$ to `points_list`, and the new `marching_zdepth` to `marching_zdepth_list` [S2, unofficial].

3.  **Termination and Validation:**
    After the loop (or if early termination conditions are met in a variant):
    *   **Max Distance Check:** Compute `valid_mask_max_marching_zdepth` by checking if $\text{marching\_zdepth} + \text{init\_zdepth} < \text{maxbound\_zdepth}$ [S2, unofficial].
    *   **Surface Hit Check:** Compute `valid_mask_ray_marching` by checking if the minimum absolute SDF across all steps is $\le \epsilon$ [S2, unofficial].
    *   **First Query Check:** Compute `valid_mask_first_query` by checking if the first SDF query $> \epsilon$. This handles the corner case where the ray starts inside the surface or exactly on it, preventing incorrect filtering [S2, unofficial].
    *   **Final Mask:** Combine masks:
        $$ \text{valid\_mask\_render} = \text{valid\_mask\_max\_marching\_zdepth} \land \text{valid\_mask\_ray\_marching} \land \text{valid\_mask\_first\_query} $$
    *   Return `sdf_list`, `marching_zdepth_list`, `points_list`, and `valid_mask_render` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def safe_raymarch_step_size(ray_origins, ray_directions, scene_func, 
                            max_bound_dist, threshold, clamp_dist, 
                            ray_marching_ratio, march_step, init_zdepth=0.0):
    """
    Performs safe raymarching using SDF-guided step sizes.
    ray_origins: list of (x, y, z) tuples
    ray_directions: list of (dx, dy, dz) tuples
    max_bound_dist: list of floats
    """
    n = len(ray_origins)
    marching_zdepth = [0.0] * n
    sdf_list = []
    points_list = []
    marching_zdepth_list = []

    for step_idx in range(march_step):
        current_sdfs = []
        current_points = []
        for j in range(n):
            # 1. Calculate current 3D position
            dist = init_zdepth + marching_zdepth[j]
            p = (ray_directions[j][0] * dist + ray_origins[j][0],
                 ray_directions[j][1] * dist + ray_origins[j][1],
                 ray_directions[j][2] * dist + ray_origins[j][2])

            # 2. Evaluate SDF
            sdf_val = scene_func(p)

            # 3. Clamp SDF to prevent instability from distorted fields
            sdf_clamped = max(-clamp_dist, min(clamp_dist, sdf_val))

            # 4. Compute step size
            step = sdf_clamped * ray_marching_ratio

            # 5. Update accumulated depth
            marching_zdepth[j] += step

            # 6. Store data
            current_sdfs.append(sdf_val)
            current_points.append(p)

        sdf_list.append(current_sdfs)
        points_list.append(current_points)
        marching_zdepth_list.append(list(marching_zdepth))

    # 7. Validation Masks
    valid_mask_render = []
    for j in range(n):
        # Check if rays stayed within max bound
        final_depth = init_zdepth + marching_zdepth[j]
        valid_max = final_depth < max_bound_dist[j]

        # Check if any step was close to surface (min absolute SDF <= threshold)
        min_abs_sdf = min(abs(sdf_list[k][j]) for k in range(march_step))
        valid_hit = min_abs_sdf <= threshold

        # Check if first query was > threshold (handles start-inside-surface case)
        first_sdf = sdf_list[0][j]
        valid_first = first_sdf > threshold

        # Combine masks
        valid_mask_render.append(valid_max and valid_hit and valid_first)

    return sdf_list, marching_zdepth_list, points_list, valid_mask_render
```

**Edge Cases and Pitfalls:**
*   **Distorted SDFs:** Domain repetition or boolean operations can distort the distance field, causing the estimated distance to be larger than the actual distance to the surface. This leads to the ray skipping over geometry [S1, unofficial]. Mitigation: Step shorter than the SDF returns (e.g., use a ratio $< 1.0$ or clamp aggressively) [S1, unofficial].
*   **Start Inside Surface:** If the ray origin is inside the geometry, the first SDF query may be below the threshold. The `valid_mask_first_query` logic ensures these rays are not incorrectly filtered out or handled as misses depending on the rendering logic [S2, unofficial].
*   **Boundary Exceedance:** Rays must not exceed the `maxbound_zdepth`. This is strictly checked against the final accumulated depth [S2, unofficial].
*   **Precision:** Using a `ray_marching_ratio` greater than 1.0 (e.g., $1.5$) can speed up convergence but risks overshooting if the SDF is not a true distance field or is distorted [S2, unofficial].

## Subtopics

### [Distance Accuracy vs Performance Trade-off](1546-distance-accuracy-vs-performance-trade-off.md)
Analyzes the inverse relationship between SDF evaluation accuracy and step count. Underestimating distance forces smaller steps, increasing iterations. An exact but slower SDF can outperform an inaccurate but fast one if the inaccuracy causes disproportionate step reduction.

### [Adaptive Lipschitz Constant (ALC)](1883-adaptive-lipschitz-constant-alc.md)
Uses a nested BVH to compute tighter, adaptive Lipschitz constants for step size calculation, solving the problem of overly conservative steps in large scenes where global constants are too large.

### [Gradient-based Sphere Tracing with Lipschitz Constraints](1936-gradient-based-sphere-tracing-with-lipschitz-constraints.md)
Uses the local SDF gradient to compute larger, safer step sizes, particularly near silhouettes. Combines first and second Lipschitz constraints to account for curvature.

### [Lipschitz Step Size Ray Marching](3729-lipschitz-step-size-ray-marching.md)
A robust method using a pre-computed global Lipschitz constant $\lambda$ to guarantee no penetration or overshooting. Simple and stable but potentially less efficient for complex scenes with varying feature scales.

## Operating characteristics
*   **Time Complexity:** $O(N \times S)$, where $N$ is the number of rays and $S$ is the number of steps [G]. The number of steps $S$ is adaptive and depends on the scene's geometry and the chosen `ray_marching_ratio` [S2, unofficial].
*   **Memory Complexity:** $O(N \times S)$ to store the history of SDF values and points for rendering or differentiable backpropagation [S2, unofficial].
*   **Strengths:** Prevents popping artifacts and missed intersections by respecting the geometric distance bound [S3, unofficial]. Simple to implement for SDF-based rendering [S1, unofficial].
*   **Limitations:** Sensitive to SDF distortion [S1, unofficial]. Requires a valid distance field or distance bound [G]. Not directly applicable to non-SDF volume data without adaptation (e.g., using density gradients) [S2, unofficial].
*   **When to Use:** Ideal for rendering implicit surfaces defined by SDFs where visual correctness is critical and the SDF is well-behaved [G]. Also used in neural rendering pipelines (e.g., DeepSDF) where differentiable SDF evaluation is required [S2, unofficial].
*   **When Not to Use:** When the scene is defined by explicit meshes (use ray-triangle intersection) or when the SDF is highly distorted and cannot be reliably clamped [S1, unofficial].

## Background
Safe Raymarching Step Size, often referred to as Distance-Aided Ray Marching, originated in the demoscene as a method to efficiently render complex 3D scenes using Signed Distance Fields [S3, unofficial]. The core insight, attributed to Adok, is that the distance to the nearest object provides a safe upper bound for the step size, ensuring no geometry is skipped [S3, unofficial]. This technique was later adopted in academic and industrial rendering, particularly in the context of neural implicit representations like DeepSDF, where it is used to sample points along rays for differentiable rendering [S2, unofficial]. Variants like Lipschitz-constrained stepping [S1, S3, unofficial] and adaptive methods [S1, unofficial] have been developed to address limitations in complex scenes.

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S3] http://www.hugi.scene.org/online/hugi37/hugi%2037%20-%20coding%20adok%20on%20ray%20casting,%20ray%20tracing,%20ray%20marching%20and%20the%20like.htm, http://www.hugi.scene.org/online/hugi37/hugi%2037%20-%20coding%20adok%20on%20ray%20casting,%20ray%20tracing,%20ray%20marching%20and%20the%20like.htm, fetched 2026-10-26 (unofficial)
[S4] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
