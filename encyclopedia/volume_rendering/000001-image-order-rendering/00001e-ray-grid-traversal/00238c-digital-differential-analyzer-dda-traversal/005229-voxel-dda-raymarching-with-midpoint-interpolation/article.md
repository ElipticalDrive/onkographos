# Voxel DDA Raymarching with Midpoint Interpolation

Also called: DDA ray marching, sphere tracing (referenced as alternative), triangular prism method

## Overview
Voxel DDA Raymarching with Midpoint Interpolation is a ray-terrain intersection technique for smooth heightfields represented by voxel corner volumes. It computes surface heights at voxel midpoints from adjacent corner values and linearly interpolates the ray's intersection point along the traversal edge [S1, unofficial]. Unlike standard DDA which steps through discrete cells, this method refines the intersection within a single voxel using midpoint geometry, offering a balance between the speed of coarse traversal and the accuracy of continuous surface tracing [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $O$ and direction $D$.
- Voxel grid of corner heights $H_{corner}(i,j,k)$.
- Voxel size $V_{size}$.
- Entry UV coordinates on the voxel face where the ray enters.

**Outputs:**
- Intersection UV coordinate $u_{intersect}$.
- Intersection height $y_{intersect}$.
- Boolean flag indicating hit or miss.

**Data Structures:**
- `midpoints`: A vector or struct containing height values at the midpoints of voxel edges. Specifically, `midpoints.y` represents the height at the midpoint of the edge being traversed.
- `entry_uv`: The UV coordinate where the ray enters the current voxel face.
- `parallel_uv`: The UV coordinate parallel to the ray's primary traversal direction along the voxel face.
- `perpendicular_uv`: The UV coordinate perpendicular to the ray's primary traversal direction, used for logic switching in the upper half of the voxel.

**Algorithm Steps:**

1.  **Compute Midpoint Height:**
    For the edge being traversed, calculate the height at the midpoint using the two adjacent corner volumes:
    $$ h_{mid} = \frac{h_{corner1} + h_{corner2}}{2} $$
    [S1, unofficial]

2.  **Determine Traversal Logic Branch:**
    Check the Z-component of the entry UV coordinate ($entry\_uv.z$) to determine the traversal logic branch.
    - If $entry\_uv.z < 0.5$: Use "normal" logic (lower half).
    - If $entry\_uv.z \ge 0.5$: Use "perpendicular" logic (upper half).
    [S1, unofficial]

3.  **Calculate Intersection Percentage:**
    Determine how far along the voxel edge the ray intersects the surface. This is done by comparing the ray's Y-coordinate ($ray\_y$) against the side value ($h_{side}$) and the midpoint value ($h_{mid}$).
    $$ p = \frac{ray\_y - h_{side}}{h_{mid} - h_{side}} $$
    [S1, unofficial]
    *Note: $h_{side}$ is the height of the corner at the start of the edge segment being considered.*

4.  **Compute Final Intersection UV:**
    Interpolate between the entry UV and the parallel UV using the calculated percentage $p$.
    $$ u_{intersect} = entry\_uv + p \times (parallel\_uv - entry\_uv) $$
    [S1, unofficial]
    Alternatively, using the specific formula provided in sources:
    $$ u_{intersect} = entry\_uv + \frac{midpoints.y - entry\_uv.y}{midpoints.y - midpoints.x.y} \times (parallel\_uv - entry\_uv) $$
    [S1, unofficial]
    *Clarification: The source formula notation is ambiguous. The logical interpretation is linear interpolation based on height difference.*

5.  **Ray Height Comparison:**
    Compare the ray's Y-coordinate at the current step against the intersection height.
    - If $ray\_y > h_{intersect}$: The ray is above the surface. Continue marching to the next voxel/step.
    - If $ray\_y \le h_{intersect}$: The ray has hit or passed below the surface. Return the color at $u_{intersect}$ and break.
    [S1, unofficial]

