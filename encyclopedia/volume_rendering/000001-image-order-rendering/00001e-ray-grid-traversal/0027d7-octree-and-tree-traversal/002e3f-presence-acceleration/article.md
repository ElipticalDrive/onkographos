# Presence Acceleration

Also called: Binary Pyramid Ray Traversal, Binary pyramid traversal, Hierarchical Spatial Enumeration, Levoy's octree acceleration

## Overview
Presence Acceleration is a ray traversal technique that uses a binary pyramid to skip empty regions in a volume by identifying cells containing at least one opaque voxel [S1, S2, S3, unofficial]. It solves the problem of inefficiently traversing large transparent areas in dense volume data. Unlike standard octree traversal which requires pointer chasing or complex stack management, Presence Acceleration uses a pointerless, complete hierarchical structure that allows for simple, predictable memory access patterns [S1, S2, unofficial]. It sets itself apart from siblings like [Sparse Voxel Octree Ray Traversal](2451-sparse-voxel-octree-ray-traversal.md) by relying on a fixed-resolution grid hierarchy rather than a sparse, dynamic tree structure.

## How it works
**Inputs:** A volume dataset of resolution $N \times N \times N$ where $N = 2^M + 1$ for some integer $M \ge 1$ [S1, S2, unofficial]. A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
**Outputs:** A set of samples taken along the ray within non-empty elementary cells, used for compositing.

### Data Structures
The method constructs a **Binary Pyramid**, which is a set of $M + 1$ volumes indexed by level $m = 0, \dots, M$ [S1, S2, unofficial].
*   **Level $V_0$:** Measures $(N - 1) \times (N - 1) \times (N - 1)$ cells. Each cell represents the rectangular region bounded by eight neighboring voxels in the original volume data.
    *   **Value:** A cell value is $1$ if at least one of its eight corner voxels has opacity $> 0$. Otherwise, the value is $0$ [S1, S2, unofficial].
*   **Level $V_m$ ($m > 0$):** Measures $(N - 1)/2^m$ cells on each side.
    *   **Value:** A cell at level $m$ is $0$ if all four corner cells at level $m-1$ are $0$. Otherwise, the value is $1$ [S1, S2, unofficial].
*   **Level $V_M$:** A single cell ($1 \times 1 \times 1$) representing the entire volume.

The pyramid is "complete" and "pointerless," meaning the children of any cell at level $m$ are located at fixed offsets in the array $V_{m-1}$ [S1, unofficial].

### Algorithm Steps

1.  **Pyramid Construction:**
    *   Compute $V_0$ by checking the opacity of the 8 corner voxels for each cell.
    *   Compute higher levels $V_m$ iteratively. For each cell in $V_m$, check the 4 corner cells in $V_{m-1}$. If all are $0$, set $V_m$ cell to $0$; else set to $1$ [S1, S2, unofficial].

2.  **Ray Initialization:**
    *   Calculate the entry point of the ray into the single cell at the top level $V_M$ [S1, S2, unofficial]. Let this be the current position $\mathbf{p}$ and the current level $m = M$ [S3, unofficial].

3.  **Traversal Loop:**
    *   While $m \ge 0$:
        *   Check the value of the current cell in $V_m$ at position $\mathbf{p}$ [S1, S2, unofficial].
        *   **If the cell value is 0 (Empty):**
            *   Calculate the intersection point of the ray with the boundary of the current cell [S3, unofficial].
            *   Advance the ray to this intersection point [S3, unofficial].
            *   Determine the next cell at the same level $m$ that the ray will enter [S1, S2, unofficial].
            *   **Parent Check:** If the parent of this next cell differs from the parent of the previous cell, move up to the parent level ($m \leftarrow m + 1$) and investigate the parent cell [S1, S2, unofficial].
            *   If the parent cell is also empty ($0$), skip it and continue iterating upwards until a non-empty parent is found or the root is reached [S1, S2, unofficial].
            *   If the parent is non-empty, move down to the appropriate child at level $m$ [S1, S2, unofficial].
        *   **If the cell value is 1 (Non-Empty):**
            *   If $m > 0$, move down one level ($m \leftarrow m - 1$) [S3, unofficial].
            *   Determine which child cell at level $m-1$ encloses the current ray position [S3, unofficial].
            *   Update the current position to the entry point of the ray into this child cell [S3, unofficial].
            *   Repeat the loop for the new level.
            *   If $m = 0$, the ray has reached an **elementary cell** (a cell in $V_0$) that contains at least one opaque voxel.

