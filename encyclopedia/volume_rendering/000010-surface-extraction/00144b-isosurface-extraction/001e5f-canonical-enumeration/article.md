# Canonical Enumeration

Also called: Linear Scan, Sequential Enumeration

## Overview
Canonical Enumeration is the fundamental baseline algorithm for isosurface cell extraction, which identifies grid cells intersecting an isosurface by sequentially visiting every cell in the volume [S1, unofficial]. It serves as the reference point for all other extraction methods, establishing the theoretical lower bound for worst-case time complexity based on isovalue comparisons [S1, unofficial]. Unlike hierarchical or adaptive methods, it performs no spatial pruning, ensuring that every cell is evaluated exactly once regardless of surface density.

## How it works
**Inputs and Outputs**
- **Input**: A scalar volume data structure $V$ defined on a regular grid, and a scalar isovalue $t$.
- **Output**: A list of "active" cells $C_{i,j,k}$ that contain the isosurface.

**Data Structures**
- **Volume Data**: A 3D array or equivalent structure storing scalar values $V_{i,j,k}$ at grid vertices or cell centers. The exact layout depends on the host implementation (e.g., row-major 3D array), but the algorithm only requires random access to vertex values within a cell.
- **Active Cell List**: A dynamic list or buffer storing the indices $(i,j,k)$ of cells where the isosurface is detected.

**Algorithm Steps**
1. Initialize an empty list `active_cells`.
2. Iterate through every cell $C_{i,j,k}$ in the volume grid. The traversal order is typically lexicographic (e.g., $i$, then $j$, then $k$), but any order covering all cells is valid.
3. For each cell, retrieve the scalar values at its vertices. Let $V_{i,j,k}$ denote the set of scalar values associated with the vertices of cell $C_{i,j,k}$.
4. Compute the minimum and maximum values within the cell:
   $$ v_{\min} = \min(V_{i,j,k}) $$
   $$ v_{\max} = \max(V_{i,j,k}) $$
5. Define the interpolation range interval for the cell as $[v_{\min}, v_{\max})$. The use of a half-open interval is critical to handle boundary cases consistently, particularly when multiple cells share vertices with values exactly equal to the isovalue [S1, unofficial].
6. Check if the isovalue $t$ lies within this interval:
   $$ t \in [v_{\min}, v_{\max}) $$
   This is equivalent to checking:
   $$ v_{\min} \le t < v_{\max} $$
7. If the condition is true, the cell is "active" and is added to `active_cells`. Otherwise, it is skipped.
8. After iterating through all cells, return `active_cells`.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def canonical_enumeration(volume, isovalue):
    """
    volume: 3D array of scalar values V[i][j][k]
    isovalue: float t
    Returns: list of (i, j, k) tuples for active cells
    """
    active_cells = []
    
    # Determine grid dimensions
    # Note: The exact iteration bounds depend on whether V stores vertex values
    # or cell values. For standard Marching Cubes, V is vertex data.
    # If V is vertex data of size (Nx+1)x(Ny+1)x(Nz+1), cells are Nx x Ny x Nz.
    # This implementation assumes V is vertex data.
    
    Nx, Ny, Nz = volume.shape[0] - 1, volume.shape[1] - 1, volume.shape[2] - 1
    
    for i in range(Nx):
        for j in range(Ny):
            for k in range(Nz):
                # Retrieve the 8 vertex values for cell (i, j, k)
                # Vertices are at (i,j,k), (i+1,j,k), ..., (i+1,j+1,k+1)
                v000 = volume[i][j][k]
                v100 = volume[i+1][j][k]
                v010 = volume[i][j+1][k]
                v110 = volume[i+1][j+1][k]
                v001 = volume[i][j][k+1]
                v101 = volume[i+1][j][k+1]
                v011 = volume[i][j+1][k+1]
                v111 = volume[i+1][j+1][k+1]
                
                vertex_values = [v000, v100, v010, v110, v001, v101, v011, v111]
                
                # Compute min and max
                vmin = min(vertex_values)
                vmax = max(vertex_values)
                
                # Check half-open interval [vmin, vmax)
                if vmin <= isovalue < vmax:
                    active_cells.append((i, j, k))
                    
    return active_cells
```

**Parameters and Edge Cases**
- **Isovalue $t$**: A real number.
- **Precision**: Floating-point comparisons should be handled carefully. The half-open interval $[v_{\min}, v_{\max})$ ensures that if $t = v_{\max}$ for one cell, it is not considered active in that cell, potentially deferring detection to a neighboring cell where $t$ might be the minimum or interior value, depending on the specific topology and shared vertex handling [S1, unofficial].
- **Degenerate Input**: If $v_{\min} = v_{\max}$ (constant value in a cell), the interval is empty or a single point. If $t = v_{\min} = v_{\max}$, the condition $t < v_{\max}$ fails, so the cell is not active. This is consistent with the half-open interval definition.

## Subtopics

### [Isosurface Propagation from Seed Cells](6942-isosurface-propagation-from-seed-cells.md)
A technique that combines geometric decomposition and span space aspects, starting extraction from a set of seed cells and propagating through neighboring cells.

## Operating characteristics
- **Time Complexity**: $O(n)$, where $n$ is the number of cells in the volume [S1, unofficial]. This is the lowest possible worst-case extraction time for any method based solely on isovalue comparisons without auxiliary spatial structures.
- **Memory Complexity**: $O(1)$ auxiliary space, excluding the output list and input volume storage. No additional data structures (like octrees or bounding volume hierarchies) are required [S1, unofficial].
- **Performance**: The method is simple to implement and has low overhead. However, it is inefficient for sparse surfaces where only a small fraction of cells are active, as it performs unnecessary checks on empty cells.
- **Strengths**: Predictable performance, minimal memory footprint, easy parallelization (each cell is independent).
- **Limitations**: No acceleration for large volumes with sparse isosurfaces. Performance does not scale with surface complexity.

## Background
Canonical Enumeration was introduced in the seminal "Marching Cubes" paper by Lorenson and Cline [S1, unofficial]. It represents the most direct approach to the cell extraction problem, forming the basis for many subsequent optimizations that introduce spatial hierarchies (e.g., octrees) to skip empty regions [S1, unofficial].

## Sources
[S1] Optimal hybrid memory constrained isosurface extraction, http://nbn-resolving.de/urn:nbn:de:bsz:352-130091, fetched 2026-10-30 (unofficial)