6.  **Logic Switching (Upper Half):**
    If the traversal logic branch is "perpendicular" ($entry\_uv.z \ge 0.5$):
    - Switch from using `parallel_uv` to `perpendicular_uv` for the interpolation direction.
    - Adjust the starting point reference to ensure consistent reading direction regardless of ray entry angle.
    [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_dda_midpoint_interpolation(ray_origin, ray_direction, voxel_grid, voxel_size):
    """
    Performs Voxel DDA Raymarching with Midpoint Interpolation.
    
    Args:
        ray_origin: (x, y, z)
        ray_direction: (dx, dy, dz)
        voxel_grid: 3D array of corner heights
        voxel_size: float, size of one voxel
        
    Returns:
        intersection_uv: (u, v) or None if no hit
    """
    # Initialize ray parameters
    t = 0.0
    current_pos = ray_origin
    
    # Determine initial voxel and entry UV
    # [G] Standard DDA initialization to find first voxel intersection
    voxel_coords = world_to_voxel(current_pos)
    entry_uv = compute_entry_uv(current_pos, ray_direction, voxel_coords, voxel_size)
    
    while True:
        # 1. Compute Midpoint Height
        # Identify the two corners defining the edge being traversed
        # [G] Logic to select corner1 and corner2 based on traversal direction and entry_uv
        corner1_height = get_corner_height(voxel_grid, voxel_coords, corner_index_1)
        corner2_height = get_corner_height(voxel_grid, voxel_coords, corner_index_2)
        midpoint_height = (corner1_height + corner2_height) / 2.0
        
        # 2. Determine Traversal Logic Branch
        is_upper_half = entry_uv.z >= 0.5
        
        # 3. Calculate Intersection Percentage
        # Get the side value (height at the start of the edge segment)
        side_height = get_side_height(voxel_grid, voxel_coords, is_upper_half)
        
        # Get ray Y at current t
        ray_y = ray_origin.y + ray_direction.y * t
        
        # Avoid division by zero
        denom = midpoint_height - side_height
        if abs(denom) < 1e-6:
            # Degenerate case: flat surface
            percentage = 0.0
        else:
            percentage = (ray_y - side_height) / denom
            
        # Clamp percentage to [0, 1] for safety
        percentage = max(0.0, min(1.0, percentage))
        
        # 4. Compute Final Intersection UV
        if is_upper_half:
            # Use perpendicular UV logic
            parallel_uv = entry_uv.perpendicular_uv
            # Adjust entry point reference if necessary
            # [G] Ensure consistent start/end reference
            base_uv = entry_uv.parallel_uv # Switched base
        else:
            # Use normal logic
            parallel_uv = entry_uv.parallel_uv
            base_uv = entry_uv
            
        intersect_uv = base_uv + percentage * (parallel_uv - base_uv)
        
        # 5. Ray Height Comparison
        # Calculate the height of the surface at the intersection point
        # [G] Interpolate surface height at intersect_uv using corner heights
        surface_height = interpolate_surface_height(voxel_grid, voxel_coords, intersect_uv)
        
        if ray_y > surface_height:
            # Ray is above surface, continue marching
            # Advance ray to next voxel boundary
            t = advance_ray_to_next_voxel(ray_origin, ray_direction, t, voxel_coords, voxel_size)
            voxel_coords = update_voxel_coords(voxel_coords, ray_direction)
            entry_uv = compute_entry_uv(ray_origin + ray_direction * t, ray_direction, voxel_coords, voxel_size)
        else:
            # Ray has hit the surface
            return intersect_uv, surface_height
```

**Parameters:**
- `entry_uv`: The UV coordinate where the ray enters the voxel face. [S1, unofficial]
- `parallel_uv`: The UV coordinate parallel to the ray's traversal direction along the voxel face. [S1, unofficial]
- `perpendicular_uv`: The UV coordinate perpendicular to the ray's traversal direction, used when switching logic for the upper voxel half. [S1, unofficial]

**Edge Cases and Pitfalls:**
- **Ray Entry Direction Ambiguity:** The algorithm must consistently read from the same starting point to the same ending point regardless of whether the ray enters from above, below, or the side. Failure to fix the start/end reference causes incorrect interpolation. [S1, unofficial]
- **Boundary Threshold at 0.5:** The `entry_uv.z < 0.5` check splits the voxel traversal logic. Crossing this threshold requires switching from parallel UV to perpendicular UV and adjusting the entry point reference, otherwise the intersection math breaks. [S1, unofficial]

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel step, with a constant number of arithmetic operations for midpoint calculation and interpolation. Total complexity depends on the number of voxels traversed, similar to standard DDA.
- **Memory Complexity:** $O(1)$ auxiliary memory, as it only stores current voxel coordinates and UV values.
- **Strengths:** Provides smoother terrain representation than nearest-neighbor methods by using midpoint interpolation. Simpler to implement than full continuous ray marching.
- **Limitations:** Accuracy is limited by the voxel resolution and the linear interpolation assumption. May exhibit artifacts if the terrain has high-frequency details smaller than the voxel size.
- **When to Use:** Suitable for real-time terrain rendering where performance is critical but some smoothness is desired.
- **When Not to Use:** Not suitable for highly detailed or non-smooth terrains where higher-order interpolation or continuous ray marching is required.

## Background
The method is described in an unofficial video tutorial on smooth terrain rendering using voxel DDA raymarching [S1, unofficial]. It references sphere tracing as an alternative approach but focuses on a DDA-based traversal with midpoint interpolation for terrain heightfields. The technique appears to be a practical implementation for game development or real-time graphics, emphasizing simplicity and performance over theoretical optimality.

## Sources
[S1] Smooth terrain with voxel DDA raymarching, https://www.youtube.com/watch?v=CxoC0E7inUg, fetched 2026-10-25 (unofficial)
