# Safe Raymarching Step Size

Also called: Conservative stepping, Grid Step Size, Ray Marching with SDF-guided Step Size, TensorBase Step Size Calculation

## Overview
Safe Raymarching Step Size is a method for determining the step length $\Delta t$ along a ray in volume rendering and implicit surface rendering to prevent the ray from overshooting geometry or density features [S2, S3, unofficial]. It solves the problem of artifacts caused by large steps that skip over thin structures or surface intersections, particularly when the distance field is distorted or when sampling neural implicit representations [S1, S2, unofficial]. It sets itself apart from its siblings by explicitly combining clamping of the distance value, a configurable ratio multiplier, and strict boundary checks to ensure numerical stability and correctness in both parallel and recursive marching implementations [S2, S3, unofficial].

## How it works
The method determines the step size $\Delta t_i$ based on the current position $\mathbf{p}_i$, the ray direction $\mathbf{d}$, and the local distance field value $f(\mathbf{p}_i)$. It operates on two primary data structures: a tensor-based batch of rays (for parallel execution) and a hierarchical recursive structure (for adaptive execution).

### Inputs and Outputs
**Inputs:**
*   Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ for each pixel/ray.
*   Initial depth $t_{init}$ (often 0 or camera near plane).
*   Signed Distance Field (SDF) function $f(\mathbf{p})$ or neural decoder.
*   Axis-Aligned Bounding Box (AABB) defining the scene bounds $[min, max]$ [S3, unofficial].
*   Grid resolution $N_{grid}$ if using voxel-based step sizing [S3, unofficial].
*   Parameters: `clamp_dist`, `ray_marching_ratio`, `threshold`, `march_step` [S2, unofficial].

**Outputs:**
*   List of sampled points $\mathbf{p}_i$.
*   Accumulated depth $t_i$.
*   SDF values $f(\mathbf{p}_i)$.
*   Validity mask indicating successful convergence or hit [S2, unofficial].

### Data Structures
1.  **Batch Tensors (Parallel):**
    *   `marching_zdepth`: Tensor of shape $(N_{rays})$, initialized to zeros. Stores accumulated distance $t_i$ [S2, unofficial].
    *   `sdf_list`: List of tensors, each of shape $(N_{rays})$, storing $f(\mathbf{p}_i)$ at each step [S2, unofficial].
    *   `points_list`: List of tensors, each of shape $(N_{rays}, 3)$, storing $\mathbf{p}_i$ [S2, unofficial].
    *   `valid_mask`: Boolean tensor tracking rays that have not yet hit the surface or escaped [S2, unofficial].

2.  **Recursive State (Adaptive):**
    *   State includes current $t$, current $\mathbf{p}$, and remaining budget. Only rays with $f(\mathbf{p}) > threshold$ are processed in subsequent steps [S2, unofficial].

### Step-by-Step Method

1.  **Initialization:**
    *   Initialize `marching_zdepth` to 0 for all valid rays [S2, unofficial].
    *   Initialize empty lists `sdf_list`, `points_list`, `marching_zdepth_list` [S2, unofficial].
    *   Compute `maxbound_zdepth` based on the AABB and camera position [S2, S3, unofficial].

2.  **Step Size Calculation:**
    The step size $\Delta t$ is calculated differently depending on the context (SDF vs. Grid):
    
    *   **SDF-based (Conservative):**
        $$ \Delta t = \text{clamp}(f(\mathbf{p}_i), -\text{clamp\_dist}, \text{clamp\_dist}) \times \text{ray\_marching\_ratio} $$
        Where `clamp_dist` prevents extreme steps from noisy or distorted SDFs [S2, unofficial].
        
    *   **Grid-based (Volumetric):**
        $$ \text{unit\_size} = \frac{\text{AABB\_dimension}}{N_{grid}} $$
        $$ \Delta t = \text{mean}(\text{unit\_size}) \times \text{step\_ratio} $$
        [S3, unofficial]

3.  **Ray Marching Loop:**
    For each iteration $i$ from 0 to `march_step`:
    a.  **Sample Points:**
        $$ \mathbf{p}_i = \mathbf{o} + (\text{init\_zdepth} + \text{marching\_zdepth}) \mathbf{d} $$
        [S2, unofficial]
    b.  **Evaluate SDF:**
        $$ f_i = f(\mathbf{p}_i) $$
        If using a neural decoder, detach gradients if not training [S2, unofficial].
    c.  **Clamp SDF:**
        $$ f_{clamped} = \max(-\text{clamp\_dist}, \min(f_i, \text{clamp\_dist})) $$
        [S2, unofficial]
    d.  **Update Depth:**
        $$ \text{marching\_zdepth}_{new} = \text{marching\_zdepth} + f_{clamped} \times \text{ray\_marching\_ratio} $$
        [S2, unofficial]
    e.  **Store Data:**
        Append $f_{clamped}$ to `sdf_list`, $\mathbf{p}_i$ to `points_list`, and new depth to `marching_zdepth_list` [S2, unofficial].

4.  **Termination and Validity Checks:**
    After the loop (or during recursive steps), compute validity masks:
    *   **Max Bound Check:**
        $$ \text{valid\_mask\_max} = (\text{final\_depth} + \text{init\_zdepth}) < \text{maxbound\_zdepth} $$
        [S2, unofficial]
    *   **Surface Hit Check:**
        $$ \text{valid\_mask\_hit} = \min_i(|f_i|) \le \text{threshold} $$
        [S2, unofficial]
    *   **First Query Check (Corner Case):**
        $$ \text{valid\_mask\_first} = f_0 > \text{threshold} $$
        This prevents false negatives if the ray starts inside the surface [S2, unofficial].
    *   **Final Valid Mask:**
        $$ \text{valid\_mask\_render} = \text{valid\_mask\_max} \land \text{valid\_mask\_hit} \land \text{valid\_mask\_first} $$
        [S2, unofficial]

