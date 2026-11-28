# Median Depth Estimation via Binary Search

Also called: GGGS Depth Rendering, Geometry-Grounded Gaussian Splatting Depth Rendering

## Overview
A method for computing the median depth of a ray and its analytical gradients for geometric regularization in Gaussian Splatting by performing a binary search for the depth where transmittance equals 0.5 [S1, unofficial]. It differs from [Expected Depth Rendering](8816-expected-depth-rendering.md) by targeting the median (50% transmittance) rather than the mean, providing a robust geometric prior that is less sensitive to outliers in depth distribution.

## How it works
**Inputs and Outputs:**
*   **Input:** Initial median depth estimate $t_{init}$ (from RaDe-GS [S1, unofficial]), Gaussian parameters $\theta$ (centers $\mathbf{x}_c$, opacities $o$, covariances $\Sigma$), camera extrinsics $\mathbf{W}$, projection Jacobian $\mathbf{J}$, and pixel location $\mathbf{u}$.
*   **Output:** Median depth $t_{med}$ and gradients $\frac{\partial L}{\partial \theta_i}$ for each Gaussian $i$ along the ray.

**Data Structures:**
*   **Gaussian List:** A list of Gaussians intersecting the pixel's screen-space support, sorted by depth. Each Gaussian $i$ has parameters $\theta_i = \{\mathbf{x}_c, o, \Sigma\}$.
*   **Search Interval:** A pair $[t_{min}, t_{max}]$ defining the current depth range.

**Preliminaries:**
The scene is represented as 3D Gaussians [S1, unofficial]:
$$G(\mathbf{x}) = o \exp \left( -(\mathbf{x} - \mathbf{x}_c)^\top \Sigma^{-1} (\mathbf{x} - \mathbf{x}_c) \right)$$
The covariance is transformed to the camera frame:
$$\Sigma_{cam} = \mathbf{W} \Sigma \mathbf{W}^\top$$
And to screen space:
$$\Sigma'_{2D} = \mathbf{J} \Sigma_{cam} \mathbf{J}^\top = \mathbf{J} \mathbf{W} \Sigma \mathbf{W}^\top \mathbf{J}^\top$$
The per-pixel alpha for a Gaussian at pixel $\mathbf{u}$ is:
$$\alpha(\mathbf{u}) = o e^{-(\mathbf{u}-\mathbf{u}_c)^\top \Sigma_{2D}^{-1}(\mathbf{u}-\mathbf{u}_c)}$$
Transmittance $T(t)$ is the product of $(1-\alpha_i)$ for all Gaussians $i$ up to depth $t$.

**Parameters:**
*   $r$: Half-width of the initial search interval. Typical value: $0.4$ during training [S1, unofficial].
*   $num\_iterations$: Number of binary search iterations. Value: $5$ [S1, unofficial].
*   $num\_segments$: Number of segments the interval is divided into. Value: $8$ (creating 7 internal segment points) [S1, unofficial].

**Algorithm Steps:**

1.  **Initialize Interval:** Set $t_{min} = t_{init} - r$ and $t_{max} = t_{init} + r$ [S1, unofficial].
2.  **Boundary Check:** Compute transmittance $T(t_{min})$ and $T(t_{max})$. If both are $> 0.5$ or both are $< 0.5$, mask the pixel for geometric regularization (no gradient update) [S1, unofficial].
3.  **Iterative Search:** Repeat for $k = 1$ to $num\_iterations$:
    a. Divide $[t_{min}, t_{max}]$ into $num\_segments$ equal parts. Calculate 7 internal points $t_j = t_{min} + j \cdot \frac{t_{max}-t_{min}}{num\_segments}$ for $j=1..7$.
    b. Traverse Gaussians along the ray to compute $T(t_j)$ for each $t_j$.
    c. Find the segment $[t_j, t_{j+1}]$ where $T(t_j)$ and $T(t_{j+1})$ bracket 0.5 (i.e., one is $>0.5$ and the other $<0.5$).
    d. Update $t_{min} = t_j$ and $t_{max} = t_{j+1}$ [S1, unofficial].
4.  **Final Depth:** Set $t_{med} = \frac{t_{min} + t_{max}}{2}$ [S1, unofficial]. The final depth error is bounded by $0.8 \times 8^{-5} \approx 2.441 \times 10^{-5}$ [S1, unofficial].

**Backward Pass:**
The gradient of the loss $L$ with respect to Gaussian parameters $\theta_i$ is computed as [S1, unofficial]:
$$\frac{\partial L}{\partial t_{med}} \cdot \frac{\partial t_{med}}{\partial \theta_i} = \frac{\partial L}{\partial t_{med}} \cdot \frac{\partial T(t_{med}; \theta)}{\partial \theta_i} / \left( -\frac{\partial T(t; \theta)}{\partial t} \right)|_{t=t_{med}}$$
This requires two traversals of the Gaussians at $t_{med}$:
1.  Compute $\frac{\partial T}{\partial t}|_{t_{med}}$:
    $$\frac{\partial T(t; \theta)}{\partial t}|_{t=t_{med}} = \sum_i \frac{0.5}{T_i(t_{med}; \theta_i)} \frac{\partial T_i(t; \theta_i)}{\partial t}|_{t=t_{med}}$$
2.  Compute $\frac{\partial T}{\partial \theta_i}|_{t_{med}}$ for each Gaussian:
    $$\frac{\partial T(t; \theta)}{\partial \theta_i} = \frac{0.5}{T_i(t_{med}; \theta_i)} \frac{\partial T_i(t_{med}; \theta_i)}{\partial \theta_i}$$
