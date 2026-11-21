# Voxel Grid Ray Marching

Also called: VoxelGridBase::rayMarch

## Overview
Voxel Grid Ray Marching is an algorithm to traverse a 3D voxel grid along a ray, returning the IDs of all visited cells [S1, unofficial]. It handles rays starting outside the grid by computing the entry point intersection with the bounding box. It builds on [DN_step_map](8596-dn-step-map.md), which casts a ray into the voxel volume to find the first intersecting voxel, returning hit position, voxel data, and surface normal.

## How it works
**Inputs:**
- Ray origin $O$ and direction $D$ (normalized).
- Voxel grid bounding box defined by minimum corner $B_{min}$ and maximum corner $B_{max}$.
- Grid dimensions $N_x, N_y, N_z$ (number of voxels per axis).
- Voxel size $s$ (assumed uniform across axes).

**Outputs:**
- List of visited voxel indices (IDs).

**Data Structures:**
- `currentVoxel`: Integer array of size 3, storing the current voxel index $(i, j, k)$.
- `steps`: Integer array of size 3, storing the direction of increment for each axis ($+1$ or $-1$).
- `deltas`: Float array of size 3, storing the distance to cross one voxel boundary along each axis.
- `ts`: Float array of size 3, storing the distance from the ray origin to the next boundary crossing along each axis.
- `finalVoxels`: Integer array of size 3, storing the boundary voxel index where traversal stops for each axis.

**Steps:**

1.  **Check Intersection with Bounding Box:**
    Determine if the ray intersects the grid bounding box. If the ray origin is inside the box, the intersection point is the origin. Otherwise, compute the intersection point $P_{entry}$ with the box faces. If there is no intersection, return an empty list.

2.  **Clamp Start Position:**
    Clamp the start position $P_{start}$ to ensure it lies strictly inside the last voxel to avoid boundary exclusion errors due to floating-point precision or floor operations.
    $$ P_{start} = \max(P_{entry}, B_{min}) $$
    $$ P_{start} = \min(P_{start}, B_{max} - \epsilon \cdot s) $$
    Where $\epsilon$ is a small constant (e.g., $0.01$) [S1, unofficial].

3.  **Determine Current Voxel:**
    Calculate the initial voxel index containing $P_{start}$:
    $$ currentVoxel[c] = \left\lfloor \frac{P_{start}[c] - B_{min}[c]}{s} \right\rfloor $$
    for each axis $c \in \{x, y, z\}$ [S1, unofficial].

4.  **Calculate Step Directions:**
    Determine the direction of traversal for each axis based on the ray direction:
    $$ steps[c] = \begin{cases} 1 & \text{if } D[c] \ge 0 \\ -1 & \text{otherwise} \end{cases} $$

5.  **Calculate Deltas:**
    Compute the distance to cross one voxel boundary along each axis:
    $$ deltas[c] = \frac{s}{|D[c]|} $$
    If $D[c] = 0$, the ray is parallel to the axis boundaries; handle as a special case (e.g., set $deltas[c] = \infty$) [G].

6.  **Calculate Fractional Distance:**
    Compute the fractional distance from $B_{min}$ to $P_{start}$, normalized by cell size:
    $$ frac[c] = \frac{P_{start}[c] - B_{min}[c]}{s} - \left\lfloor \frac{P_{start}[c] - B_{min}[c]}{s} \right\rfloor $$

7.  **Initialize T-values and Final Voxels:**
    Initialize the distance to the next boundary crossing $ts[c]$ and the final voxel index $finalVoxels[c]$ for each axis:
    $$ ts[c] = \begin{cases} deltas[c] \cdot (1.0 - frac[c]) & \text{if } D[c] \ge 0 \\ deltas[c] \cdot frac[c] & \text{otherwise} \end{cases} $$
    $$ finalVoxels[c] = \begin{cases} N_c & \text{if } D[c] \ge 0 \\ -1 & \text{otherwise} \end{cases} $$