5.  **Recursive Optimization (Optional):**
    In `ray_marching_recursive`, only rays where `valid_mask_render` is true are processed in the next iteration, skipping rays that have already hit the surface or escaped [S2, unofficial].

### Parameters
*   `clamp_dist`: Float. Threshold for clamping SDF values to prevent instability. Typical value: 0.1 [S2, unofficial].
*   `ray_marching_ratio`: Float. Multiplier applied to the clamped SDF to control step aggressiveness. Typical value: 1.5 (from source args) [S2, unofficial].
*   `threshold`: Float. SDF value threshold for surface hit detection. Typical value: $5 \times 10^{-5}$ [S2, unofficial].
*   `march_step`: Integer. Maximum number of iterations. Typical value: 100 [S2, unofficial].
*   `step_ratio`: Float. Multiplier for grid-based step size [S3, unofficial].

### Edge Cases and Pitfalls
*   **Distorted SDFs:** Domain repetition or boolean operations can distort the distance field, causing the estimated distance to be larger than the actual distance. This leads to the ray skipping over geometry. Mitigation: Step shorter than the SDF returns (use `ray_marching_ratio < 1.0` or clamp aggressively) [S1, unofficial].
*   **Immediate Surface Hit:** If the first SDF query $f_0$ is already below the threshold, standard logic might fail to register a hit if not handled. The `valid_mask_first_query` check ensures rays starting inside or on the surface are handled correctly [S2, unofficial].
*   **Boundary Escape:** Rays must not exceed `maxbound_zdepth`. Failure to check this leads to sampling outside the valid volume or infinite loops in unbounded scenes [S2, unofficial].
*   **Precision:** Clamping SDF values prevents numerical instability from very large or small distance values, especially in neural implicit representations [S2, unofficial].

## Subtopics

### [Distance Accuracy vs Performance Trade-off](1546-distance-accuracy-vs-performance-trade-off.md)
Analyzes the inverse relationship between SDF evaluation accuracy and computational cost. Underestimating distance forces smaller steps, increasing iterations. An exact but slower SDF may outperform an inaccurate but fast one if the inaccuracy causes disproportionate step reduction.

### [Adaptive Lipschitz Constant (ALC)](1883-adaptive-lipschitz-constant-alc.md)
Uses a nested BVH to compute tighter, adaptive Lipschitz constants for step size calculation, solving the problem of overly conservative steps in large scenes with high global Lipschitz constants.

### [Gradient-based Sphere Tracing with Lipschitz Constraints](1936-gradient-based-sphere-tracing-with-lipschitz-constraints.md)
Uses local SDF gradients to compute larger, safer step sizes, particularly near silhouettes. Combines first and second Lipschitz constraints to account for curvature.

### [Lipschitz Step Size Ray Marching](3729-lipschitz-step-size-ray-marching.md)
Uses a pre-computed global Lipschitz constant $\lambda$ to guarantee no surface penetration. Simple and stable but potentially less efficient for complex scenes with varying feature scales.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of rays and $M$ is the number of steps per ray. Recursive variants reduce effective $M$ for rays that hit early [S2, unofficial].
*   **Memory Complexity:** $O(N \cdot M)$ for storing history lists (`sdf_list`, `points_list`) in parallel implementations. Recursive implementations use $O(N)$ per active ray [S2, unofficial].
*   **Performance:** Parallel implementation (`ray_marching_trivial`) processes all rays simultaneously, leveraging GPU tensor cores. Recursive implementation (`ray_marching_recursive`) optimizes for scenes with early occlusion by skipping finished rays [S2, unofficial].
*   **Strengths:** Robust against artifacts from distorted SDFs; handles both implicit surfaces and volumetric grids; supports differentiable rendering via gradient detachment options [S2, unofficial].
*   **Limitations:** Requires careful tuning of `clamp_dist` and `ray_marching_ratio`; grid-based step sizing may be inefficient for sparse volumes; global Lipschitz bounds (in related methods) can be overly conservative [S1, S3, unofficial].
*   **When to Use:** When rendering distorted or neural implicit SDFs where standard sphere tracing fails due to overshooting; when high precision is required near complex boundaries [S1, S2, unofficial].
*   **When Not to Use:** When performance is critical and the scene has simple, well-behaved geometry with known Lipschitz constants; when memory is severely constrained and storing full history lists is prohibitive [S2, unofficial].

## Background
Safe Raymarching Step Size originates from the practical challenges of rendering distorted Signed Distance Fields, such as those created by domain repetition or boolean operations [S1, unofficial]. The blog post [S1, unofficial] highlights that naive sphere tracing fails when the distance field is distorted, leading to artifacts where the ray skips geometry. The solution proposed is to step shorter than the distance field returns.

The tensor-based implementation ([S2, unofficial]) emerged from differentiable rendering frameworks (e.g., DeepSDF, NeRF variants) where stable gradient flow and correct surface hit detection are crucial for optimization. The `DIST-Renderer` [S2, unofficial] formalized the clamping and validity mask logic to handle neural implicit representations.

The grid-based approach ([S3, unofficial]) is common in volumetric rendering and collaborative NeRF systems, where step size is derived from voxel dimensions to ensure consistent sampling density across the scene [S3, unofficial].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S3] Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
