# Recursive Ray Marching with Early Stopping

Also called: Dynamic Synchronized Inference

## Overview
Recursive Ray Marching with Early Stopping is a parallelization technique for sphere tracing that maintains a dynamic mask of unfinished rays to prevent wasted computation on rays that have already converged or escaped [S1][S2, unofficial]. It differs from naive synchronous batching by selectively processing only active rays at each step, significantly reducing memory and compute overhead for high-resolution rendering [S1].

## How it works
The method extends the standard sphere tracing loop by introducing a boolean mask to track ray status. Instead of evaluating the SDF for all pixels simultaneously regardless of their state, the algorithm filters the batch to only include rays that have not yet terminated [S1][S2, unofficial].

**Inputs and Outputs:**
*   **Input:** Camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), SDF function $f(\mathbf{p})$, maximum marching steps $N_{max}$, convergence threshold $\epsilon$, and step ratio $\alpha$.
*   **Output:** Rendered depth, normal, or color image.

**Data Structures:**
*   `marching_zdepth`: Array of floats, size $H \times W$, storing the accumulated distance $t$ for each ray. Initialized to 0 (or near-plane) [S2, unofficial].
*   `unfinished_mask`: Boolean array, size $H \times W$. Initially all `True` (or valid within bounds) [S1][S2, unofficial].
*   `query_points`: List or tensor of 3D points $\mathbf{p}$ corresponding to active rays, used for SDF evaluation.

**Parameters:**
*   $\epsilon$: Convergence threshold. Typical value: $5 \times 10^{-5}$ [S2, unofficial].
*   $\alpha$: Step ratio (multiplier for SDF value). Typical range: $1.2$ to $1.8$ [S1].
*   $N_{max}$: Maximum marching steps. Typical value: $100$ [S1].
*   $K$: Number of queries to keep for backpropagation (differentiable rendering). $K=1$ for depth, $K=3$ for shape completion [S1].

**Algorithm Steps:**
1.  **Initialization**:
    *   Set `marching_zdepth` to zeros.
    *   Set `unfinished_mask` to `True` for all rays within the viewing frustum/bounds [S2, unofficial].
    *   Initialize loop counter $i = 0$.

2.  **Marching Loop** (repeat while $i < N_{max}$ and any ray is unfinished):
    *   **Select Active Rays**: Identify indices where `unfinished_mask` is `True`. Let this set be $A$ [S1][S2, unofficial].
    *   **Check Termination**: If $A$ is empty, break loop [S1].
    *   **Evaluate SDF**: For each ray $r \in A$, compute position $\mathbf{p}_r = \mathbf{o}_r + \text{marching\_zdepth}_r \cdot \mathbf{d}_r$. Query SDF: $d_r = f(\mathbf{p}_r)$ [S1][S2, unofficial].
    *   **Update Depth**: Advance ray by safe step: $\text{marching\_zdepth}_r \leftarrow \text{marching\_zdepth}_r + \alpha \cdot d_r$ [S2, unofficial]. Note: In standard sphere tracing $\alpha=1$, but implementations may use $\alpha > 1$ for speed, risking missed thin structures [S1].
    *   **Check Convergence**: For each ray $r \in A$:
        *   If $|d_r| < \epsilon$ (hit surface) OR `marching_zdepth`$_r$ exceeds maximum bound (escaped), set `unfinished_mask`$_r = \text{False}$ [S1][S2, unofficial].
    *   Increment $i$.

