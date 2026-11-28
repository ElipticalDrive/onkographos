# NeRF Background Rendering

Also called: Background Rendering with Occupancy Grid and Density Function

## Overview
This method renders the background behind the primary scene geometry using a separate occupancy grid and density function, distinct from the foreground scene representation [S1, unofficial]. It addresses the limitation of constant background colors by allowing the background to be modeled as a distinct volumetric entity, while still supporting a fallback constant color for rays that do not intersect the background geometry [S1, unofficial]. It builds on [Background Color Resolution](18468-background-color-resolution.md) by providing a learned volumetric background component.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origins ($t_{origins}$), ray directions ($t_{dirs}$), ray indices ($ray\_indices$), and the primary scene's bounding box intersection distance ($t_{max}$).
*   **Outputs:** A dictionary containing `comp_rgb` (composite RGB), `opacity`, `depth`, `rays_valid`, `num_samples`. In training mode, it also returns `weights`, `points`, `intervals`, and `ray_indices` [S1, unofficial].

**Data Structures:**
*   `geometry_bg`: A neural network or implicit function that takes 3D positions and returns density ($\sigma$) and features ($f$).
*   `texture_bg`: A neural network that takes features ($f$) and ray directions ($t_{dirs}$) to return RGB color.
*   `occupancy_grid_bg`: A sparse grid used for efficient ray marching culling [S1, unofficial].
*   `background_color`: A constant 3D vector used as the fallback background color [S1, unofficial].

**Parameters:**
*   `near_plane_bg`: The minimum distance for background ray marching, used when rays do not intersect the scene bounding box [S1, unofficial].
*   `far_plane_bg`: The maximum distance for background ray marching [S1, unofficial].
*   `render_step_size_bg`: The step size for background ray marching, controlling sampling granularity [S1, unofficial].

**Algorithm Steps:**

1.  **Define Density Query Function:**
    Create a function `sigma_fn(t_starts, t_ends, ray_indices)` that:
    *   Extracts ray origins and directions using `ray_indices`.
    *   Computes midpoints: $positions = t_{origins} + t_{dirs} \times \frac{t_{starts} + t_{ends}}{2}$.
    *   Returns density from `self.geometry_bg(positions)` [S1, unofficial].

2.  **Determine Ray Bounds:**
    For each ray, determine the near plane for background sampling:
    *   If the ray intersects the scene AABB ($t_{max} > 10^9$), set the near plane to $t_{max}$.
    *   Otherwise, set the near plane to `self.near_plane_bg` [S1, unofficial].

3.  **Ray Marching:**
    Perform ray marching using `ray_marching` with:
    *   `sigma_fn`
    *   `near_plane` (computed in Step 2)
    *   `far_plane=self.far_plane_bg`
    *   `render_step_size=self.render_step_size_bg`
    *   Optional occupancy grid `self.occupancy_grid_bg` [S1, unofficial].
    *   This returns $t_{starts}$, $t_{ends}$, and $ray\_indices$ for the sampled intervals.

4.  **Compute Sample Positions:**
    Calculate sample midpoints and 3D positions:
    $$midpoints = \frac{t_{starts} + t_{ends}}{2}$$
    $$positions = t_{origins} + t_{dirs} \times midpoints$$ [S1, unofficial]

5.  **Query Geometry and Texture:**
    *   Query background geometry for density and features:
        $$density, feature = self.geometry_bg(positions)$$
    *   Compute RGB from features and ray directions:
        $$rgb = self.texture_bg(feature, t_{dirs})$$ [S1, unofficial]

6.  **Compute Rendering Weights:**
    Calculate weights from density using `render_weight_from_density`:
    $$weights = render\_weight\_from\_density(t_{starts}, t_{ends}, density[..., None], ray\_indices=ray\_indices, n\_rays=n\_rays)$$ [S1, unofficial]

7.  **Accumulate Along Rays:**
    Accumulate opacity, depth, and RGB along rays using `accumulate_along_rays` with the computed weights to get initial `comp_rgb`, `opacity`, and `depth` [S1, unofficial].

