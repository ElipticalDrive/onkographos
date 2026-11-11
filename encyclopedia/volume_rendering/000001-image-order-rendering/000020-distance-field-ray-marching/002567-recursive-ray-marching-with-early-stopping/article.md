# Recursive Ray Marching with Early Stopping

Also called: Dynamic Synchronized Inference

## Overview
Recursive Ray Marching with Early Stopping is a parallelization technique for sphere tracing that maintains a dynamic mask of unfinished rays to prevent wasted computation on rays that have already converged or escaped [S1][S2, unofficial]. It differs from naive synchronous batching by selectively processing only active rays at each step, significantly reducing memory and compute overhead for high-resolution rendering [S1].

## How it works
The method extends the standard sphere tracing loop by introducing a boolean mask to track ray status [S1][S2, unofficial]. Instead of evaluating the SDF for all pixels simultaneously regardless of their state, the algorithm filters the batch to only include rays that have not yet terminated [S1][S2, unofficial].

**Inputs and Outputs:**
*   **Input:** Camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), SDF function $f(\mathbf{p})$, maximum marching steps $N_{max}$, convergence threshold $\epsilon$, and step ratio $\alpha$ [S1][S2, unofficial].
*   **Output:** Rendered depth, normal, or color image [S1].

**Data Structures:**
*   `marching_zdepth`: Array of floats, size $H \times W$, storing the accumulated distance $t$ for each ray. Initialized to 0 (or near-plane) [S2, unofficial].
*   `unfinished_mask`: Boolean array, size $H \times W$. Initially all `True` (or valid within bounds) [S1][S2, unofficial].
*   `query_points`: List or tensor of 3D points $\mathbf{p}$ corresponding to active rays, used for SDF evaluation [S1].

**Parameters:**
*   $\epsilon$: Convergence threshold. Typical value: $5 \times 10^{-5}$ [S2, unofficial].
*   $\alpha$: Step ratio (multiplier for SDF value). Typical range: $1.2$ to $1.8$ [S1].
*   $N_{max}$: Maximum marching steps. Typical value: $100$ [S1].
*   $K$: Number of queries to keep for backpropagation (differentiable rendering). $K=1$ for depth, $K=3$ for shape completion [S1].

**Algorithm Steps:**
1.  **Initialization**:
    *   Set `marching_zdepth` to zeros [S2, unofficial].
    *   Set `unfinished_mask` to `True` for all rays within the viewing frustum/bounds [S2, unofficial].
    *   Initialize loop counter $i = 0$ [G].
2.  **Marching Loop** (repeat while $i < N_{max}$ and any ray is unfinished):
    *   **Select Active Rays**: Identify indices where `unfinished_mask` is `True` [S1][S2, unofficial].
    *   **Check Termination**: If no active rays remain, break loop [S1].
    *   **Evaluate SDF**: For each active ray $r$, compute position $\mathbf{p}_r = \mathbf{o}_r + \text{marching\_zdepth}_r \cdot \mathbf{d}_r$. Query SDF: $d_r = f(\mathbf{p}_r)$ [S1][S2, unofficial].
    *   **Update Depth**: Advance ray by safe step: $\text{marching\_zdepth}_r \leftarrow \text{marching\_zdepth}_r + \alpha \cdot d_r$ [S2, unofficial]. Note: In standard sphere tracing $\alpha=1$, but implementations may use $\alpha > 1$ for speed, risking missed thin structures [S1].
    *   **Check Convergence**: For each active ray $r$:
        *   If $|d_r| < \epsilon$ (hit surface) OR `marching_zdepth`$_r$ exceeds maximum bound (escaped), set `unfinished_mask`$_r = \text{False}$ [S1][S2, unofficial].
    *   Increment $i$ [G].
3.  **Backpropagation Optimization (Differentiable Case)**:
    *   To avoid memory explosion from storing all query points for gradients, save query locations globally [S1].
    *   Perform minimization over a detached tensor graph to find the minimum absolute SDF value for each ray [S1].
    *   Retrieve the minimum $K$ queries for each ray [S1].
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
        camera_origins: Array of shape (H, W, 3)
        camera_directions: Array of shape (H, W, 3), normalized
        sdf_function: Callable f(p) -> float
        max_steps: Integer, maximum number of marching steps
        threshold: Float, convergence threshold epsilon
        step_ratio: Float, multiplier alpha for step size
        
    Returns:
        depth_map: Array of shape (H, W)
        mask: Array of shape (H, W), boolean
    """
    # Determine dimensions safely for lists or arrays
    H = len(camera_origins)
    W = len(camera_origins[0])
    
    # Initialize depth accumulation
    marching_zdepth = [[0.0] * W for _ in range(H)]
    
    # Initialize mask: True means ray is still active/unfinished
    unfinished_mask = [[True] * W for _ in range(H)]
    
    for step in range(max_steps):
        # Collect active ray coordinates
        active_coords = [(r, c) for r in range(H) for c in range(W) if unfinished_mask[r][c]]
        
        if not active_coords:
            break
            
        # Process active rays
        for r, c in active_coords:
            # Compute current 3D position: p = o + t * d
            ox, oy, oz = camera_origins[r][c]
            dx, dy, dz = camera_directions[r][c]
            t = marching_zdepth[r][c]
            
            px = ox + t * dx
            py = oy + t * dy
            pz = oz + t * dz
            
            # Evaluate SDF
            d = sdf_function((px, py, pz))
            
            # Update depth: t = t + alpha * d
            marching_zdepth[r][c] += step_ratio * d
            
            # Check convergence criteria
            if abs(d) < threshold:
                unfinished_mask[r][c] = False
                
    # Note: Backpropagation optimization (detached graph minimization, K-query retrieval)
    # is performed separately during the differentiable training pass [S1].
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
*   **Time Complexity**: $O(N_{active} \cdot N_{steps})$, where $N_{active}$ is the number of unfinished rays [S1]. As rays converge, the dynamic mask sets their flags to zero, reducing the active batch size and leading to significant speedups over synchronous batching for scenes with varying depth complexity [S1].
*   **Memory Complexity**: $O(H \cdot W)$ for masks and depth buffers [S2, unofficial]. The backpropagation optimization reduces gradient storage from $O(N_{steps} \cdot H \cdot W)$ to $O(K \cdot H \cdot W)$ [S1].
*   **Performance**: Enables high-resolution rendering (2048x2048) on limited hardware (e.g., GTX-1080Ti) by avoiding wasted computation on converged rays [S1].
*   **Strengths**: Efficient for sparse or complex scenes where rays terminate at different depths. Differentiable [S1].
*   **Limitations**: Requires dynamic masking logic, which can be complex to implement efficiently on GPUs compared to static batching. Step ratio $\alpha$ tuning is critical [S1].

## Background
The method was introduced as "Dynamic Synchronized Inference" in the context of differentiable sphere tracing for rendering Deep Implicit Signed Distance Functions (DeepSDF) [S1]. It addresses the inefficiency of naive parallelization where all rays are updated synchronously, leading to wasted cycles on rays that have already converged [S1]. The implementation details and code examples are available in the unofficial DIST-Renderer repository [S2, unofficial].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
