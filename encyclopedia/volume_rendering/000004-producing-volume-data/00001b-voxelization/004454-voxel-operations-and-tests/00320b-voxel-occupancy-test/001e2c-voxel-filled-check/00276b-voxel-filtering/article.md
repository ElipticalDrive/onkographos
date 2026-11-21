# Voxel Filtering

Also called: Voxel Predicate Filter

## Overview
Voxel Filtering iterates over a voxel grid to identify and return the linear indices of voxels that satisfy a user-defined predicate function [S1, unofficial]. It solves the problem of isolating relevant data from a dense grid by discarding voxels that do not meet specific criteria, such as occupancy or material properties. It differs from [Voxel Material Classification](3645-voxel-material-classification.md) by returning indices rather than class labels, and from [Voxel Solidity Check](3758-voxel-solidity-check.md) by allowing arbitrary boolean conditions rather than checking for structural integrity.

## How it works
**Inputs:**
- `grid`: A 3D array of voxel data (e.g., `CellType`).
- `predicate`: A callable function `func(cell) -> bool` that evaluates a single voxel [S1, unofficial].

**Outputs:**
- `indices`: A list of linear indices corresponding to voxels where `predicate` returns `True` [S1, unofficial].

**Data Structures:**
- The grid is treated as a flat array for iteration, with linear index $i$ mapping to 3D coordinates $(x, y, z)$ via standard row-major or column-major ordering depending on the implementation [S1, unofficial].
- The output is a dynamic list (vector) of integers.

**Steps:**
1. Initialize an empty list `output_indices`.
2. Determine the total number of voxels $N$ in the grid.
3. Iterate $i$ from $0$ to $N-1$:
   a. Retrieve the voxel value `cell` at linear index $i$.
   b. Evaluate `result = predicate(cell)`.
   c. If `result` is `True`, append $i$ to `output_indices`.
4. Return `output_indices`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_filter(grid, predicate):
    """
    Filters a voxel grid based on a predicate function.
    
    Args:
        grid: 3D array or flattened list of voxel data.
        predicate: Function taking a single voxel element and returning a boolean.
        
    Returns:
        List of linear indices where predicate returns True.
    """
    output_indices = []
    # Assuming grid is flattened or accessible via linear indexing
    # If grid is 3D, linear index i corresponds to specific (x,y,z)
    # The source implies direct iteration over elements [S1, unofficial]
    for i in range(len(grid)):
        cell = grid[i]
        if predicate(cell):
            output_indices.append(i)
    return output_indices
```

**Parameters:**
- `predicate`: A user-defined function. Typical predicates include checking if density $> 0$, checking if material ID $\neq$ background, or checking if opacity $> \epsilon$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- If the grid is empty, the function returns an empty list.
- Precision issues may arise if the predicate relies on floating-point comparisons (e.g., `cell > 0.0`) without a tolerance threshold [G].
- The method assumes a contiguous memory layout for linear indexing; non-contiguous grids may require coordinate-to-index mapping [G].

## Subtopics

### [Sample Uniform and Occupied Cells](4651-sample-uniform-and-occupied-cells.md)
Generates a mixed set of voxel indices by combining uniformly sampled cells with cells known to be occupied by geometry. It filters out cells with invalid density values (specifically -1). It is chosen when a renderer needs to balance coverage of empty space with detail in occupied regions. Link: [Sample Uniform and Occupied Cells](4651-sample-uniform-and-occupied-cells.md).

### [Volume Subset Extraction](5602-volume-subset-extraction.md)
Reduces a dense 3D voxel grid to a sparse list of significant voxels by applying a user-defined predicate, typically based on a transfer function. It differs from Voxel Filtering by organizing the output into a 2D array of splat rows for rasterization, and from Sample Uniform and Occupied Cells by using continuous transfer function evaluation rather than simple occupancy checks. It is chosen for efficient splat-based rendering of large volumes. Link: [Volume Subset Extraction](5602-volume-subset-extraction.md).

### [Empty-Brick Threshold Calculation](8812-empty-brick-threshold-calculation.md)
Computes the voxel intensity threshold below which a volume brick is considered empty and can be skipped during streaming. It determines a static cutoff value based on metadata and channel windows, enabling the renderer to avoid loading or processing bricks that contain no relevant data. It is chosen for memory-efficient streaming of large volumes. Link: [Empty-Brick Threshold Calculation](8812-empty-brick-threshold-calculation.md).

### [Connected Component Labeling for Voxel Filtering](14971-connected-component-labeling-for-voxel-filtering.md)
Identifies and retains only the largest connected components of occupied voxels in a 3D boolean grid, filtering out small noise or isolated artifacts. It operates by performing a flood-fill traversal with a configurable neighborhood distance, enforcing spherical connectivity within cubic bounds to determine adjacency. Unlike sibling methods that sample uniformly or extract arbitrary subsets, this method specifically targets topological coherence, making it suitable for cleaning sparse voxel representations where small disconnected clusters represent noise rather than signal. Link: [Connected Component Labeling for Voxel Filtering](14971-connected-component-labeling-for-voxel-filtering.md).

### Grid Homogenization
Sets all non-empty voxels to `VOXEL_FREE`, effectively removing fragment IDs and marking all occupied space as free [S2, unofficial]. It iterates over all voxels and, if a voxel is not `VOXEL_EMPTY`, sets its value to `VOXEL_FREE` [S2, unofficial]. It is used to reset voxel states for re-processing or to clear fragment metadata without altering the empty/occupied status [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the total number of voxels in the grid, as every voxel must be visited to evaluate the predicate [S1, unofficial].
- **Memory Complexity:** $O(K)$, where $K$ is the number of voxels satisfying the predicate, for the output list [S1, unofficial].
- **Strengths:** Simple to implement; flexible via the predicate function; efficient for sparse results [S1, unofficial].
- **Limitations:** Requires full grid traversal; does not optimize for spatial locality unless the predicate is spatially coherent [G].
- **When to use:** When a specific subset of voxels needs to be identified for further processing, such as rendering or analysis [S1, unofficial].
- **When not to use:** When the grid is extremely large and the predicate is expensive, as the full traversal cost may be prohibitive [G].

## Background
Voxel Filtering is a fundamental operation in voxel-based rendering pipelines, often used in conjunction with densification and pruning steps in 3D Gaussian Splatting variants [S1, unofficial]. The specific implementation cited iterates over voxels to support dynamic scene updates and optimization [S1, unofficial]. The Grid Homogenization variant is used in voxel fragment processing to reset states [S2, unofficial].

## Sources
[S1] 3DGS-FLoD/flod, https://github.com/3DGS-FLoD/flod, fetched 2026-10-27 (unofficial)
[S2] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
