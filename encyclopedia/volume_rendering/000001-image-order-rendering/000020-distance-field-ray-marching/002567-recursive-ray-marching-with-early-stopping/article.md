# Recursive Ray Marching with Early Stopping

Also called: Dynamic Synchronized Inference

## Overview
Recursive Ray Marching with Early Stopping is a parallelization technique for sphere tracing that maintains a dynamic mask of unfinished rays to prevent wasted computation on rays that have already converged or escaped [S1][S2, unofficial]. It solves the problem of synchronous batch updates where rays with varying convergence speeds stall the entire batch, by selectively processing only active rays in each iteration [S1]. It differs from its siblings, such as [Woodcock Tracking with Macrocell Majorant](986-woodcock-tracking-with-macrocell-majorant.md) which handles participating media via statistical sampling, and [Ray-marching on 3D Gaussians](1036-ray-marching-on-3d-gaussians.md) which uses explicit primitives, by operating on implicit Signed Distance Functions (SDFs) and optimizing the *control flow* of the marching loop rather than the sampling density or geometry representation [S1][S2, unofficial].

## How it works
The method extends the standard sphere tracing loop [Parent](21-distance-field-ray-marching.md) by introducing a boolean mask to track ray status and a batching strategy to maintain computational efficiency in deep learning frameworks.

**Inputs and Outputs:**
- **Input:** Camera rays (origin $\mathbf{o}$, direction $\mathbf{d}$), SDF function $f(\mathbf{p})$, maximum marching steps $N_{max}$ (typically 100 [S1]), convergence threshold $\epsilon$ (e.g., $5 \times 10^{-5}$ [S2, unofficial]), and step ratio $\alpha$ (typically 1.2–1.8 [S1]).
- **Output:** Rendered depth, normal, or color for each pixel.

**Data Structures:**
- `unfinished_mask`: A boolean tensor of shape $(H, W)$, where $H, W$ are image dimensions. Initially all `True`.
- `marching_zdepth`: A float tensor of shape $(H, W)$, initialized to 0. Stores the accumulated distance $t$ along the ray.
- `query_points`: A list or tensor storing the 3D coordinates $\mathbf{p}_i$ queried at each step for backpropagation.

**Numbered Steps:**
1.  **Initialization:** Set `marching_zdepth` to zeros. Set `unfinished_mask` to `valid_mask_max_marching_zdepth` (rays within the bounding volume, e.g., unit sphere [S1]).
2.  **Iteration Loop:** For $i$ from 0 to $N_{max}-1$:
    a.  **Selection:** Identify active rays where `unfinished_mask` is `True`.
    b.  **Batching:** Concatenate the origins and directions of only the active rays into a single batch tensor.
    c.  **SDF Query:** Evaluate $f(\mathbf{p})$ for all active rays. The query point is $\mathbf{p} = \mathbf{o} + \text{marching\_zdepth} \cdot \mathbf{d}$.
    d.  **Step Update:** Calculate step size $\Delta t = \alpha \cdot f(\mathbf{p})$. Update `marching_zdepth` for active rays: $\text{marching\_zdepth} \leftarrow \text{marching\_zdepth} + \Delta t$.
    e.  **Convergence Check:** For each active ray, check if $|f(\mathbf{p})| < \epsilon$ or if the ray has escaped the bounding volume.
    f.  **Mask Update:** Set `unfinished_mask` entries to `False` for rays that converged or escaped.
    g.  **Gradient Management (Implementation Detail):** To avoid unaffordable memory consumption from storing the full computational graph for all queries, save query locations globally. Perform minimization over a detached tensor graph to identify the minimum $K$ queries per ray for backpropagation, then re-evaluate those specific queries with gradients attached [S1]. $K=1$ for depth, $K=3$ for shape completion [S1].
3.  **Finalization:** Map results back to original image resolution. Pad inactive rays with zeros or previous values if necessary [S2, unofficial].

**Formulas:**
- Ray position: $\mathbf{p}_i = \mathbf{o} + t_i \mathbf{d}$
- Step update: $t_{i+1} = t_i + \alpha \cdot f(\mathbf{p}_i)$
- Convergence: Stop if $|f(\mathbf{p}_i)| < \epsilon$ [S1][S2, unofficial].

