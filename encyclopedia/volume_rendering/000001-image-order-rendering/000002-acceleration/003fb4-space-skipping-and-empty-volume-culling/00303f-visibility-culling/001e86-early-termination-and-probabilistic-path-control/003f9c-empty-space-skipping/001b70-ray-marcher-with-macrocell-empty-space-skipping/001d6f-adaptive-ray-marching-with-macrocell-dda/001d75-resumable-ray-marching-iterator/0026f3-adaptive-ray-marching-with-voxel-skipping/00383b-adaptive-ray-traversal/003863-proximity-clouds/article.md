# Proximity Clouds


## Overview
Proximity clouds are an acceleration technique for 3D grid traversal that replaces density values in empty space with pre-computed distance-to-nearest-occupied-voxel metrics, enabling long, safe skips along a ray [S1][S2, S3, S4, unofficial]. Unlike standard empty-space culling which often relies on binary occupancy or fixed block sizes, this method allows arbitrary step leaps based on the geometric distance to the object boundary, significantly reducing traversal overhead in large empty regions [S2, S4, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D volume grid where each voxel is either occupied (contains data) or empty.
*   **Output:** A transformed grid (the "proximity cloud") where empty voxels store the Euclidean distance to the nearest occupied voxel.

**Data Structures**
The grid layout remains identical to the original volume grid. Each voxel $V$ stores a single scalar value $d$:
*   If $V$ is occupied: $d$ is the original data value (e.g., density).
*   If $V$ is empty: $d$ is the distance to the closest occupied voxel.
*   *Note:* Since empty regions have no density, this transformation incurs no additional storage cost compared to storing zeros or a binary flag [S3, unofficial].

**Preprocessing Step**
1.  Identify all occupied voxels in the input volume.
2.  Compute the distance transform for all empty voxels. For each empty voxel, calculate the shortest distance to any occupied voxel in the grid [S2, S4, unofficial].
    *   This is typically done using a multi-pass algorithm (e.g., Chamfer distance or exact Euclidean distance transform) [G].
3.  Store the resulting distance values in the empty voxels.

**Ray Traversal Algorithm**
The ray marcher iterates along the ray direction. At each step, it checks the current voxel's value $d$:

1.  **Check Occupancy:** If the current voxel is occupied (or if $d$ indicates occupancy, depending on implementation details where occupied voxels retain their data values), perform standard volume rendering operations (sampling, compositing) [S4, unofficial].
2.  **Check Proximity:** If the current voxel is empty, read its stored distance value $n$ [S2, S4, unofficial].
3.  **Leap:** Advance the ray position by $n$ voxels along the ray direction [S2, S4, unofficial].
    *   *Assurance:* The algorithm guarantees that no occupied voxels exist within the span of $n$ steps from the current position [S4, unofficial].
4.  **Repeat:** Continue traversal from the new position.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_proximity_clouds(ray_origin, ray_direction, volume_grid):
    """
    ray_origin: (x, y, z) float
    ray_direction: (dx, dy, dz) float
    volume_grid: 3D array where empty voxels store distance to nearest occupied voxel
    """
    # Initialize ray parameters
    t = 0.0
    current_pos = ray_origin
    
    # Determine voxel step size based on ray direction and grid resolution
    # This assumes a standard DDA-like step calculation or normalized grid coordinates
    step_x = 1.0 / abs(ray_direction.x) if ray_direction.x != 0 else float('inf')
    step_y = 1.0 / abs(ray_direction.y) if ray_direction.y != 0 else float('inf')
    step_z = 1.0 / abs(ray_direction.z) if ray_direction.z != 0 else float('inf')
    
    # Standard DDA initialization would go here to find the first voxel
    # For simplicity, we assume current_pos is aligned to a voxel grid start
    voxel_x, voxel_y, voxel_z = floor(current_pos)
    
    while t < ray_length:
        # Get the value at the current voxel
        # Note: In a real implementation, bounds checking is required
        dist = volume_grid[voxel_x, voxel_y, voxel_z]
        
        # Check if the voxel is occupied.
        # Implementation detail: Occupied voxels typically store density > 0.
        # Empty voxels store distance > 0.
        # We assume a threshold or a separate flag distinguishes them.
        # If the source implies occupied voxels keep their data, we check if it's a "proximity" value.
        # However, S4 says "encounter an occupied voxel... handle as usual" or "proximity voxel... take leap".
        # Let's assume occupied voxels have a specific property or we check if dist is a "skip" value.
        # A common approach: if the voxel is marked as empty/proximity, use dist.
        
        if is_occupied(voxel_x, voxel_y, voxel_z):
            # Handle occupied voxel: sample, composite, potentially terminate if opaque
            sample = volume_grid[voxel_x, voxel_y, voxel_z]
            # ... standard volume rendering logic ...
            # Advance by one voxel step for precision
            voxel_x, voxel_y, voxel_z = next_voxel(voxel_x, voxel_y, voxel_z, ray_direction)
            t += min(step_x, step_y, step_z) # Advance to next voxel boundary
        else:
            # It is a proximity voxel (empty)
            n = dist # Distance to nearest occupied voxel
            
            # Leap forward by n voxels
            # We need to advance the ray position by n voxel units along the ray
            # Assuming unit voxel size for simplicity in distance metric
            leap_distance = n * voxel_size
            
            # Update ray position
            current_pos = ray_origin + ray_direction * (t + leap_distance)
            t += leap_distance
            
            # Update voxel coordinates
            voxel_x, voxel_y, voxel_z = floor(current_pos)
            
            # Pitfall: If the leap lands exactly on an occupied voxel or inside it,
            # the next iteration will handle it. If it lands in empty space,
            # the next proximity value will be smaller (distance to the now-nearer boundary).
            
            # Safety check: if n is 0 or negative, break to avoid infinite loop
            if n <= 0:
                break
                
    return color
```

**Parameters and Edge Cases**
*   **Distance Metric:** The distance $n$ must be consistent with the voxel grid spacing. If the distance transform uses Euclidean distance in world units, the leap must convert this to voxel steps [S4, unofficial].
*   **Missed Rays:** Rays that miss the object entirely may still traverse many proximity layers, as the distance values are iso-distance shells around the object and do not account for the ray's trajectory relative to the object's center [S3, unofficial].
*   **Traversal Efficiency:** The method's effectiveness depends heavily on the underlying line traversal algorithm's ability to jump an arbitrary number of steps efficiently [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing is $O(N)$ where $N$ is the number of voxels, dominated by the distance transform. Traversal time is reduced significantly in empty regions, scaling with the number of boundary crossings rather than empty volume size.
*   **Memory Complexity:** $O(N)$, same as the original volume, as distance values replace density values in empty space [S3, unofficial].
*   **Strengths:** Eliminates the need for hierarchical structures (like octrees) for empty-space culling; simple to implement on regular grids; effective for large empty spaces [S1][S4, unofficial].
*   **Limitations:** Ineffective for rays missing the object, as they still traverse the proximity shells [S3, unofficial]. The leap size is limited by the local distance to the boundary, which can be small near complex geometries [S4, unofficial].
*   **When to use:** Ideal for volumes with large, contiguous empty regions surrounding the object.
*   **When not to use:** When the object is highly complex with many small voids, or when viewing direction is highly variable and misses are common (consider view-sensitive extensions like Sramek and Kaufman [S3, unofficial]).

## Background
Proximity clouds were proposed by Cohen and Sheffer in 1994 [S1][S3, unofficial]. The method extends the "vicinity cloud" concept, which used a one-voxel-deep flag to indicate proximity to data, by computing the actual distance to the nearest occupied voxel for all empty voxels [S4, unofficial]. This allows for arbitrary step leaps rather than just single-voxel checks. The technique is particularly noted for its storage efficiency, as it repurposes empty space storage without increasing memory footprint [S3, unofficial].

## Sources
[S1] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
[S2] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
[S3] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S4] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
