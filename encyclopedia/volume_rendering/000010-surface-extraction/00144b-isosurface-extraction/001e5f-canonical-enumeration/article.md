# Canonical Enumeration

Also called: Linear Scan, Sequential Enumeration

## Overview
Canonical Enumeration is the elementary method for cell extraction that sequentially visits all cells in a discretized volume to identify active cells containing the isosurface [S1, unofficial]. It solves the cell extraction problem by checking if a target isovalue lies within the scalar range of each cell [S1, unofficial]. Unlike [Isosurface Propagation from Seed Cells](6942-isosurface-propagation-from-seed-cells.md), which skips empty regions by traversing a hierarchical search structure of seeds, Canonical Enumeration visits every cell regardless of occupancy [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume data structure $V$ defined on a regular grid, where each cell $C_{i,j,k}$ has scalar values $V_{i,j,k}$ at its vertices, and a scalar isovalue $t \in \mathbb{R}$ [S1, unofficial].
*   **Output:** A list of active cells $C_{i,j,k}$ where the isosurface intersects the cell [S1, unofficial].

**Data Structures:**
*   **Volume Data:** A 3D array or linearized buffer storing vertex scalar values. No auxiliary structures are required [S1, unofficial].
*   **Variables:** Temporary storage for $v_{min}$, $v_{max}$, and loop indices [S1, unofficial].

**Steps:**
1. Iterate through every cell $C_{i,j,k}$ in the volume grid [S1, unofficial].
2. For each cell, determine the minimum scalar value $v_{min}$ and maximum scalar value $v_{max}$ among its vertices [S1, unofficial].
3. Evaluate the condition $t \in [v_{min}, v_{max})$ [S1, unofficial].
4. If the condition is true, mark $C_{i,j,k}$ as an active cell; otherwise, skip it [S1, unofficial].

**Formulas:**
The range of the interpolation function $\hat{f}$ in a cell is defined as:
$$ \hat{f}(C_{i,j,k}) = [v_{min}, v_{max}] $$
where:
$$ v_{min} = \min(V_{i,j,k}) $$
$$ v_{max} = \max(V_{i,j,k}) $$
[S1, unofficial]

The cell extraction condition is:
$$ t \in [v_{min}, v_{max}) $$
[S1, unofficial]

**Parameters:**
*   **Isovalue ($t$):** The threshold scalar value defining the surface [S1, unofficial].
*   **Grid Dimensions:** $N_x, N_y, N_z$ defining the total number of cells $n$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Interval Boundary:** The method uses the half-open interval $[v_{min}, v_{max})$ rather than the closed interval $[v_{min}, v_{max}]$ to force consistent reporting of boundary conditions [S1, unofficial].
*   **Precision:** Floating-point comparisons for $t \ge v_{min}$ and $t < v_{max}$ must handle precision errors near the boundaries [G].
*   **Degenerate Input:** If $v_{min} = v_{max}$, the interval collapses to a single point; the half-open rule still applies [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def canonical_enumeration(volume_data, isovalue, grid_dims):
    """
    volume_data: 3D array of shape (Nx+1, Ny+1, Nz+1) storing scalar values at vertices.
    isovalue: The scalar threshold t.
    grid_dims: Tuple (Nx, Ny, Nz) defining the number of cells.
    Returns: List of active cell indices (i, j, k).
    """
    active_cells = []
    Nx, Ny, Nz = grid_dims
    
    for k in range(Nz):
        for j in range(Ny):
            for i in range(Nx):
                # Extract the 8 vertex values for cell (i, j, k)
                cell_vertices = [
                    volume_data[i, j, k], volume_data[i+1, j, k],
                    volume_data[i, j+1, k], volume_data[i+1, j+1, k],
                    volume_data[i, j, k+1], volume_data[i+1, j, k+1],
                    volume_data[i, j+1, k+1], volume_data[i+1, j+1, k+1]
                ]
                
                # Calculate min and max scalar values in the cell
                vmin = min(cell_vertices)
                vmax = max(cell_vertices)
                
                # Check if isovalue is in the half-open interval [vmin, vmax)
                if vmin <= isovalue < vmax:
                    active_cells.append((i, j, k))
                    
    return active_cells
```

## Subtopics

### [Isosurface Propagation from Seed Cells](6942-isosurface-propagation-from-seed-cells.md)
A geometric decomposition technique that identifies sparse "seed" cells and propagates extraction to neighbors, skipping empty regions. Choose this when the volume is sparse or large, as it offers near-optimal worst-case time complexity by avoiding full enumeration.

## Operating characteristics
*   **Time Complexity:** $O(n)$, where $n$ is the total number of cells in the volume data [S1, unofficial]. This is the lowest possible worst-case extraction time for any method based solely on isovalue comparisons without auxiliary spatial structures [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space. It requires only storage for the original volume data and a small constant amount for variables [S1, unofficial].
*   **Strengths:** Simplicity of implementation; no preprocessing or auxiliary data structure construction required [G].
*   **Limitations:** Inefficient for sparse volumes or large datasets where the surface occupies a tiny fraction of the volume, as it processes every cell regardless of whether it contains the surface [G].
*   **When to use:** Small volumes, dense surfaces, or as a baseline for correctness verification [G].
*   **When not to use:** Large-scale volumes with sparse surfaces where acceleration via hierarchical subdivision (e.g., octrees) is necessary [G].

## Background
Canonical Enumeration is the elementary method for cell extraction, historically used in the original "Marching Cubes" paper by Lorenson and Cline [S1, unofficial].

## Sources
[S1] Optimal hybrid memory constrained isosurface extraction, http://nbn-resolving.de/urn:nbn:de:bsz:352-130091, fetched 2026-10-30 (unofficial)