8.  **Blend with Constant Background:**
    Add the constant background color to the accumulated RGB for rays with zero opacity:
    $$comp\_rgb = comp\_rgb + self.background\_color \times (1.0 - opacity)$$ [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_background(t_origins, t_dirs, ray_indices, t_max, n_rays):
    # Step 1: Define density query function
    def sigma_fn(t_starts, t_ends, ray_indices):
        # Extract ray origins and directions using ray_indices
        origins = t_origins[ray_indices]
        dirs = t_dirs[ray_indices]
        # Compute midpoints
        midpoints = (t_starts + t_ends) / 2.0
        positions = origins + dirs * midpoints
        # Return density from background geometry
        density, _ = self.geometry_bg(positions)
        return density

    # Step 2: Determine Ray Bounds
    # If ray intersects scene AABB (t_max > 1e9), use t_max; else use near_plane_bg
    near_planes = torch.where(t_max > 1e9, t_max, torch.full_like(t_max, self.near_plane_bg))
    
    # Step 3: Ray Marching
    # Perform ray marching with sigma_fn, near_planes, far_plane_bg, render_step_size_bg, occupancy_grid_bg
    t_starts, t_ends, ray_indices = ray_marching(
        sigma_fn=sigma_fn,
        near_plane=near_planes,
        far_plane=self.far_plane_bg,
        render_step_size=self.render_step_size_bg,
        occupancy_grid=self.occupancy_grid_bg,
        ray_indices=ray_indices
    )
    
    # Step 4: Compute Sample Positions
    midpoints = (t_starts + t_ends) / 2.0
    origins = t_origins[ray_indices]
    dirs = t_dirs[ray_indices]
    positions = origins + dirs * midpoints
    
    # Step 5: Query Geometry and Texture
    density, feature = self.geometry_bg(positions)
    rgb = self.texture_bg(feature, dirs)
    
    # Step 6: Compute Rendering Weights
    # density[..., None] adds a dimension for broadcasting
    weights = render_weight_from_density(
        t_starts, t_ends, density[..., None], ray_indices=ray_indices, n_rays=n_rays
    )
    
    # Step 7: Accumulate Along Rays
    comp_rgb, opacity, depth = accumulate_along_rays(
        weights, rgb, t_starts, t_ends, ray_indices, n_rays
    )
    
    # Step 8: Blend with Constant Background
    # Add constant background color for rays with zero opacity
    comp_rgb = comp_rgb + self.background_color * (1.0 - opacity)
    
    return {
        'comp_rgb': comp_rgb,
        'opacity': opacity,
        'depth': depth,
        'rays_valid': torch.ones(n_rays, dtype=torch.bool), # [G] Assuming all rays valid if not specified
        'num_samples': t_starts.shape[0] // n_rays # [G] Approximation if not explicitly returned
    }
```

## Subtopics

### [NeRF with Learned Background](20060-nerf-with-learned-background.md)
A NeRF variant that combines primary scene rendering with a separate learned background rendering pass, blending the results based on the primary scene's opacity. It addresses the limitation of constant background colors by allowing the background to be modeled as a distinct volumetric entity, while still supporting a fallback constant color for rays that do not intersect the background geometry.

## Operating characteristics
**Time and Memory Complexity:**
The complexity depends on the number of samples per ray, which is determined by `render_step_size_bg` and the distance between `near_plane_bg` and `far_plane_bg`. The occupancy grid `occupancy_grid_bg` accelerates ray marching by skipping empty space, reducing the number of density queries [S1, unofficial].

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Allows for a learned, potentially complex background rather than a flat color, improving realism for scenes with detailed backgrounds [S1, unofficial]. The occupancy grid provides efficiency gains [S1, unofficial].
*   **Limitations:** Requires additional memory for the background geometry and texture networks. If the background is truly distant and uniform, a constant color may be more efficient and sufficient [S1, unofficial].

**When to Use:**
Use when the background contains significant detail that cannot be captured by a constant color, and when the computational budget allows for an additional volumetric rendering pass [S1, unofficial].

## Background
This method is part of the GSDF (Gaussian Splatting with Density Fields) framework, which extends 3D Gaussian Splatting by incorporating density fields for background rendering [S1, unofficial]. It was introduced in the `city-super/GSDF` repository [S1, unofficial].

## Sources
[S1] GSDF, https://github.com/city-super/GSDF, fetched 2026-10-27 (unofficial)