**Parameters:**
- $N_{max}$: Maximum marching steps. Default 100 [S1].
- $\epsilon$: Convergence threshold. Default $5 \times 10^{-5}$ [S2, unofficial].
- $\alpha$: Step ratio. Range 1.2–1.8 [S1].
- $K$: Number of queries kept for backpropagation. 1 for depth, 3 for shape completion [S1].

**Edge Cases and Pitfalls:**
- **Slow Convergence:** Near the surface, $f(\mathbf{p})$ approaches zero, causing small steps. The threshold $\epsilon$ prevents infinite loops but must be chosen to preserve details supported by the 2D resolution [S1].
- **Memory Consumption:** Storing the full graph for all rays and steps is prohibitive. The detached tensor minimization trick is required for high-resolution rendering (e.g., 2048x2048) [S1].
- **Thin Geometry:** Large $\alpha$ can lead to small holes in thin areas where the ray overshoots the surface before the mask can update [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def recursive_ray_marching_recursive(rays_o, rays_d, sdf_func, max_steps=100, threshold=5e-5, alpha=1.5):
    """
    rays_o: Tensor (H, W, 3)
    rays_d: Tensor (H, W, 3)
    sdf_func: Callable returning SDF value for a batch of points
    """
    H, W = rays_o.shape[:2]
    # Initialize state
    zdepth = torch.zeros(H, W)
    unfinished_mask = torch.ones(H, W, dtype=torch.bool)
    
    # Storage for gradient management
    all_query_points = [] 
    
    for step in range(max_steps):
        # Select active rays
        active_indices = torch.nonzero(unfinished_mask, as_tuple=False)
        if active_indices.numel() == 0:
            break
            
        # Extract origins and directions for active rays
        active_o = rays_o[unfinished_mask]
        active_d = rays_d[unfinished_mask]
        
        # Calculate query points
        # p = o + zdepth * d
        # Reshape zdepth for broadcasting
        zdepth_active = zdepth[unfinished_mask].unsqueeze(-1) # (N, 1)
        query_points = active_o + active_d * zdepth_active    # (N, 3)
        
        # Store points for potential gradient re-evaluation
        all_query_points.append(query_points.detach())
        
        # Evaluate SDF
        sdf_values = sdf_func(query_points) # (N,)
        
        # Check convergence
        converged = torch.abs(sdf_values) < threshold
        
        # Update zdepth for non-converged active rays
        # Step size = alpha * sdf_value
        step_size = alpha * sdf_values
        zdepth[unfinished_mask] = zdepth[unfinished_mask] + step_size
        
        # Update mask: set converged rays to False
        # Also handle escape logic if bounding volume check is implemented
        unfinished_mask[unfinished_mask] = ~converged
        
    # Final rendering logic would map zdepth to depth/normal/color
    # For inactive rays, zdepth remains 0 or initial value
    return zdepth
```

## Subtopics

### [Adaptive Ray Marching with Dynamic Step Count](3888-adaptive-ray-marching-with-dynamic-step-count.md)
Optimizes computational load by batching active rays into fixed-size chunks of steps to maximize hardware throughput, rather than processing rays individually or with static step counts.

## Operating characteristics
- **Time Complexity:** $O(N \cdot K)$, where $N$ is the number of active rays and $K$ is the average number of steps per ray. Early stopping reduces the effective $K$ for rays that converge quickly.
- **Memory Complexity:** $O(H \cdot W)$ for the mask and zdepth tensors. The gradient management strategy reduces memory from $O(N \cdot N_{max})$ to $O(N \cdot K)$ by only storing the minimum $K$ queries for backpropagation [S1].
- **Performance:** Enables high-resolution rendering (2048x2048) on single GPUs (e.g., GTX-1080Ti) by avoiding wasted computation on converged rays [S1].
- **Strengths:** Significant speedup over synchronous sphere tracing for scenes with varying depth complexity. Differentiable, allowing integration into optimization pipelines [S1].
- **Limitations:** Requires careful tuning of $\alpha$ to avoid holes in thin geometry [S1]. The detached tensor trick adds implementation complexity [S1].

## Background
The method was introduced as "Dynamic Synchronized Inference" in the context of differentiable sphere tracing for rendering Deep Implicit Signed Distance Functions [S1]. It addresses the inefficiency of naive parallelization where all rays are updated synchronously, leading to wasted cycles on rays that have already converged [S1]. The implementation details and codebase are available in the unofficial DIST-Renderer repository [S2, unofficial].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
