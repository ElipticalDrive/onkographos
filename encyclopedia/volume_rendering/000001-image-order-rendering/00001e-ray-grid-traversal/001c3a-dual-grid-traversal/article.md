# Dual Grid Traversal
Also called: Dual Grid Traversal for Adaptive Octree (in the context of adaptive structures)
## Overview
Dual Grid Traversal is a method for generating topologically consistent isosurface representations (such as Marching Cubes) by operating on a "dual grid" rather than the primal voxel grid. It solves the problem of connectivity errors in adaptive hierarchical structures (like adaptive octrees) where adjacent nodes have different refinement levels. Unlike standard primal grid traversal which processes cells independently, this method traverses the dual grid vertices to ensure that every intercepted edge of the isosurface is processed exactly once, maintaining 4-connectivity (or 6/8-connectivity in 3D) across level boundaries [S1, unofficial].

## How it works
### Inputs and Outputs
- **Input:** A primal grid (or adaptive octree) of size $N \times N \times N$ (or hierarchical equivalent).
- **Output:** A set of dual grid vertices and associated data (e.g., normals, values) required for mesh generation.

### Data Structures
The core data structure is the `DualGrid<D>` struct [S1, unofficial]:
- `size`: `usize`, the dimension of the primal grid ($N$).
- `primal_grid`: `PrimalGrid<D>`, the underlying primal data.
- `duals`: An array of two vectors, `Vec<(Vec3, D)>`, each of length $(N-1) \times (N-1)$. This represents the dual vertices. Each entry stores a position (`Vec3`) and a value (`D`).

### Algorithm Steps
1. **Initialization:**
   - Calculate `size_minus_one = size - 1`.
   - Initialize the `duals` array with two vectors, each of length `size_minus_one * size_minus_one`, filled with zero values (`Vec3::zero(), D::zero()`) [S1, unofficial].
   - Initialize the `primal_grid` with the given `size` [S1, unofficial].

2. **Traversal Logic:**
   - The traversal iterates over the primal grid cells.
   - For each cell, it identifies the dual vertices (centers of primal edges/faces/cells depending on the specific dual definition) that straddle the boundaries.
   - In adaptive octrees, parent nodes act as stand-ins for missing children to maintain connectivity across different levels of refinement [S1, unofficial].
   - A recursive face and edge traversal strategy is used to ensure topological consistency [S1, unofficial].

3. **Dual Grid Construction:**
   - A dual grid is formed by placing a vertex in the center of each cube in the primal grid, and connecting adjacent vertices into cubes.
   - Each $2 \times 2 \times 2$ cube in the dual grid spans a $3 \times 3 \times 3$ region in the primal grid [S1, unofficial].

### Parameters
- `size`: The dimension of the primal grid ($N_x \times N_y \times N_z$). The dual grid will have dimension $(N-1)$ along each axis [S1, unofficial].

### Edge Cases and Pitfalls
- **Adaptive Level Mismatches:** When adjacent octree nodes have different depths, standard traversal may miss edges or create gaps. The algorithm must use parent nodes as proxies for missing children to ensure every intercepted edge is processed exactly once [S1, unofficial].
- **Boundary Conditions:** The dual grid size is $(N-1)$, so boundary checks must account for the reduced dimensionality.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode based on [S1, unofficial]
# Note: The source provides a high-level struct and initialization, 
# but the detailed recursive traversal logic for adaptive octrees is described textually.

struct DualGrid(D):
    size: usize
    primal_grid: PrimalGrid<D>
    duals: [Vec<(Vec3, D)>; 2]

def initialize_dual_grid(size: usize):
    size_minus_one = size - 1
    # Initialize duals with two vectors of length (size-1)*(size-1)
    # Filled with zero values
    duals = [
        [(Vec3(0,0,0), D(0))] * (size_minus_one * size_minus_one),
        [(Vec3(0,0,0), D(0))] * (size_minus_one * size_minus_one)
    ]
    primal_grid = PrimalGrid<D>(size)
    return DualGrid(size, primal_grid, duals)

def traverse_dual_grid(dual_grid: DualGrid[D]):
    # Recursive face and edge traversal strategy is used.
    # Parent nodes act as stand-ins for missing children in adaptive structures.
    # This ensures topological consistency.
    # Specific implementation details for the recursive step are not provided in the source.
    pass
```

## Subtopics

### [Dual Grid Traversal for Adaptive Octree](4043-dual-grid-traversal-for-adaptive-octree.md)
This method traverses an adaptive octree to generate a dual grid representation by identifying all dual cells (vertices) that straddle the boundaries between octree nodes of varying depths. It solves the problem of topological consistency in adaptive structures where adjacent nodes have different refinement levels, ensuring that every intercepted edge of the isosurface is processed exactly once. Unlike sibling methods that may process cells independently, this algorithm uses a recursive face and edge traversal strategy with parent nodes acting as stand-ins for missing children to maintain connectivity across different levels of refinement.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** The dual grid requires storage for $(N-1)^2$ vertices per vector, with two vectors. For a 3D grid, the memory usage scales with the surface area of the dual grid, which is $O(N^2)$ for the dual vertices if stored as a 2D array of vectors, or potentially $O(N^3)$ if the dual grid is fully 3D (though the source specifies a 2D vector structure for `duals`, which is ambiguous for 3D traversal). The source specifies `duals` as `[Vec<(Vec3, D)>; 2]` with length `(size-1)*(size-1)`, which suggests a 2D representation or a specific flattening for a 3D problem.
- **Strengths:** Ensures topological consistency in adaptive hierarchical structures.
- **Limitations:** Requires careful handling of level mismatches.
- **When to use:** When generating isosurfaces from adaptive octrees or hierarchical grids where connectivity across different refinement levels is critical.

## Background
The method is associated with the `isosurface` project by SwiftCoder [S1, unofficial]. It is used in the context of generating isosurfaces from volumetric data, particularly when using adaptive octrees.

## Sources
[S1] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
