# Proximity Clouds


## Overview
Proximity clouds are an acceleration technique for 3D grid traversal that assists rays traversing empty space by using pre-computed distance values to perform long skips along the ray direction [S1][S2, S3, unofficial]. Unlike standard empty-space skipping which may step voxel-by-voxel, proximity clouds store distance transform values in empty volume regions where volume densities are irrelevant, effectively replacing density data with distance-to-nearest-voxel information without increasing storage [S3, unofficial]. This method is an extension of adaptive ray traversal [8185-adaptive-ray-traversal.md] that leverages the geometric structure of empty space to leap over large voids.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume grid where empty voxels contain distance transform values (distance to the nearest non-empty/data voxel) instead of density values.
*   **Output:** A ray traversal path that skips empty regions, reducing the number of sample points evaluated.

**Data Structures:**
The data structure is a 3D grid of voxels. For empty voxels, the stored value is a scalar representing the distance to the nearest occupied voxel. The specific bit-packing or data type (e.g., integer vs. float) is not given in the sources, but the value $n$ represents the number of voxels to skip [S2, unofficial].

**Algorithm Steps:**
1.  **Pre-processing:** Compute the distance transform of the object. For each empty voxel, calculate the distance to the nearest data voxel [S2, S3, unofficial]. Store this value in the voxel's memory location, replacing the density value [S3, unofficial].
2.  **Ray Traversal:**
    a.  Cast a ray through the volume.
    b.  When the ray enters a voxel with a proximity value $n$ (where $n > 0$), skip the next $n$ voxels along the ray direction [S2, unofficial].
    c.  Resume traversal at the voxel immediately following the skipped region [S2, unofficial].
    d.  If the ray encounters a non-empty voxel (or a voxel with value 0), process it normally or continue traversal [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_proximity_clouds(ray, volume_grid):
    """
    ray: Object with origin, direction, and step_size.
    volume_grid: 3D array where empty voxels store distance to nearest data voxel.
    """
    current_pos = ray.origin
    while current_pos is within bounds:
        voxel = volume_grid.get_voxel_at(current_pos)
        
        # If voxel is empty and has a proximity value
        if voxel.is_empty and voxel.value > 0:
            # Skip 'n' voxels, where n is the stored distance value
            skip_distance = voxel.value * ray.step_size
            current_pos = current_pos + (ray.direction * skip_distance)
        else:
            # Process the voxel (e.g., sample density, composite color)
            process_voxel(voxel, ray)
            # Advance by one step for standard traversal or next check
            current_pos = current_pos + (ray.direction * ray.step_size)
            
        if current_pos is outside bounds or ray is terminated:
            break
```

**Parameters:**
*   **Distance Transform:** The method relies on the accuracy of the distance transform. Typical values are not specified, but the value $n$ directly dictates the skip length [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Missed Rays:** Rays that ultimately miss the object are often still slowed down because proximity clouds are iso-distance layers insensitive to viewing direction [S3, unofficial]. The distance is computed based on object geometry, not the ray's path, so a ray passing through a large empty region far from the object may still encounter proximity voxels that do not offer significant skips relative to the ray's trajectory [S3, unofficial].
*   **Boundary Precision:** The skip length is discrete (number of voxels). If the ray direction does not align perfectly with the voxel grid axes, the skip might overshoot or undershoot the intended empty region, potentially leading to redundant checks or missing data [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Reduces the number of sample points for rays passing through large empty regions, potentially offering $O(1)$ skip per empty region visited. However, for rays missing the object, performance may degrade compared to view-sensitive methods due to the lack of directional awareness [S3, unofficial].
*   **Memory Complexity:** Does not increase storage requirements compared to a standard density grid, as distance values replace density values in empty regions [S3, unofficial].
*   **Strengths:** Simple to implement as an extension of adaptive ray traversal; effective for rays passing near object boundaries; no additional memory overhead for empty space [S3, unofficial].
*   **Limitations:** Insensitive to viewing direction, leading to inefficiencies for rays that miss the object or pass through empty space far from the object [S3, unofficial].
*   **When to Use:** Suitable for scenes with large empty regions and when memory is constrained. Less suitable for scenes with many rays missing the object or when view-sensitive acceleration is critical.

## Background
Proximity clouds were proposed by Cohen and Sheffer in 1994 [S1][S3, unofficial]. The technique was introduced to accelerate ray traversal in empty space by using distance values to perform long skips [S1]. It is an extension of adaptive ray traversal [8185-adaptive-ray-traversal.md] and has been discussed in various surveys on volume rendering and grid-based ray tracing [S1][S2, S3, unofficial].

## Sources
[S1] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
[S2] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
[S3] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