3.  **Backpropagation Optimization (Differentiable Case)**:
    *   To avoid memory explosion from storing all query points for gradients, save query locations globally [S1].
    *   Perform minimization over a detached tensor graph to find the minimum absolute SDF value for each ray [S1].
    *   Retrieve the minimum $K$ queries for each ray.
    *   Re-evaluate (feedforward) only those specific $K$ queries with gradients attached [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def recursive_ray_marching_recursive(
    camera_origins, 
    camera_directions, 
    sdf_function, 
    max_steps=100, 
    threshold=5e-5, 
    step_ratio=1.5
):
    """
    Recursive ray marching with early stopping using a dynamic mask.
    
    Args:
        camera_origins: Tensor of shape (H, W, 3)
        camera_directions: Tensor of shape (H, W, 3), normalized
        sdf_function: Callable f(p) -> float
        max_steps: Integer, maximum number of marching steps
        threshold: Float, convergence threshold epsilon
        step_ratio: Float, multiplier alpha for step size
        
    Returns:
        depth_map: Tensor of shape (H, W)
        mask: Tensor of shape (H, W), boolean
    """
    H, W, _ = camera_origins.shape
    # Initialize depth accumulation
    marching_zdepth = torch.zeros((H, W), device=camera_origins.device)
    
    # Initialize mask: True means ray is still active/unfinished
    # Assuming all rays are initially valid within bounds
    unfinished_mask = torch.ones((H, W), dtype=torch.bool, device=camera_origins.device)
    
    for step in range(max_steps):
        # Select active rays
        active_indices = torch.nonzero(unfinished_mask, as_tuple=False)
        
        if active_indices.numel() == 0:
            break
            
        # Extract origins and directions for active rays
        # Reshape active_indices to (N, 2)
        ys, xs = active_indices[:, 0], active_indices[:, 1]
        
        active_origins = camera_origins[ys, xs]
        active_directions = camera_directions[ys, xs]
        
        # Compute current 3D positions
        # p = o + t * d
        active_positions = active_origins + marching_zdepth[ys, xs].unsqueeze(1) * active_directions
        
        # Evaluate SDF for active rays
        # Note: In practice, sdf_function should handle batched inputs
        active_sdf_values = sdf_function(active_positions)
        
        # Update depth: t = t + alpha * d
        # Clamp negative distances if necessary depending on SDF definition
        # Standard sphere tracing uses d directly. 
        # If d < 0, we are inside, step size is still |d| or d depending on convention.
        # Assuming standard sphere tracing where d is distance to surface.
        step_sizes = step_ratio * active_sdf_values
        
        # Update marching_zdepth
        marching_zdepth[ys, xs] = marching_zdepth[ys, xs] + step_sizes
        
        # Check convergence criteria
        # Converged if |sdf| < threshold
        converged = torch.abs(active_sdf_values) < threshold
        
        # Update mask: set to False if converged
        unfinished_mask[ys, xs] = ~converged
        
    return marching_zdepth, unfinished_mask
```

**Edge Cases and Pitfalls:**
*   **Precision**: If $\alpha$ is too large, rays may overshoot thin geometry, causing holes [S1]. If $\alpha$ is too small, convergence is slow [S1].
*   **Memory**: Storing all query points for backpropagation is memory-intensive. The detached graph minimization trick is required for high resolutions (e.g., 2048x2048) [S1].
*   **Boundaries**: Rays escaping the scene (e.g., outside unit sphere) must be masked out to prevent infinite loops or invalid SDF queries [S1].

## Subtopics

### [Adaptive Ray Marching with Dynamic Step Count](3888-adaptive-ray-marching-with-dynamic-step-count.md)
Adjusts step count based on active ray density rather than fixed iteration limits. Use when scene complexity varies significantly across the image.

## Operating characteristics
*   **Time Complexity**: $O(N_{active} \cdot N_{steps})$, where $N_{active}$ is the number of unfinished rays. As rays converge, $N_{active}$ decreases, leading to significant speedups over synchronous batching for scenes with varying depth complexity [S1].
*   **Memory Complexity**: $O(H \cdot W)$ for masks and depth buffers. The backpropagation optimization reduces gradient storage from $O(N_{steps} \cdot H \cdot W)$ to $O(K \cdot H \cdot W)$ [S1].
*   **Performance**: Enables high-resolution rendering (2048x2048) on limited hardware (e.g., GTX-1080Ti) by avoiding wasted computation on converged rays [S1].
*   **Strengths**: Efficient for sparse or complex scenes where rays terminate at different depths. Differentiable.
*   **Limitations**: Requires dynamic masking logic, which can be complex to implement efficiently on GPUs compared to static batching. Step ratio $\alpha$ tuning is critical [S1].

## Background
The method was introduced as "Dynamic Synchronized Inference" in the context of differentiable sphere tracing for rendering Deep Implicit Signed Distance Functions (DeepSDF) [S1]. It addresses the inefficiency of naive parallelization where all rays are updated synchronously, leading to wasted cycles on rays that have already converged [S1]. The implementation details and code examples are available in the unofficial DIST-Renderer repository [S2, unofficial].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