4.  **Sampling and Compositing:**
    *   Once an elementary cell at level $0$ is identified, take samples at evenly spaced locations along the ray within this cell [S1, S2, unofficial].
    *   Perform volume compositing using these samples [S1, S2, unofficial].
    *   After compositing, the ray continues from the exit point of this elementary cell, and the traversal resumes from the current level (typically re-evaluating the current cell or moving to the next cell in $V_0$) [S1, S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def presence_acceleration_traversal(ray_origin, ray_direction, volume_resolution, pyramid):
    """
    ray_origin: (x, y, z)
    ray_direction: (dx, dy, dz)
    volume_resolution: N (where N = 2^M + 1)
    pyramid: list of 3D arrays [V_0, V_1, ..., V_M]
    """
    M = int(log2(volume_resolution - 1))
    
    # 1. Calculate entry into top level cell V_M
    # V_M is a single cell covering the whole volume [0, N-1]^3
    # Ray entry is simply the start of the ray or intersection with volume bounds
    t_entry = max(0, intersection_t(ray_origin, ray_direction, volume_bounds))
    current_pos = ray_origin + ray_direction * t_entry
    current_level = M
    
    # Helper to get cell index from position
    def get_cell_index(pos, level):
        # Map position to grid coordinates in V_level
        # V_level has side length (N-1)/2^level
        side_len = (volume_resolution - 1) / (2 ** level)
        # Assuming cell size is 1.0 in grid units for V_0
        # Scale position relative to volume bounds
        # Note: Exact mapping depends on coordinate system, assuming standard grid alignment
        x = int((pos.x - volume_min.x) / cell_size(level))
        y = int((pos.y - volume_min.y) / cell_size(level))
        z = int((pos.z - volume_min.z) / cell_size(level))
        return (x, y, z)

    # Helper to get parent index
    def get_parent_index(child_idx, level):
        # In a binary pyramid, parent is formed by 4 corners
        # Parent index is child_idx // 2
        return (child_idx[0] // 2, child_idx[1] // 2, child_idx[2] // 2)

    # Helper to get next cell intersection
    def next_cell_intersection(current_pos, ray_direction, cell_size):
        # Calculate t to exit current cell
        # Standard DDA step calculation
        tx = (cell_size - (current_pos.x % cell_size)) / ray_direction.x if ray_direction.x != 0 else inf
        ty = (cell_size - (current_pos.y % cell_size)) / ray_direction.y if ray_direction.y != 0 else inf
        tz = (cell_size - (current_pos.z % cell_size)) / ray_direction.z if ray_direction.z != 0 else inf
        
        t_max = min(tx, ty, tz)
        return current_pos + ray_direction * t_max

    # 2. Traversal Loop
    while current_level >= 0:
        cell_idx = get_cell_index(current_pos, current_level)
        cell_value = pyramid[current_level][cell_idx]
        
        if cell_value == 0:
            # Empty cell: skip
            cell_size = (volume_resolution - 1) / (2 ** current_level)
            next_pos = next_cell_intersection(current_pos, ray_direction, cell_size)
            
            # Determine next cell index at same level
            # This requires determining which face was exited
            # Simplified: assume we move to the next cell in traversal order
            # In practice, this involves checking ray direction signs
            next_idx = get_next_cell_index(cell_idx, ray_direction)
            
            # Check parent change
            current_parent = get_parent_index(cell_idx, current_level)
            next_parent = get_parent_index(next_idx, current_level)
            
            if current_parent != next_parent:
                # Move up to parent
                current_level += 1
                current_pos = next_pos # Update position to intersection
                # Re-evaluate parent cell
                continue
            
            # If parent is same, stay at same level
            current_pos = next_pos
            
        else:
            # Non-empty cell
            if current_level > 0:
                # Move down
                current_level -= 1
                # Determine which child encloses current_pos
                # Child index is derived from current_pos relative to child grid
                # Or from current_idx: child_idx = current_idx * 2 + offset
                # Offset determined by ray position within parent cell
                current_pos = get_entry_point_into_child(current_pos, ray_direction, current_level)
            else:
                # Reached elementary cell (level 0)
                # Sample and composite
                cell_size = 1.0 # V_0 cells are unit size in grid
                samples = take_evenly_spaced_samples(current_pos, ray_direction, cell_size, num_samples)
                composite(samples)
                
                # After compositing, advance to next cell in V_0
                next_pos = next_cell_intersection(current_pos, ray_direction, cell_size)
                current_pos = next_pos
                # Stay at level 0 to process next elementary cell
```

### Parameters and Edge Cases
*   **Volume Resolution:** Must be $N = 2^M + 1$. If the volume does not meet this, padding or resampling is required [S1, S2, unofficial].
*   **Precision:** Floating-point precision errors in ray-cell intersection calculations can lead to missed cells or double-sampling. Robust intersection tests are recommended [G].
*   **Homogeneity:** Presence Acceleration only detects "presence" of opacity. It does not handle homogeneous transparent regions efficiently. For that, Denskin and Hanrahan's Homogeneity Acceleration (using range pyramids) is an extension [S1, S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Construction is $O(N^3)$ for the volume, but the pyramid is smaller. Traversal time is proportional to the number of non-empty cells visited. In sparse volumes, it significantly outperforms fixed-step DDA by skipping large empty regions [S1, S2, unofficial].
*   **Memory Complexity:** The pyramid requires $O(N^3)$ bits (since it is a complete binary tree structure stored as a grid). Specifically, the total number of cells is $\sum_{m=0}^{M} ((N-1)/2^m)^3$. This is roughly $1/7$ of the original volume size if $N$ is large, but still significant for high-resolution volumes [S1, S2, unofficial].
*   **Strengths:** Simple implementation, no pointers, cache-friendly due to regular grid structure, effective for sparse volumes with large empty regions [S1, S2, S3, unofficial].
*   **Limitations:** Requires specific volume resolution ($2^M+1$). Less efficient than sparse octrees for extremely sparse data where the overhead of the pyramid is high. Does not handle homogeneous regions as well as range pyramids [S1, S2, unofficial].

## Background
Presence Acceleration was introduced by Marc Levoy in 1991 [S1, S2, S3, unofficial]. It is also known as Hierarchical Spatial Enumeration [S3, unofficial]. The method exploits the discrete nature of volume data to create a pointerless octree representation (binary pyramid) that allows for rapid traversal of empty space [S1, S2, unofficial]. It was later extended by Denskin and Hanrahan in 1992 with Homogeneity Acceleration, which used range pyramids to approximate homogeneous regions [S1, S2, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S2] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S3] Empty Space Skipping, https://users.cg.tuwien.ac.at/bruckner/homepage/content/mastersthesis/html/node11.html, fetched 2026-10-28 (unofficial)
