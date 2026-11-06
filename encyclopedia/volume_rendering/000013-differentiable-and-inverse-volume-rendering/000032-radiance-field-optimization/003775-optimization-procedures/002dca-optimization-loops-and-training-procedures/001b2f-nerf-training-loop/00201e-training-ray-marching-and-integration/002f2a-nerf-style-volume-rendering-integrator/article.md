# NeRF-style Volume Rendering Integrator

Also called: NeRF forward raymarching, NeRFIntegrator

## Overview
A simplified NeRF-style integrator that accumulates emission along a ray through a medium, backed by a dense grid instead of a neural network, with no direction-dependent emission [S1, unofficial]. It differs from standard NeRF implementations by using a dense grid for density/emission storage and explicitly handling primal/dual modes for gradient computation in inverse rendering contexts [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $o$ and direction $d$.
- Scene intersection $si$ (providing entry point $t_{entry}$ and exit point $t_{exit}$).
- Grid data structure containing density $\sigma$ and emission $L_e$.
- Parameters: `queries_per_ray` ($N$), `density_noise_std`, `jittering_enabled`, `activation`, `hide_emitters`.

**Outputs:**
- Accumulated radiance $L$.

**Data Structures:**
- **Grid:** Dense 3D array storing $\sigma$ and $L_e$. Access is via trilinear interpolation or nearest neighbor depending on implementation details not fully specified in sources [S1, unofficial].
- **Active Mask:** Boolean array indicating which rays are still valid (have not exited the medium or reached max steps).

**Steps:**

1. **Initialization:**
   - Initialize accumulated radiance $L = 0$.
   - Initialize throughput $\alpha_{through} = 1.0$.
   - Initialize weight sum $W_{sum} = 0$.
   - Initialize step counter $j = 0$.
   - Determine primal mode (for gradient direction).

2. **Ray Intersection:**
   - Intersect ray with scene to find medium exit $t_{exit}$ [S1, unofficial].
   - Update active mask to include only rays that hit the exit [S1, unofficial].

3. **Step Size Calculation:**
   - Calculate `step_size` using `step_size_for_marching(N, 0, si.t, jittering_enabled)` [S1, unofficial].
   - Sample jitter offset $s_{jitter}$ from sampler [S1, unofficial].

4. **Marching Loop:**
   - While `still_walking` (mask is true) AND $j < N$:
     a. **Calculate Next T:**
        - $t_b = \text{query\_t\_for\_step}(j + 1, \text{step\_size}, s_{jitter}, \text{jittering\_enabled})$ [S1, unofficial].
        - $t_a$ is the previous $t_b$ (initially $si.t$).
        - $\Delta t = t_b - t_a$.

     b. **Calculate Alpha and Weight:**
        - If $j + 1 < N$:
          - $\alpha_{recip} = \exp(-\sigma \cdot \Delta t)$ [S1, unofficial].
        - Else (last step):
          - $\alpha_{recip} = 1.0$ [S1, unofficial].
        - Add safety epsilon: $\alpha_{recip} \leftarrow \alpha_{recip} + 10^{-10}$ [S1, unofficial].
        - $weight = (1.0 - \alpha_{recip}) \cdot \alpha_{through}$ [S1, unofficial].

     c. **Accumulate Radiance:**
        - Sample emission $L_e$ at $t_a$ (or midpoint, depending on specific grid query implementation not fully detailed, but typically at sample point) [S1, unofficial].
        - If primal mode: $L \leftarrow L + weight \cdot L_e$ [S1, unofficial].
        - Else (dual mode): $L \leftarrow L - weight \cdot L_e$ [S1, unofficial].

     d. **Update State:**
        - $j \leftarrow j + 1$.
        - $t_a \leftarrow t_b$.
        - $\alpha_{through} \leftarrow \alpha_{through} \cdot \alpha_{recip}$ [S1, unofficial].
        - $W_{sum} \leftarrow W_{sum} + weight$ [S1, unofficial].
        - Update `still_walking` mask: $j < N$ [S1, unofficial].

5. **Background Contribution:**
   - Calculate background contribution: $L_{bg} = (1 - W_{sum}) \cdot L_{emitter}$ [S1, unofficial].
   - If `hide_emitters` is true and $W_{sum} == 0$, $L_{bg} = 0$ [S1, unofficial].
   - Add $L_{bg}$ to $L$ for active pixels [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_integrator(ray_o, ray_d, scene, grid, params):
    # Parameters
    N = params['queries_per_ray']  # Default 128
    noise_std = params['density_noise_std']  # Default 0
    jitter = params['jittering_enabled']  # Default True
    activation = params['activation']  # Default 'identity'
    hide_emitters = params['hide_emitters']  # Default False
    
    # Initialization
    L = 0.0
    alpha_through = 1.0
    W_sum = 0.0
    j = 0
    primal_mode = True  # Assumed default, check scene context
    
    # Ray Intersection
    si = scene.intersect(ray_o, ray_d)
    if not si.hit:
        return 0.0  # Or background color
    
    t_exit = si.t  # Assuming si.t is the exit point for the medium
    active_mask = True  # Simplified for single ray
    
    # Step Size
    step_size = step_size_for_marching(N, 0, si.t, jitter)
    jitter_sample = sampler.sample()
    
    t_a = si.t
    
    while active_mask and j < N:
        # Calculate next t
        t_b = query_t_for_step(j + 1, step_size, jitter_sample, jitter)
        delta_t = t_b - t_a
        
        # Calculate alpha_recip
        if j + 1 < N:
            # Sample density at t_a or midpoint? Sources imply using sigma at step
            sigma = grid.sample_density(t_a)  # Implementation detail [G]
            # Apply noise
            if noise_std > 0:
                sigma += noise_std * random_noise()
            # Apply activation
            if activation == 'relu':
                sigma = max(0.0, sigma)
            alpha_recip = exp(-sigma * delta_t)
        else:
            alpha_recip = 1.0
            
        # Safety epsilon
        alpha_recip += 1e-10
        
        # Weight
        weight = (1.0 - alpha_recip) * alpha_through
        
        # Accumulate
        emission = grid.sample_emission(t_a)  # Implementation detail [G]
        if primal_mode:
            L += weight * emission
        else:
            L -= weight * emission
            
        # Update state
        j += 1
        t_a = t_b
        alpha_through *= alpha_recip
        W_sum += weight
        
        # Update mask
        if j >= N:
            active_mask = False
            
    # Background
    if hide_emitters and W_sum == 0:
        L_bg = 0.0
    else:
        L_bg = (1.0 - W_sum) * scene.background_emission()
        
    L += L_bg
    return L
```

**Parameters:**
- `queries_per_ray`: Number of steps. Default: 128 [S1, unofficial].
- `density_noise_std`: Standard deviation for density noise. Default: 0 [S1, unofficial].
- `jittering_enabled`: Enable jittering. Default: True [S1, unofficial].
- `activation`: Density activation ('identity', 'relu'). Default: 'identity' [S1, unofficial].
- `hide_emitters`: Hide background if weight sum is zero. Default: False [S1, unofficial].

**Edge Cases:**
- **Precision:** Add $10^{-10}$ to `alpha_recip` to prevent division by zero or NaN in gradient computation [S1, unofficial].
- **Boundaries:** Ensure `t_b` does not exceed `t_exit` if the medium is bounded [S1, unofficial].
- **Degenerate Input:** If `delta_t` is 0, skip step or handle as no-op [G].

## Operating characteristics
- **Time Complexity:** $O(N)$ per ray, where $N$ is `queries_per_ray` [S1, unofficial].
- **Memory Complexity:** $O(V)$ for the dense grid, where $V$ is the volume size [S1, unofficial].
- **Performance:** Depends on grid resolution and $N$. Higher $N$ improves accuracy but increases cost [S1, unofficial].
- **Strengths:** Simple, differentiable, suitable for inverse rendering [S1, unofficial].
- **Limitations:** Dense grid memory usage, no direction-dependent emission [S1, unofficial].

## Background
Originated from NeRF (Neural Radiance Fields) but adapted for unbiased inverse volume rendering using dense grids [S1, unofficial]. Developed by EPFL researchers [S1, unofficial].

## Sources
[S1] https://github.com/rgl-epfl/unbiased-inverse-volume-rendering, https://github.com/rgl-epfl/unbiased-inverse-volume-rendering, fetched 2026-10-26 (unofficial)
