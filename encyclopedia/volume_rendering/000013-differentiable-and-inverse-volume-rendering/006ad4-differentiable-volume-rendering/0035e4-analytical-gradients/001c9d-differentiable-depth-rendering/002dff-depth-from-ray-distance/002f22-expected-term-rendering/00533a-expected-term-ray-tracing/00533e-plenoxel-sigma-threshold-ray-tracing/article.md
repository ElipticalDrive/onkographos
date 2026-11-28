# Plenoxel Sigma Threshold Ray Tracing

Also called: Dex-NeRF sigma threshold ray tracing

## Overview
Plenoxel Sigma Threshold Ray Tracing is a CUDA kernel function that computes the distance to the first sample along a ray where the volume density ($\sigma$) exceeds a specified threshold, returning the normalized distance to that sample [S1, S2, unofficial]. It differs from standard volume rendering by discretizing the volume into opaque/transparent regions based on a density cutoff rather than integrating transmittance, and it differs from its parent [Expected Term Ray Tracing](14910-expected-term-ray-tracing.md) by using a hard threshold on density rather than integrating the expected distance contribution [S1, S2, unofficial].

## How it works
**Inputs:**
- `ray`: A ray structure containing `tmin`, `tmax`, and world-space step size `world_step` [S1, S2, unofficial].
- `opt`: An options structure containing `step_size` (the step size in lattice units) and `sigma_thresh` (the density threshold) [S1, S2, unofficial].
- `grid.density_data`: A sparse voxel grid storing density values [S1, unofficial].

**Outputs:**
- `*out`: A float pointer storing the result. If no sample exceeds the threshold, it remains 0.0 [S1, S2, unofficial].

**Data Structures:**
- The density grid is accessed via lattice indices `l` and fractional positions derived from the ray position [S1, S2, unofficial].
- Trilinear interpolation is used to sample $\sigma$ from `grid.density_data` [S1, S2, unofficial].

**Algorithm:**
1. If `ray.tmin > ray.tmax`, set `*out = 0.f` and return [S2, unofficial].
2. Initialize `t = ray.tmin` and `*out = 0.f` [S1, S2, unofficial].
3. Loop while `t <= ray.tmax`:
   a. Compute the ray position at `t` [S1, S2, unofficial].
   b. Clamp the position if necessary [S2, unofficial].
   c. Compute the lattice indices `l` and the fractional position within the voxel [S1, S2, unofficial].
   d. Trilinearly interpolate the density $\sigma$ from `grid.density_data` using `l` and the fractional position [S1, S2, unofficial].
   e. If $\sigma > \text{sigma\_thresh}$:
      i. Set `*out = (t / opt.step_size) * ray.world_step` [S1, S2, unofficial].
      ii. Break the loop [S1, S2, unofficial].
   f. Advance `t` by `opt.step_size` [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trace_ray_sigma_thresh(ray, opt, grid_density_data):
    # Guard: invalid ray range
    if ray.tmin > ray.tmax:
        return 0.0

    t = ray.tmin
    out = 0.0

    while t <= ray.tmax:
        # Compute ray position at t
        pos = ray.origin + t * ray.direction
        
        # Clamp position (implementation detail for boundary handling)
        pos = clamp(pos, grid.bounds.min, grid.bounds.max)
        
        # Compute lattice indices and fractional part
        l, frac = compute_lattice_and_fraction(pos, grid.voxel_size)
        
        # Trilinear interpolation of density
        sigma = trilinear_interpolate(grid_density_data, l, frac)
        
        # Check threshold
        if sigma > opt.sigma_thresh:
            # Normalize distance: t is in world units, opt.step_size is in lattice steps?
            # Formula from sources: (t / opt.step_size) * ray.world_step
            # Note: This formula implies t and opt.step_size are in compatible units,
            # and ray.world_step scales the result.
            out = (t / opt.step_size) * ray.world_step
            break
        
        # Advance t
        t += opt.step_size

    return out
```

**Parameters:**
- `sigma_thresh`: The density threshold. Typical values are not given in the sources.
- `opt.step_size`: The step size for ray marching. Typical values are not given in the sources.

**Edge Cases and Pitfalls:**
- If `ray.tmin > ray.tmax`, the function returns 0.0 immediately [S2, unofficial].
- Precision issues may arise if `opt.step_size` is very small or very large relative to `ray.world_step` [G].
- If the ray never encounters a density above the threshold within `tmax`, `*out` remains 0.0 [S1, S2, unofficial].

## Operating characteristics
**Time Complexity:** Linear in the number of steps along the ray, $O((t_{max} - t_{min}) / \text{step\_size})$ [S1, S2, unofficial].
**Memory Complexity:** $O(1)$ auxiliary memory per ray, plus access to the sparse voxel grid [S1, S2, unofficial].
**Performance:** Not given in the sources.
**Strengths:** Simple implementation, fast convergence if the threshold is low [G].
**Limitations:** Discontinuous output with respect to ray direction or density changes [G].
**When to use:** When a binary opaque/transparent representation is sufficient [G].
**When not to use:** When soft boundaries or alpha blending are required [G].

## Background
Plenoxel Sigma Threshold Ray Tracing is part of the Dex-NeRF project, which is associated with the NeRF-Factory repository by POSTECH, KAIST, and Kakao Brain Corp. [S1, S2, unofficial]. It builds on the Plenoxel sparse voxel representation [S1, S2, unofficial]. The method is implemented as a CUDA kernel within the `svox2` library [S2, unofficial].

## Sources
[S1] kakaobrain/nerf-factory, https://github.com/kakaobrain/nerf-factory, fetched 2026-10-26 (unofficial)
[S2] OSHMOS/IPRF, https://github.com/OSHMOS/IPRF, fetched 2026-10-27 (unofficial)
