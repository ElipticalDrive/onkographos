# Aggressive Ray Marching with SDF

Also called: Ray Marching with SDF, Ray Marching Ratio

## Overview
Aggressive Ray Marching with SDF is a rendering technique that uses Signed Distance Field (SDF) values to dynamically determine step sizes along camera rays, allowing for large skips in empty space and fine-grained sampling near surfaces [S1, unofficial]. It is distinguished from standard uniform ray marching by its use of a configurable `ray_marching_ratio` to scale the SDF-derived step size, and from pure SDF rendering by its integration with neural implicit representations (latent codes) and an explicit termination mask based on depth bounds and SDF thresholds [S1, unofficial].

## How it works
**Inputs:**
- Camera rays `cam_rays_now` (origin and direction) [S1, unofficial].
- Initial depth offsets `init_zdepth_now` derived from an `unfinished_mask` [S1, unofficial].
- A neural network decoder `self.decoder` and a latent code vector `latent` [S1, unofficial].
- Parameters: `clamp_dist` (default 0.1), `stop_threshold`, `ray_marching_ratio`, `maxbound_zdepth`, `march_step`, `buffer_size` [S1, unofficial].

**Outputs:**
- Tensors `sdf_list`, `marching_zdepth_list`, and `points_list` concatenated along dimension 0 to form shapes `(self.march_step, N)`, `(self.march_step, N)`, and `(self.march_step, N, 3)` respectively [S1, unofficial].

**Data Structures:**
- `unfinished_mask`: Boolean tensor indicating which rays are still active.
- `marching_zdepth`: Accumulated step distance along the ray.
- `points_now`: 3D coordinates of sample points.
- `sdf_now`: Raw SDF values from the decoder.
- `sdf_marching`: Clamped SDF values used for stepping.

**Algorithm:**
1. Initialize `init_zdepth_now` and `marching_zdepth_now` for all rays where `unfinished_mask` is true [S1, unofficial].
2. Generate sample points `points_now` along `cam_rays_now` starting at `init_zdepth_now + marching_zdepth_now` [S1, unofficial].
3. Decode the SDF value `sdf_now` for `points_now` using `self.decoder` and `latent` [S1, unofficial].
4. Clamp `sdf_now` to the range `[-clamp_dist, clamp_dist]` to produce `sdf_marching` [S1, unofficial].
5. Update `marching_zdepth` by adding `sdf_marching * ray_marching_ratio` [S1, unofficial].
6. Update `unfinished_mask`:
   - Keep rays where the total depth (`marching_zdepth + init_zdepth`) is less than `maxbound_zdepth`.
   - Keep rays where `abs(sdf_marching) >= stop_threshold`.
   - Keep rays that were previously unfinished [S1, unofficial].
