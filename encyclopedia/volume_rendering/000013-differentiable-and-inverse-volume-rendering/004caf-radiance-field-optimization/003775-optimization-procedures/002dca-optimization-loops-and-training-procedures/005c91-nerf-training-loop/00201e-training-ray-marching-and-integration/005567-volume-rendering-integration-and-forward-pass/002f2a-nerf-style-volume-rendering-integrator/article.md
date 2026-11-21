# NeRF-style Volume Rendering Integrator

Also called: NeRF forward raymarching, NeRFIntegrator

## Overview
This method computes the expected color $\hat{C}(r)$ for a ray $r$ by discretizing the medium interaction interval into fixed steps and accumulating emission weighted by transmittance [S1, unofficial]. It differs from its sibling [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md) by using a dense grid-based density representation with no direction-dependent emission, and by employing a fixed-step marching loop with optional jittering rather than adaptive sampling or neural network queries [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray $r$ (origin, direction), scene geometry (for exit intersection), density grid (scalar field $\sigma$), emission field (vector field $c$), parameters: `queries_per_ray`, `density_noise_std`, `jittering_enabled`, `activation`, `hide_emitters`.
*   **Output:** Rendered color $\hat{C}(r)$, accumulated weights, and optionally depth.

**Data Structures:**
*   **Density Grid:** Dense 3D array of scalar values $\sigma$. Layout is not specified in sources [S1, unofficial].
*   **Emission Field:** Dense 3D array of RGB values $c$. Layout is not specified in sources [S1, unofficial].
*   **State Variables:** `result` (color accumulator), `throughput` (transmittance $T$), `weights_sum` (accumulated opacity weight), `step_j` (current step index), `t_a` (previous sample position), `t_b` (current sample position).

**Parameters:**
*   `queries_per_ray`: Number of steps for ray marching. Default: 128 [S1, unofficial].
*   `density_noise_std`: Standard deviation for noise added to density. Default: 0 [S1, unofficial].
*   `jittering_enabled`: Boolean flag enabling jittering for step positions. Default: True [S1, unofficial].
*   `activation`: Activation function type for density ('identity', 'relu'). Default: 'identity' [S1, unofficial].
*   `hide_emitters`: Boolean flag to hide background emitters if weights_sum is zero. Default: Not specified [S1, unofficial].

**Steps:**
1.  **Initialization:** Initialize `result` = 0, `throughput` = 1, `weights_sum` = 0, `step_j` = 0, `t_a` = 0. Determine primal mode (forward vs. adjoint).
2.  **Exit Intersection:** Intersect ray with scene geometry to find medium exit distance $t_{exit}$. Update active mask to include only rays that hit the exit.
3.  **Step Size Calculation:** Calculate `step_size` using `step_size_for_marching(queries_per_ray, 0, t_{exit}, jittering_enabled)`. Sample `jitter_sample` from sampler.
4.  **Marching Loop:** While `still_walking` (mask: `step_j < queries_per_ray`):
    a.  Calculate `t_b` using `query_t_for_step(step_j + 1, step_size, jitter_sample, jittering_enabled)`.
    b.  Calculate `actual_step` = `t_b` - `t_a`.
    c.  Calculate `alpha_recip`:
        *   If `step_j + 1 < queries_per_ray`: `alpha_recip` = $\exp(-\sigma \cdot \text{actual\_step})$.
        *   Else: `alpha_recip` = 1.0.
    d.  **Numerical Stability:** Add $\epsilon = 10^{-10}$ to `alpha_recip` to prevent NaNs [S1, unofficial].
    e.  Calculate `weight` = $(1.0 - \text{alpha\_recip}) \cdot \text{throughput}$.
    f.  **Accumulate Color:**
        *   If primal: `result` += `weight` * `emission`.
        *   If adjoint: `result` -= `weight` * `emission`.
    g.  Increment `step_j`. Update `t_a` = `t_b`.
    h.  **Update Throughput:** `throughput` *= `alpha_recip`.
    i.  **Update Weights Sum:** `weights_sum` += `weight`.
5.  **Background Contribution:** Calculate `contribution` = $(1 - \text{weights\_sum}) \cdot c_{bg}$. Add to `result` for active pixels. If `hide_emitters` is true and `weights_sum` is zero, do not add background emission [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_integrator(ray, scene, density_grid, emission_grid, params):
    """
    NeRF-style volume rendering integrator.
    
    Args:
        ray: Ray object with origin and direction.
        scene: Scene object for intersection.
        density_grid: 3D array of density values.
        emission_grid: 3D array of emission values.
        params: Dictionary of parameters.
        
    Returns:
        result: Rendered color.
    """
    # Parameters
    queries_per_ray = params.get('queries_per_ray', 128)
    density_noise_std = params.get('density_noise_std', 0)
    jittering_enabled = params.get('jittering_enabled', True)
    activation = params.get('activation', 'identity')
    hide_emitters = params.get('hide_emitters', False)
    
    # Initialization
    result = 0.0
    throughput = 1.0
    weights_sum = 0.0
    step_j = 0
    t_a = 0.0
    
    # Exit Intersection
    t_exit = scene.intersect(ray)
    if t_exit is None:
        return 0.0  # Ray misses scene
        
    # Step Size Calculation
    step_size = step_size_for_marching(queries_per_ray, 0, t_exit, jittering_enabled)
    jitter_sample = sampler.sample()
    
    # Marching Loop
    while step_j < queries_per_ray:
        # Calculate current sample position
        t_b = query_t_for_step(step_j + 1, step_size, jitter_sample, jittering_enabled)
        actual_step = t_b - t_a
        
        # Query density and emission
        sigma = query_density(density_grid, ray, t_b)
        c = query_emission(emission_grid, ray, t_b)
        
        # Apply activation if needed
        if activation == 'relu':
            sigma = max(0, sigma)
        elif activation == 'identity':
            pass
        else:
            raise ValueError(f"Unknown activation: {activation}")
            
        # Add density noise if enabled
        if density_noise_std > 0:
            sigma += random_noise(std=density_noise_std)
            
        # Calculate alpha_recip
        if step_j + 1 < queries_per_ray:
            alpha_recip = math.exp(-sigma * actual_step)
        else:
            alpha_recip = 1.0
            
        # Numerical stability
        alpha_recip += 1e-10
        
        # Calculate weight
        weight = (1.0 - alpha_recip) * throughput
        
        # Accumulate color
        result += weight * c
        
        # Update throughput and weights_sum
        throughput *= alpha_recip
        weights_sum += weight
        
        # Update step variables
        step_j += 1
        t_a = t_b
        
    # Background contribution
    c_bg = (1.0, 1.0, 1.0)  # Default white background
    if not hide_emitters or weights_sum > 0:
        result += (1.0 - weights_sum) * c_bg
        
    return result
```

**Edge Cases and Pitfalls:**
*   **Precision:** Adding $\epsilon = 10^{-10}$ to `alpha_recip` is critical to prevent NaNs during gradient computation [S1, unofficial].
*   **Boundaries:** Rays missing the scene return 0. Rays hitting the exit are processed up to `queries_per_ray` steps.
*   **Degenerate Input:** If `queries_per_ray` is 0, the loop does not execute, and only background is returned. If `density_noise_std` is large, noise may dominate the signal.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per ray, where $N$ is `queries_per_ray`. Total time is $O(N \cdot R)$, where $R$ is the number of rays.
*   **Memory Complexity:** $O(V)$ for the density and emission grids, where $V$ is the volume of the grid.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Linear with respect to `queries_per_ray` and number of rays.
*   **Strengths:** Simple implementation, deterministic (with fixed jitter), easy to differentiate.
*   **Limitations:** Fixed step size may lead to aliasing or inefficiency in sparse regions. No direction-dependent emission.
*   **When to Use:** When a dense grid representation is sufficient and direction-dependent effects are not required.
*   **When Not to Use:** When adaptive sampling is needed for efficiency or accuracy, or when direction-dependent emission is required.

## Background
*   **Origin:** The method is based on the NeRF (Neural Radiance Fields) framework [S1, unofficial].
*   **History:** Developed as part of the unbiased inverse volume rendering project [S1, unofficial].
*   **Usage:** Used in differentiable rendering for inverse problems, such as scene reconstruction and optimization [S1, unofficial].

## Sources
[S1] Unbiased Inverse Volume Rendering, https://github.com/rgl-epfl/unbiased-inverse-volume-rendering, fetched 2026-10-26 (unofficial)
