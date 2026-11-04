# Dual Grid Traversal

Also called: Dual Grid, Primal-Dual Grid Mapping

## Overview
Dual Grid Traversal constructs a secondary grid structure (the dual grid) from a primary voxel grid (the primal grid) to facilitate isosurface extraction or topological analysis. It solves the problem of mapping discrete voxel data to a continuous vertex-based representation by placing a vertex at the center of each primal cube. It sets itself apart from standard marching cubes by operating on the dual lattice, where each dual cell spans a $3 \times 3 \times 3$ region of the primal grid, ensuring that every intercepted edge of the isosurface is processed exactly once [S1, unofficial].

## How it works
**Inputs:** A primal grid of size $N \times N \times N$ (denoted as `size`).
**Outputs:** A `DualGrid` structure containing the primal grid reference and a `duals` array of vertices and associated data.

**Data Structures:**
The `DualGrid<D>` struct contains the following fields:
1.  `size`: `usize`, the dimension of the primal grid ($N$).
2.  `primal_grid`: `PrimalGrid<D>`, the source voxel data.
3.  `duals`: An array of two vectors, `[Vec<(Vec3, D)>; 2]`. Each vector has length $(N-1) \times (N-1)$. The elements are tuples of a 3D position (`Vec3`) and a data value (`D`).

**Algorithm Steps:**
1.  Initialize the `DualGrid` with the input `size`.
2.  Calculate `size_minus_one` as $N - 1$.
3.  Initialize the `primal_grid` with the input data of size $N \times N \times N$.
4.  Initialize the `duals` array. Create two vectors, each of length `size_minus_one * size_minus_one`.
5.  Fill both vectors with default values: `(Vec3::zero(), D::zero())`.

**Parameters:**
*   `size`: The dimension of the primal grid. The dual grid dimension is $(N-1)$ along each axis [S1, unofficial].

**Edge Cases and Pitfalls:**
*   If `size` is 1, `size_minus_one` is 0, resulting in empty dual vectors.
*   The mapping assumes a specific alignment where dual vertices correspond to the centers of primal cubes.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def init_dual_grid(size: int):
    """
    Initializes a DualGrid structure for a primal grid of dimension 'size'.
    """
    # Calculate dimensions
    size_minus_one = size - 1
    
    # Initialize primal grid (assumed to be populated externally or with zeros)
    # PrimalGrid<D> is a 3D array of size [size][size][size]
    primal_grid = PrimalGrid(size) 
    
    # Initialize duals array: two vectors of length (size-1)*(size-1)
    # Each element is a tuple (position: Vec3, data: D)
    num_duals = size_minus_one * size_minus_one
    
    # Vector 1: Positions
    duals_pos = [Vec3(0.0, 0.0, 0.0)] * num_duals
    
    # Vector 2: Data values
    duals_data = [D(0)] * num_duals
    
    duals = [duals_pos, duals_data]
    
    return DualGrid(size, primal_grid, duals)

class DualGrid:
    def __init__(self, size, primal_grid, duals):
        self.size = size
        self.primal_grid = primal_grid
        self.duals = duals

class Vec3:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

class D:
    def __init__(self, val):
        self.val = val

class PrimalGrid:
    def __init__(self, size):
        self.size = size
        # Placeholder for actual voxel data storage
        self.data = [[[D(0) for _ in range(size)] for _ in range(size)] for _ in range(size)]
```

## Subtopics

### [Dual Grid Traversal for Adaptive Octree](4043-dual-grid-traversal-for-adaptive-octree.md)
Traverses an adaptive octree to generate a dual grid representation by identifying all dual cells that straddle boundaries between octree nodes of varying depths. It uses a recursive face and edge traversal strategy with parent nodes acting as stand-ins for missing children to maintain connectivity across different levels of refinement. Choose this when working with hierarchical structures where topological consistency is required across varying refinement levels.

## Operating characteristics
*   **Time Complexity:** Initialization is $O(N^2)$ for the dual vectors, where $N$ is the grid dimension. The traversal cost depends on the specific isosurface extraction algorithm applied to the dual grid.
*   **Memory Complexity:** The dual grid stores $2 \times (N-1)^2$ elements. This is significantly less than the primal grid's $N^3$ voxels for large $N$, but the structure is specific to 2D slices or specific projection methods as implied by the $(N-1)^2$ size rather than $(N-1)^3$.
*   **Strengths:** Ensures topological consistency by processing every intercepted edge exactly once.
*   **Limitations:** The source notes describe a 2D-like dual structure ($N-1 \times N-1$) for a 3D primal grid, which may imply a slice-based or specific projection context rather than a full 3D dual volume. The exact 3D extension is not detailed in the provided sources.

## Background
The method originates from implementations related to isosurface generation, specifically cited in the context of the `isosurface` repository by swiftcoder. It is used in graphics applications to map voxel data to mesh vertices.

## Sources
[S1] isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