8.  **Traverse Voxel Grid:**
    Loop indefinitely:
    a. Add $currentVoxel$ to the list of visited voxel IDs.
    b. Find the axis $c_{min}$ with the minimum $ts$ value: $c_{min} = \arg\min_c ts[c]$.
    c. Increment the current voxel index along $c_{min}$: $currentVoxel[c_{min}] += steps[c_{min}]$.
    d. If $currentVoxel[c_{min}] == finalVoxels[c_{min}]$, break the loop.
    e. Update the t-value for axis $c_{min}$: $ts[c_{min}] += deltas[c_{min}]$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_grid_ray_marching(ray_origin, ray_dir, box_min, box_max, dims, cell_size):
    """
    Traverses a 3D voxel grid along a ray.
    
    Args:
        ray_origin: (3,) array-like, ray origin point.
        ray_dir: (3,) array-like, normalized ray direction vector.
        box_min: (3,) array-like, minimum corner of the bounding box.
        box_max: (3,) array-like, maximum corner of the bounding box.
        dims: (3,) array-like, number of voxels per axis (Nx, Ny, Nz).
        cell_size: float, size of each voxel (assumed uniform).
        
    Returns:
        List of voxel indices (tuples of 3 integers) visited by the ray.
    """
    import numpy as np
    
    # Step 1: Check intersection with bounding box
    # Simplified intersection check; assumes ray intersects if it passes through the box
    # In a full implementation, compute entry point P_entry here.
    # For this method, we assume P_entry is computed or ray_origin is inside.
    # If ray_origin is outside, compute intersection with box faces.
    # If no intersection, return empty list.
    
    # Placeholder for intersection logic if origin is outside
    # P_start = compute_intersection(ray_origin, ray_dir, box_min, box_max)
    # If no intersection: return []
    
    # Use ray_origin as start if inside, else use intersection point
    P_start = np.array(ray_origin)
    
    # Step 2: Clamp start position
    epsilon = 0.01
    P_start = np.maximum(P_start, box_min)
    P_start = np.minimum(P_start, box_max - epsilon * cell_size)
    
    # Step 3: Determine current voxel
    current_voxel = np.floor((P_start - box_min) / cell_size).astype(int)
    
    # Step 4: Calculate step directions
    steps = np.where(ray_dir >= 0, 1, -1)
    
    # Step 5: Calculate deltas
    # Handle zero direction components to avoid division by zero
    abs_dir = np.abs(ray_dir)
    deltas = np.where(abs_dir > 1e-8, cell_size / abs_dir, np.inf)
    
    # Step 6: Calculate fractional distance
    frac = ((P_start - box_min) / cell_size) - np.floor((P_start - box_min) / cell_size)
    
    # Step 7: Initialize T-values and final voxels
    ts = np.where(ray_dir >= 0, deltas * (1.0 - frac), deltas * frac)
    final_voxels = np.where(ray_dir >= 0, dims, -1)
    
    visited_cells = []
    
    # Step 8: Traverse
    while True:
        visited_cells.append(tuple(current_voxel))
        
        # Find axis with minimum t-value
        min_idx = np.argmin(ts)
        
        # Increment current voxel along that axis
        current_voxel[min_idx] += steps[min_idx]
        
        # Check if we have reached the final voxel for that axis
        if current_voxel[min_idx] == final_voxels[min_idx]:
            break
            
        # Update t-value for the axis
        ts[min_idx] += deltas[min_idx]
        
    return visited_cells
```

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels visited along the ray. In the worst case, $N$ is proportional to the diagonal length of the grid in voxel units.
- **Memory Complexity:** $O(1)$ auxiliary space, excluding the output list of visited voxels.
- **Strengths:** Simple to implement, efficient for uniform grids, handles boundary conditions explicitly.
- **Limitations:** Does not handle non-uniform voxel sizes without modification. Precision issues may occur at boundaries, mitigated by clamping.
- **When to use:** When a simple, fast traversal of a uniform voxel grid is required, and all visited voxels need to be processed.
- **When not to use:** For non-uniform grids, or when only the first hit is needed (use [DN_step_map](8596-dn-step-map.md)).

## Background
This method is derived from the implementation in the FDS-GS repository [S1, unofficial]. It is a standard approach for ray-voxel intersection, often referred to as the Amanatides-Woo algorithm or DDA traversal, adapted for specific grid structures.

## Sources
[S1] whoiszzj/FDS-GS, https://github.com/whoiszzj/FDS-GS, fetched 2026-10-27 (unofficial)
