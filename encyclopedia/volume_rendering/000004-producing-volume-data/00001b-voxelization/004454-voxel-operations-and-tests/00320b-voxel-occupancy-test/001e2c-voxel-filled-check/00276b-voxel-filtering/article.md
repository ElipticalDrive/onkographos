# Voxel Filtering

Also called: Voxel Predicate Filter

## Overview
Voxel Filtering iterates over a voxel grid to identify and return the linear indices of voxels that satisfy a user-defined predicate function [S1, unofficial]. It solves the problem of isolating relevant data from a dense grid by discarding voxels that do not meet specific criteria [S1, unofficial].

## How it works
**Inputs:**
- `grid`: A 3D array of voxel data (e.g., `CellType`) [S1, unofficial].
- `func`: A callable predicate that takes a single voxel as input and returns a boolean indicating whether the voxel satisfies the condition [S1, unofficial].

**Outputs:**
- `indices`: A dynamic list (vector) of integers containing the linear indices of voxels where `func` returns `True` [S1, unofficial].

**Data Structures:**
- The grid is accessed as a flat array for iteration, with linear index $i$ mapping to the underlying voxel data [S1, unofficial].

**Steps:**
1. Initialize an empty output vector `output_indices` [S1, unofficial].
2. Determine the total number of voxels $N$ in the grid [S1, unofficial].
3. Iterate $i$ from $0$ to $N-1$:
   a. Retrieve the voxel value `cell` at linear index $i$ [S1, unofficial].
   b. Evaluate `result = func(cell)` [S1, unofficial].
   c. If `result` is `True`, append $i$ to `output_indices` [S1, unofficial].
4. Return `output_indices` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_filter(grid, func):
    """
    Filters a voxel grid based on a predicate function.
    
    Args:
        grid: 3D array or flattened list of voxel data.
        func: Function taking a single voxel element and returning a boolean.
        
    Returns:
        List of linear indices where func returns True.
    """
    output_indices = []
    N = len(grid)
    for i in range(N):
        cell = grid[i]
        if func(cell):
            output_indices.append(i)
    return output_indices
```

**Parameters:**
- `func`: A callable predicate that takes a single voxel (CellType) as input and returns a boolean indicating whether the voxel satisfies the condition [S1, unofficial].

**Edge Cases and Pitfalls:**
- If the grid is empty, the function returns an empty list [G].
- Precision issues may arise if the predicate relies on floating-point comparisons without a tolerance threshold [G].
- The method assumes a contiguous memory layout for linear indexing; non-contiguous grids may require coordinate-to-index mapping [G].

## Subtopics

### [Sample Uniform and Occupied Cells](4651-sample-uniform-and-occupied-cells.md)
Generates a mixed set of voxel indices for volume rendering by combining uniformly sampled cells with cells known to be occupied by geometry. It filters out cells with invalid density values (specifically -1). It is chosen when a renderer needs to balance coverage of empty space with detail in occupied regions. Link: [Sample Uniform and Occupied Cells](4651-sample-uniform-and-occupied-cells.md).

### [Volume Subset Extraction](5602-volume-subset-extraction.md)
Reduces a dense 3D voxel grid to a sparse set of voxels that satisfy a user-defined predicate, typically based on a transfer function evaluating density and gradient. It solves the problem of rendering or processing large volumes by discarding empty or irrelevant voxels, storing the result as a 2D array of splat rows to enable efficient front-to-back or back-to-front traversal. It differs from [Voxel Filtering](4035-voxel-filtering.md) by specifically structuring the output for splat-based rendering pipelines rather than returning a flat list of indices, and from [Sample Uniform and Occupied Cells](4651-sample-uniform-and-occupied-cells.md) by using a continuous transfer function threshold rather than simple occupancy checks. Link: [Volume Subset Extraction](5602-volume-subset-extraction.md).

### [Empty-Brick Threshold Calculation](8812-empty-brick-threshold-calculation.md)
Computes the voxel intensity threshold below which a volume brick is considered empty and can be skipped during streaming. Link: [Empty-Brick Threshold Calculation](8812-empty-brick-threshold-calculation.md).

### [Connected Component Labeling for Voxel Filtering](14971-connected-component-labeling-for-voxel-filtering.md)
Connected Component Labeling (CCL) for Voxel Filtering identifies and retains only the largest connected components of occupied voxels in a 3D boolean grid, filtering out small noise or isolated artifacts. It operates by performing a flood-fill traversal with a configurable neighborhood distance, enforcing spherical connectivity within cubic bounds to determine adjacency. Unlike sibling methods that sample uniformly or extract arbitrary subsets, this method specifically targets topological coherence, making it suitable for cleaning sparse voxel representations where small disconnected clusters represent noise rather than signal. Link: [Connected Component Labeling for Voxel Filtering](14971-connected-component-labeling-for-voxel-filtering.md).

### Grid Homogenization
Sets all non-empty voxels to `VOXEL_FREE`, effectively removing fragment IDs and marking all occupied space as free [S2, unofficial]. It iterates over all voxels and, if a voxel is not `VOXEL_EMPTY`, sets its value to `VOXEL_FREE` [S2, unofficial].

### Voxel Filtering by Value Range
Selects grid cells based on two disjoint value ranges for a physical property [S3]. Selection condition: $(−200 \le \text{pressure} \le 20) \lor (500 \le \text{pressure} \le 1000)$, where pressure is measured in Pascal units [S3].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] 3DGS-FLoD/flod, https://github.com/3DGS-FLoD/flod, fetched 2026-10-27 (unofficial)
[S2] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
[S3] Query-Driven Visualization of Time-Varying Adaptive Mesh Refinement Data, https://digital.library.unt.edu/ark:/67531/metadc897115/?q=amr volume rendering, fetched 2026-10-28