The terms $\frac{\partial T_i}{\partial t}$ and $\frac{\partial T_i}{\partial \theta_i}$ are derived from the closed-form formulation of $T_i$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def median_depth_rendering(gaussians, t_init, r, num_iterations=5, num_segments=8):
    """
    Compute median depth and gradients for geometric regularization.
    gaussians: List of Gaussian objects with attributes:
               - x_c: center (3,)
               - o: opacity (scalar)
               - Sigma: covariance (3x3)
               - W: world-to-camera rotation (3x3)
               - J: projection Jacobian (2x3)
               - u_c: projected center (2,)
               - alpha(u): method to compute per-pixel alpha
    t_init: Initial median depth estimate
    r: Half-width of initial interval
    """
    t_min = t_init - r
    t_max = t_init + r
    
    # Boundary check
    T_min = compute_transmittance(gaussians, t_min)
    T_max = compute_transmittance(gaussians, t_max)
    
    if (T_min > 0.5 and T_max > 0.5) or (T_min < 0.5 and T_max < 0.5):
        return None, None  # Mask pixel
    
    # Binary search
    for _ in range(num_iterations):
        step_size = (t_max - t_min) / num_segments
        segment_points = [t_min + j * step_size for j in range(1, num_segments)]
        transmittances = [compute_transmittance(gaussians, t) for t in segment_points]
        
        # Find bracketing segment
        found = False
        for j in range(num_segments - 1):
            t_start = segment_points[j] if j > 0 else t_min
            t_end = segment_points[j+1] if j+1 < num_segments else t_max
            
            # Use precomputed or recompute boundary transmittances if needed
            # For simplicity, assume we have T at segment_points and boundaries
            T_start = T_min if j == 0 else transmittances[j-1]
            T_end = transmittances[j]
            
            if (T_start > 0.5 and T_end < 0.5) or (T_start < 0.5 and T_end > 0.5):
                t_min = t_start
                t_max = t_end
                found = True
                break
        
        if not found:
            # Fallback or error handling if no segment found (should not happen if boundaries bracket)
            break
            
    t_med = (t_min + t_max) / 2
    
    # Backward pass
    # Compute dT/dt at t_med
    dT_dt = 0.0
    dT_dtheta = {}
    
    for i, g in enumerate(gaussians):
        T_i = compute_transmittance_single_gaussian(g, t_med) # T_i is (1-alpha_i)
        if T_i == 0:
            continue
            
        # dT/dt = sum_i [0.5 / T_i] * dT_i/dt
        dT_i_dt = compute_dT_i_dt(g, t_med)
        dT_dt += (0.5 / T_i) * dT_i_dt
        
        # dT/dtheta_i = [0.5 / T_i] * dT_i/dtheta_i
        dT_i_dtheta = compute_dT_i_dtheta(g, t_med)
        dT_dtheta[i] = (0.5 / T_i) * dT_i_dtheta
        
    # Gradient backprop
    # dL/dt_med is provided by the caller
    # dL/dtheta_i = dL/dt_med * dT/dtheta_i / (-dT/dt)
    
    return t_med, dT_dtheta, dT_dt

def compute_transmittance(gaussians, t):
    """Compute total transmittance at depth t."""
    T = 1.0
    for g in gaussians:
        # Check if Gaussian is in front of t
        if is_in_front(g, t):
            alpha = g.alpha(g.u_c) # Simplified: alpha at center, usually evaluated at pixel
            # In practice, alpha is evaluated at the intersection point or projected center
            # For ray marching, we evaluate alpha along the ray
            # This is a simplification; actual implementation requires ray-Gaussian intersection
            T *= (1.0 - alpha)
    return T

def compute_transmittance_single_gaussian(g, t):
    """Compute T_i = 1 - alpha_i at depth t."""
    # Simplified: assume alpha is computed at the projected center
    # In reality, alpha varies along the ray
    alpha = g.alpha(g.u_c)
    return 1.0 - alpha

def compute_dT_i_dt(g, t):
    """Compute derivative of T_i with respect to t."""
    # Derived from closed-form T_i
    # dT_i/dt = -d(alpha_i)/dt
    # This requires chain rule through Gaussian parameters and position
    pass # Implementation depends on specific Gaussian parameterization

def compute_dT_i_dtheta(g, t):
    """Compute derivative of T_i with respect to Gaussian parameters."""
    # Derived from closed-form T_i
    pass # Implementation depends on specific Gaussian parameterization

def is_in_front(g, t):
    """Check if Gaussian is in front of depth t."""
    # Check if Gaussian center is in front and ray intersects
    pass
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot I \cdot S)$ per pixel, where $N$ is the number of Gaussians, $I$ is the number of iterations (5), and $S$ is the number of segments (8). This is significantly faster than full ray marching for high-resolution depth estimation.
*   **Memory Complexity:** $O(N)$ to store Gaussian parameters.
*   **Accuracy:** The final depth error is bounded by $2.441 \times 10^{-5}$ with 5 iterations [S1, unofficial].
*   **Strengths:** Provides a robust geometric prior for Gaussian Splatting optimization. Efficient compared to iterative root-finding methods.
*   **Limitations:** Requires an initial estimate $t_{init}$ (e.g., from RaDe-GS). If the initial interval does not bracket the median, the pixel is masked.

## Background
Originated in "Geometry-Grounded Gaussian Splatting" (GGGS) [S1, unofficial]. It builds on [Median Depth Rendering](8815-median-depth-rendering.md) and uses an initial estimate from RaDe-GS [S1, unofficial].

## Sources
[S1] Geometry-Grounded Gaussian Splatting, https://arxiv.org/abs/2601.17835v2, fetched 2026-10-26 (unofficial)
