# Connected Component Labeling for Voxel Filtering

Also called: CCL Voxel Filtering, Large Component Filtering, Max Connected Component Extraction

## Overview
Connected Component Labeling for Voxel Filtering identifies and retains only large connected components of occupied voxels in a 3D boolean grid, filtering out small noise or isolated artifacts [S2, unofficial]. It operates by performing a flood-fill traversal with a configurable neighborhood distance, enforcing spherical connectivity within cubic bounds to determine adjacency [S2, unofficial]. Unlike sibling methods that sample uniformly or extract arbitrary subsets, this method specifically targets topological coherence, making it suitable for cleaning sparse voxel representations where small disconnected clusters represent noise rather than signal [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `voxels`, a boolean matrix of shape `[D, H, W]` representing the 3D voxel grid [S2, unofficial].
- Input: `distance`, an integer neighborhood radius for connectivity (default 3) [S2, unofficial].
- Input: `min_component_size`, an integer threshold for the minimum number of voxels required to retain a component (default 10) [S2, unofficial].
- Output: `filtered_voxels`, a boolean matrix of the same shape `[D, H, W]` containing only the retained large components [S2, unofficial].

**Data Structures:**
- `voxels_copy`: A boolean matrix of shape `[D, H, W]`, initialized as a copy of `voxels` [S2, unofficial]. Used to track visited status; a voxel is marked `False` once visited [S2, unofficial].
- `component`: A boolean matrix of shape `[D, H, W]`, initialized to all `False` [S2, unofficial]. Tracks the current connected component being explored [S2, unofficial].
- `stack`: A list of coordinates `[x, y, z]` used for depth-first traversal [S2, unofficial].
- `component_list`: A list of `component` matrices, storing all identified connected components [S2, unofficial].
- `max_component`: A boolean matrix of shape `[D, H, W]`, initialized to all `False` [S2, unofficial]. Accumulates the final filtered result [S2, unofficial].

**Steps:**
1. Assert `distance > 0` [S2, unofficial].
2. Initialize `voxels_copy` as a deep copy of the input `voxels` [S2, unofficial].
3. Initialize `component_list` as an empty list [S2, unofficial].
4. Iterate over all grid positions `(startx, starty, startz)` in the dimensions of `voxels` [S2, unofficial].
5. If `voxels_copy[startx, starty, startz]` is `False`, continue to the next position [S2, unofficial].
6. Initialize `component` as a zeroed boolean matrix of shape `[D, H, W]` [S2, unofficial].
7. Initialize `stack` with `[startx, starty, startz]` [S2, unofficial].
8. Mark `component[startx, starty, startz]` as `True` [S2, unofficial].
9. Mark `voxels_copy[startx, starty, startz]` as `False` to indicate it has been visited [S2, unofficial].
10. While `stack` is not empty:
    a. Pop `(x, y, z)` from `stack` [S2, unofficial].
    b. Retrieve the category/value of the voxel at `(x, y, z)` from the original `voxels` grid [S2, unofficial].
    c. Iterate through neighbor coordinates `(i, j, k)` in the cubic range `[x-distance, x+distance]` for all three axes (inclusive) [S2, unofficial].
    d. Calculate the squared Euclidean distance: $d^2 = (i-x)^2 + (j-y)^2 + (k-z)^2$ [S2, unofficial].
    e. If $d^2 > distance^2$, skip this neighbor [S2, unofficial].
    f. Check if `(i, j, k)` is within grid bounds and `voxels_copy[i, j, k]` is `True` (occupied and unvisited) [S2, unofficial].
    g. Check if the category/value of `voxels[i, j, k]` matches the category/value of `voxels[x, y, z]` [S2, unofficial].
    h. If both conditions are met:
        i. Mark `component[i, j, k]` as `True` [S2, unofficial].
        ii. Mark `voxels_copy[i, j, k]` as `False` [S2, unofficial].
        iii. Push `(i, j, k)` to `stack` [S2, unofficial].
11. After the stack is empty, append `component` to `component_list` [S2, unofficial].
12. Initialize `max_component` as a zeroed boolean matrix of shape `[D, H, W]` [S2, unofficial].
13. Iterate through each `component` in `component_list`:
    a. Calculate the size of the component: $size = \sum component$ (sum of `True` values) [S2, unofficial].
    b. If $size > min\_component\_size$, perform a logical OR with `max_component`: $max\_component = max\_component \lor component$ [S2, unofficial].
14. Return `max_component` as the filtered voxel grid [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def connected_component_labeling_filter(voxels, distance=3, min_component_size=10):
    """
    Filters a 3D boolean voxel grid by retaining only large connected components.
    
    Args:
        voxels (np.ndarray): Boolean array of shape (D, H, W).
        distance (int): Neighborhood radius for connectivity.
        min_component_size (int): Minimum voxel count to retain a component.
        
    Returns:
        np.ndarray: Filtered boolean array of shape (D, H, W).
    """
    if distance <= 0:
        raise ValueError("Distance must be greater than 0")
        
    # Copy input to track visited status
    voxels_copy = np.copy(voxels)
    component_list = []
    D, H, W = voxels.shape
    
    # Iterate over all voxels to find connected components
    for startx in range(D):
        for starty in range(H):
            for startz in range(W):
                # Skip if already visited or empty
                if not voxels_copy[startx, starty, startz]:
                    continue
                
                # Initialize new component and stack
                component = np.zeros((D, H, W), dtype=bool)
                stack = [[startx, starty, startz]]
                
                # Mark start node
                component[startx, starty, startz] = True
                voxels_copy[startx, starty, startz] = False
                
                # Get category of the starting voxel for consistency check
                start_category = voxels[startx, starty, startz]
                
                # Flood fill
                while len(stack) > 0:
                    x, y, z = stack.pop()
                    
                    # Define neighbor range based on distance
                    x_min = max(0, x - distance)
                    x_max = min(D, x + distance + 1)
                    y_min = max(0, y - distance)
                    y_max = min(H, y + distance + 1)
                    z_min = max(0, z - distance)
                    z_max = min(W, z + distance + 1)
                    
                    for i in range(x_min, x_max):
                        for j in range(y_min, y_max):
                            for k in range(z_min, z_max):
                                # Calculate squared Euclidean distance
                                dist_sq = (i - x)**2 + (j - y)**2 + (k - z)**2
                                
                                # Enforce spherical connectivity
                                if dist_sq > distance * distance:
                                    continue
                                
                                # Check bounds (already handled by range limits)
                                # Check if occupied and unvisited in copy
                                if voxels_copy[i, j, k]:
                                    # Check category consistency
                                    if voxels[i, j, k] == start_category:
                                        component[i, j, k] = True
                                        voxels_copy[i, j, k] = False
                                        stack.append([i, j, k])
                
                component_list.append(component)
    
    # Filter components by size
    max_component = np.zeros((D, H, W), dtype=bool)
    for component in component_list:
        if np.sum(component) > min_component_size:
            max_component = np.logical_or(max_component, component)
            
    return max_component
```

**Parameters:**
- `distance`: Integer, neighborhood radius for connectivity. Two blocks are connected if their squared Euclidean distance is $\le distance^2$ [S2, unofficial]. Default value is 3 [S2, unofficial].
- `min_component_size`: Integer, the minimum number of voxels required for a component to be retained. Default value is 10 [S2, unofficial].

**Edge Cases and Pitfalls:**
- The algorithm modifies `voxels_copy` in-place to prevent re-processing [S2, unofficial].
- The neighbor check relies on bounds checking [G].
- The category/value check ensures that only voxels with the same semantic label are connected [S2, unofficial].
- The code snippet provided in the source cuts off before the final mask application, but the logic implies that `max_component` is the final output [S2, unofficial].

## Subtopics

### [Connected Component Labeling](20133-connected-component-labeling.md)
Identifies and assigns unique identifiers to contiguous groups of occupied voxels in a 3D grid. It provides the raw labeling data structure required for filtering or identifying separate objects, unlike this method which directly filters by component size. Choose it when you need explicit labels for each distinct geometric entity rather than a binary mask.

## Operating characteristics
**Time and Memory Complexity:**
- Time Complexity: $O(D \cdot H \cdot W \cdot R^3)$, where $D, H, W$ are grid dimensions and $R$ is the `distance` radius. In the worst case, every voxel is visited, and for each, a cubic neighborhood of size $(2 \cdot distance + 1)^3$ is checked [G].
- Memory Complexity: $O(D \cdot H \cdot W \cdot C)$, where $C$ is the number of connected components. Each component is stored as a full boolean grid of the same size as the input [G]. This can be significant for large grids with many small components [G].

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- The method scales linearly with the number of voxels but cubically with the neighborhood radius. For large `distance` values, performance degrades rapidly [G].

**Strengths and Limitations:**
- Strengths: Effectively removes small noise and isolated artifacts. Configurable connectivity allows tuning for specific voxel resolutions [S2, unofficial].
- Limitations: High memory usage due to storing all components. Slow for large grids with many small components. The spherical connectivity check within a cubic range adds computational overhead [G].

**When to Use:**
- Use when the voxel grid contains significant noise or small disconnected artifacts that need to be removed while preserving the main object structure [S2, unofficial].
- Use when semantic consistency is required for connectivity (i.e., only voxels of the same class should be connected) [S2, unofficial].

**When Not to Use:**
- Do not use when memory is constrained, as storing all components requires significant RAM [G].
- Do not use when real-time performance is critical and the grid is large, due to the cubic neighborhood search [G].

## Background
**Origin:**
- The method is described in the context of the VoxelSplat project, which focuses on 3D occupancy prediction and evaluation [S2, unofficial].

**History and Development:**
- The specific implementation details, including the spherical connectivity check and the filtering logic, are derived from the VoxelSplat evaluation metrics code [S2, unofficial].

**Where it is used:**
- Used in the evaluation of 3D occupancy prediction models, specifically for calculating metrics like RayIoU and mAVE on datasets such as NuScenes [S2, unofficial].

## Sources
[S1] navis-org/sparse-cubes, https://github.com/navis-org/sparse-cubes, fetched 2026-10-26 (unofficial)
[S2] ZZY816/VoxelSplat, https://github.com/ZZY816/VoxelSplat, fetched 2026-10-27 (unofficial)