7. Terminate the loop if no rays remain unfinished (`torch.nonzero(unfinished_mask).shape[0] == 0`), padding the result lists to `self.buffer_size` if necessary [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def aggressive_ray_marching(cam_rays_origins, cam_rays_directions, init_zdepth_now, unfinished_mask, 
                            decoder, latent, clamp_dist=0.1, stop_threshold=0.01, 
                            ray_marching_ratio=1.0, maxbound_zdepth=10.0, 
                            march_step=100, buffer_size=1):
    """
    Performs aggressive ray marching using SDF values.
    Assumes cam_rays_origins and cam_rays_directions are arrays of shape (N, 3).
    """
    sdf_list = []
    marching_zdepth_list = []
    points_list = []
    
    marching_zdepth_now = [0.0] * len(init_zdepth_now)
    
    for step in range(march_step):
        # Check if any rays are unfinished
        if not any(unfinished_mask):
            break
            
        # Get indices of unfinished rays
        active_indices = [i for i, active in enumerate(unfinished_mask) if active]
        
        # Initialize depths for current step
        current_z = [init_zdepth_now[i] + marching_zdepth_now[i] for i in active_indices]
        
        # Generate 3D point samples along camera rays
        # points = origin + direction * depth
        points_now = []
        for i, idx in enumerate(active_indices):
            ox, oy, oz = cam_rays_origins[idx]
            dx, dy, dz = cam_rays_directions[idx]
            z = current_z[i]
            points_now.append((ox + dx * z, oy + dy * z, oz + dz * z))
        
        # Decode SDF values
        sdf_now = [decoder(p, latent) for p in points_now]
        
        # Clamp SDF values
        sdf_marching = [max(-clamp_dist, min(clamp_dist, s)) for s in sdf_now]
        
        # Update marching depth
        for i, idx in enumerate(active_indices):
            marching_zdepth_now[idx] += sdf_marching[i] * ray_marching_ratio
        
        # Store results
        sdf_list.append(sdf_marching)
        marching_zdepth_list.append([marching_zdepth_now[i] for i in active_indices])
        points_list.append(points_now)
        
        # Update unfinished mask
        for i, idx in enumerate(active_indices):
            total_depth = marching_zdepth_now[idx] + init_zdepth_now[idx]
            condition_depth = total_depth < maxbound_zdepth
            condition_sdf = abs(sdf_marching[i]) >= stop_threshold
            unfinished_mask[idx] = condition_depth and condition_sdf
            
    # Pad results to buffer_size
    # Note: Actual padding logic depends on batch processing requirements.
    # The source indicates concatenation along dim 0 to form (march_step, N)
    # Here we assume the lists are already structured for concatenation
    
    return sdf_list, marching_zdepth_list, points_list
```

**Parameters:**
- `clamp_dist`: Maximum distance used to clamp SDF values. Default 0.1 [S1, unofficial].
- `stop_threshold`: Minimum absolute SDF value to continue. If `abs(sdf) < stop_threshold`, the ray is finished [S1, unofficial].
- `ray_marching_ratio`: Scaling factor for the SDF step size [S1, unofficial].

**Edge Cases:**
- If `latent_tensor` does not correspond to a valid shape, `points_tmp` is `None` and Chamfer distance calculation fails [S1, unofficial].
- Precision issues may arise if `stop_threshold` is too small relative to floating-point precision [G].
- Boundary conditions are handled by `maxbound_zdepth` [S1, unofficial].

## Subtopics

### [Pyramid Recursive Ray Marching](3802-pyramid-recursive-ray-marching.md)
A multi-scale ray marching technique that starts at a low resolution (large scale) to initialize depth, then upsamples and refines the depth and SDF at higher resolutions (smaller scales) recursively [3802-pyramid-recursive-ray-marching.md]. It is chosen when multi-scale detail is required without excessive computation at high resolution from the start.

### Hybrid Ray Marching
Integrates ray marching with distance-based empty space leaping for complex scenes, using coarse SDFs to compute large skips in low-density regions before switching to fine-grained tracing near features [S2]. It balances speed and accuracy in multi-material volumes [S2].

## Operating characteristics
**Time Complexity:** Linear with respect to the number of active rays and steps, but reduced by large skips in empty space [S1, unofficial][S2].
**Memory Complexity:** Proportional to `march_step * N` for storing SDF, depth, and point lists [S1, unofficial].
**Performance:** Efficient for scenes with large empty regions due to SDF-based leaping [S2].
**Strengths:** Adaptive step size, efficient empty space skipping, compatible with neural implicit representations [S1, unofficial][S2].
**Limitations:** Requires SDF computation (neural or explicit), sensitive to `clamp_dist` and `stop_threshold` parameters [S1, unofficial].
**When to Use:** Scenes with significant empty space and complex surfaces, especially when using neural implicit representations [S1, unofficial][S2].
**When Not to Use:** Uniform media where constant step marching is more appropriate [S2].

## Background
The technique leverages the properties of SDFs for efficient ray traversal, a concept well-established in computer graphics, but applies it within a neural rendering framework using latent codes [S1, unofficial]. It is part of a broader family of SDF-accelerated ray marching methods [1629-performance-optimizations-for-sdf-ray-marching.md].

## Sources
[S1] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S2] Volume ray casting — Grokipedia, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
