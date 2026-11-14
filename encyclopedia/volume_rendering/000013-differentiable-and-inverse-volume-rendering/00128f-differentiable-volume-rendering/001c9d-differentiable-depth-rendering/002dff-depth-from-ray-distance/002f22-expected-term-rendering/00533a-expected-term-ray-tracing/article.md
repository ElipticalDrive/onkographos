# Expected Term Ray Tracing

Also called: Expected Term Rendering Kernel, Plenoxel Expected Term Ray Tracing

## Overview
Expected Term Ray Tracing computes the expected distance to the first opaque surface (or significant density threshold) along a ray passing through a sparse voxel grid [S1, unofficial]. It differs from standard volume rendering by outputting a scalar expected depth value rather than a color or opacity-weighted sum, making it suitable for uncertainty estimation or specific depth-based loss functions [S1, unofficial]. It builds on [Expected Term Rendering](5952-expected-term-rendering.md) by providing the specific CUDA kernel implementation for this metric.

## How it works
**Inputs:**
- `grid`: Sparse voxel grid containing `density_data` (trilinearly interpolatable density field).
- `ray`: Ray specification including origin, direction, `tmin`, `tmax`, and `world_step` (physical length of one voxel step).
- `opt`: Options struct containing `sigma_thresh` (density threshold for opacity calculation), `stop_thresh` (transmission threshold for early termination), and `step_size` (number of samples per voxel).
- `out`: Output buffer for the computed expected term.

**Data Structures:**
- `grid.density_data`: Array of density values $\sigma$.
- `ray`: Struct with `origin` (float3), `direction` (float3), `tmin` (float), `tmax` (float), `world_step` (float).
- `opt`: Struct with `sigma_thresh` (float), `stop_thresh` (float), `step_size` (int).

**Algorithm:**
1. Initialize ray position and lattice indices using `fmaf`, `clamp`, `floor`, and fractional position calculations, identical to `trace_ray_cuvol` [S1, unofficial].
2. Initialize `log_transmit = 0.0f` (representing $\ln(T)$ where $T$ is transmission).
3. Initialize `outv = 0.0f` (accumulator for the expected term).
4. Loop while `t <= ray.tmax`:
   a. Compute ray position and lattice indices.
   b. Trilinearly interpolate density $\sigma$ from `grid.density_data` at the current ray position [S1, unofficial].
   c. If $\sigma > \text{opt.sigma\_thresh}$:
      i. Calculate weight: $w = e^{\text{log\_transmit}} \cdot (1 - e^{-\text{ray.world\_step} \cdot \sigma})$.
      ii. Update transmission: $\text{log\_transmit} \leftarrow \text{log\_transmit} - \text{ray.world\_step} \cdot \sigma$.
      iii. Accumulate expected term: $\text{outv} \leftarrow \text{outv} + w \cdot \frac{t}{\text{opt.step\_size}} \cdot \text{ray.world\_step}$.
   d. If $e^{\text{log\_transmit}} < \text{opt.stop\_thresh}$:
      i. Set $\text{log\_transmit} = -1000.0f$ (effectively zero transmission).
      ii. Break loop.
   e. Advance $t \leftarrow t + \text{opt.step\_size}$.
5. Write `*out = outv` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trace_ray_expected_term(grid, ray, opt, out):
    # Initialize ray state
    t = ray.tmin
    log_transmit = 0.0f
    outv = 0.0f
    
    # Precompute constants
    world_step = ray.world_step
    sigma_thresh = opt.sigma_thresh
    stop_thresh = opt.stop_thresh
    step_size = opt.step_size
    
    # Loop through ray
    while t <= ray.tmax:
        # Compute lattice indices and fractional position
        # (Identical to trace_ray_cuvol: fmaf, clamp, floor, fractional pos)
        # [G] Implementation detail: Calculate grid coordinates from ray origin + t * direction
        # and interpolate density sigma from grid.density_data
        
        sigma = trilinear_interpolate(grid.density_data, ray.position(t))
        
        if sigma > sigma_thresh:
            # Calculate weight: T * (1 - exp(-sigma * delta))
            # T = exp(log_transmit)
            weight = exp(log_transmit) * (1.0f - exp(-world_step * sigma))
            
            # Update log transmission
            log_transmit -= world_step * sigma
            
            # Accumulate expected term: weight * (t / step_size) * world_step
            # Note: (t / step_size) represents the normalized distance index
            outv += weight * (t / step_size) * world_step
            
        # Early termination if transmission is negligible
        if exp(log_transmit) < stop_thresh:
            log_transmit = -1000.0f
            break
            
        # Advance ray
        t += step_size
        
    out[0] = outv
```

**Parameters:**
- `sigma_thresh`: Density threshold above which opacity is calculated. Typical value: Not given in the sources.
- `stop_thresh`: Transmission threshold for early termination. Typical value: Not given in the sources.
- `step_size`: Number of samples per voxel. Typical value: Not given in the sources.

**Edge Cases:**
- If `sigma` never exceeds `sigma_thresh`, `outv` remains 0.
- If `log_transmit` underflows, it is clamped to -1000.0f to prevent numerical instability [S1, unofficial].

## Subtopics

### Plenoxel Sigma Threshold Ray Tracing
A variant that finds the first sample along a ray where density exceeds a given threshold, returning the normalized distance to that sample [S1, unofficial]. It initializes `t = ray.tmin` and `*out = 0.f`, loops while `t <= ray.tmax`, computes ray position and lattice indices, interpolates density $\sigma$, and if $\sigma > \text{sigma\_thresh}$, sets `*out = (t / opt.step_size) * ray.world_step` and breaks [S1, unofficial]. It is used when a hard threshold distance is needed rather than an expected value.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of samples along the ray, with early termination possible if transmission drops below `stop_thresh`.
- **Memory Complexity:** Constant, $O(1)$ auxiliary memory per ray, plus access to the sparse voxel grid.
- **Performance:** Implemented as a CUDA kernel for parallel execution on GPUs [S1, unofficial].
- **Strengths:** Efficient computation of expected depth metrics for differentiable rendering.
- **Limitations:** Requires a sparse voxel grid structure; performance depends on density distribution and threshold settings.
- **When to use:** When expected depth or uncertainty estimation is required for volume rendering.
- **When not to use:** When color or opacity-weighted accumulation is needed instead of a scalar depth metric.

## Background
The method is part of the NeRF-Factory project by POSTECH, KAIST, and Kakao Brain Corp., released in 2022 [S1, unofficial]. It is implemented as a CUDA kernel within the framework to support differentiable volume rendering techniques.

## Sources
[S1] kakaobrain/nerf-factory, https://github.com/kakaobrain/nerf-factory, fetched 2026-10-26 (unofficial)
