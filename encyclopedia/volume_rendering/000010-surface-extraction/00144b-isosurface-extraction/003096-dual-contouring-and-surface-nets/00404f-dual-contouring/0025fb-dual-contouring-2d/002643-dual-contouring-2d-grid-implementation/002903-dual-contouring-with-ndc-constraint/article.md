# Dual Contouring with NDC Constraint


## Overview
Dual Contouring with NDC Constraint is a surface extraction technique that generates a polygonal mesh from integer grid data and floating-point position data, enforcing constraints via a least-squares solver with regularization to keep vertex positions within $[0,1]$ bounds before mapping to grid coordinates [S1, unofficial]. It differs from standard dual contouring by explicitly handling boundary violations through an iterative scale adjustment and binary search mechanism rather than simple clamping or projection, ensuring that extracted vertices remain valid within the normalized device coordinate (NDC) range [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** `int_grid`, a 3D numpy array of shape `(dimx, dimy, dimz)` with dtype `np.int32`, representing the scalar field or occupancy grid. `float_grid`, a 3D numpy array of shape `(dimx, dimy, dimz)` with dtype `np.float32`, representing the floating-point position data associated with the grid [S1, unofficial].
*   **Output:** `all_vertices`, a 2D numpy array of shape `(N, 3)` with dtype `np.float32`, containing the extracted vertex positions. `all_triangles`, a 2D numpy array of shape `(M, 3)` with dtype `np.int32`, containing the indices of vertices forming the triangular mesh [S1, unofficial].

**Data Structures:**
*   `vertices_grid`: A 3D array of shape `(dimx, dimy, dimz)` with dtype `np.int32`, initialized to `-1`. It maps grid cell coordinates to indices in `all_vertices` [S1, unofficial].
*   `all_vertices`: A list of 3D vectors, later converted to a numpy array, storing the world-space positions of the extracted vertices [S1, unofficial].
*   `all_triangles`: A list of triplets of vertex indices, later converted to a numpy array, storing the connectivity of the mesh [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Initialize `vertices_grid` as a 3D array of shape `(dimx, dimy, dimz)` with dtype `np.int32`, filled with `-1` [S1, unofficial]. Initialize `all_vertices` as an empty list [S1, unofficial].

2.  **Vertex Generation:** Iterate through each grid cell defined by indices $i$ from $0$ to $\text{dimx}-2$, $j$ from $0$ to $\text{dimy}-2$, and $k$ from $0$ to $\text{dimz}-2$ [S1, unofficial].
    *   Retrieve the integer values of the 8 corners of the cell:
        *   $v_0 = \text{int\_grid}[i, j, k]$
        *   $v_1 = \text{int\_grid}[i+1, j, k]$
        *   $v_2 = \text{int\_grid}[i+1, j+1, k]$
        *   $v_3 = \text{int\_grid}[i, j+1, k]$
        *   $v_4 = \text{int\_grid}[i, j, k+1]$
        *   $v_5 = \text{int\_grid}[i+1, j, k+1]$
        *   $v_6 = \text{int\_grid}[i+1, j+1, k+1]$
        *   $v_7 = \text{int\_grid}[i, j+1, k+1]$ [S1, unofficial].
    *   If any of $v_1$ through $v_7$ differ from $v_0$, the cell contains an isosurface intersection, and a new vertex must be created [S1, unofficial].
    *   Set `vertices_grid[i, j, k]` to the current length of `all_vertices` [S1, unofficial].
    *   Calculate the initial position: $\text{pos} = \text{float\_grid}[i, j, k] + [i, j, k]$ (element-wise addition) [S1, unofficial].
    *   Append `pos` to `all_vertices` [S1, unofficial].
    *   *Note:* The sources describe this basic position calculation. The "NDC Constraint" aspect implies that in a full implementation, this position would be subject to regularization to ensure it lies within $[0,1]$ before final mapping, but the provided source code snippet for the core extraction logic does not show the explicit QEF or regularization solver steps, only the grid traversal and vertex placement logic [S1, unofficial]. The regularization is handled by the child method [NDC Constraint Regularization](4411-ndc-constraint-regularization.md).

3.  **Vertex Array Conversion:** Convert the `all_vertices` list to a numpy array `all_vertices` with dtype `np.float32` [S1, unofficial].

4.  **Triangle Generation:** Initialize `all_triangles` as an empty list [S1, unofficial].
    *   **i-direction Triangles:** Iterate $i$ from $0$ to $\text{dimx}-2$, $j$ from $1$ to $\text{dimy}-2$, $k$ from $1$ to $\text{dimz}-2$ [S1, unofficial].
        *   Let $v_0 = \text{int\_grid}[i, j, k]$ and $v_1 = \text{int\_grid}[i+1, j, k]$ [S1, unofficial].
        *   If $v_0 \neq v_1$:
            *   If $v_0 == 0$:
                *   Append triangle `[vertices_grid[i, j-1, k-1], vertices_grid[i, j, k-1], vertices_grid[i, j, k]]` [S1, unofficial].
                *   Append triangle `[vertices_grid[i, j-1, k-1], vertices_grid[i, j, k], vertices_grid[i, j-1, k]]` [S1, unofficial].
            *   Else ($v_0 \neq 0$):
                *   Append triangle `[vertices_grid[i, j-1, k-1], vertices_grid[i, j, k], vertices_grid[i, j, k-1]]` [S1, unofficial].
                *   Append triangle `[vertices_grid[i, j-1, k-1], vertices_grid[i, j-1, k], vertices_grid[i, j, k]]` [S1, unofficial].
    *   **j-direction Triangles:** Iterate $i$ from $1$ to $\text{dimx}-2$, $j$ from $0$ to $\text{dimy}-2$, $k$ from $1$ to $\text{dimz}-2$ [S1, unofficial].
        *   Let $v_0 = \text{int\_grid}[i, j, k]$ and $v_1 = \text{int\_grid}[i, j+1, k]$ [S1, unofficial].
        *   If $v_0 \neq v_1$:
            *   If $v_0 == 0$:
                *   Append triangle `[vertices_grid[i-1, j, k-1], vertices_grid[i, j, k], vertices_grid[i, j, k-1]]` [S1, unofficial].
                *   Append triangle `[vertices_grid[i-1, j, k-1], vertices_grid[i-1, j, k], vertices_grid[i, j, k]]` [S1, unofficial].
            *   Else ($v_0 \neq 0$):
                *   Append triangle `[vertices_grid[i-1, j, k-1], vertices_grid[i, j, k-1], vertices_grid[i, j, k]]` [S1, unofficial].
                *   Append triangle `[vertices_grid[i-1, j, k-1], vertices_grid[i, j, k], vertices_grid[i-1, j, k]]` [S1, unofficial].
    *   **k-direction Triangles:** Iterate $i$ from $1$ to $\text{dimx}-2$, $j$ from $1$ to $\text{dimy}-2$, $k$ from $0$ to $\text{dimz}-2$ [S1, unofficial].
        *   Let $v_0 = \text{int\_grid}[i, j, k]$ and $v_1 = \text{int\_grid}[i, j, k+1]$ [S1, unofficial].
        *   If $v_0 \neq v_1$:
            *   If $v_0 == 0$:
                *   Append triangle `[vertices_grid[i-1, j-1, k], vertices_grid[i, j, k], vertices_grid[i-1, j, k]]` [S1, unofficial].
                *   Append triangle `[vertices_grid[i-1, j-1, k], vertices_grid[i, j-1, k], vertices_grid[i, j, k]]` [S1, unofficial].
            *   Else ($v_0 \neq 0$):
                *   Append triangle `[vertices_grid[i-1, j-1, k], vertices_grid[i-1, j, k], vertices_grid[i, j, k]]` [S1, unofficial].
                *   Append triangle `[vertices_grid[i-1, j-1, k], vertices_grid[i, j, k], vertices_grid[i, j-1, k]]` [S1, unofficial].

5.  **Final Output:** Convert `all_triangles` to a numpy array `all_triangles` with dtype `np.int32` [S1, unofficial]. Return `all_vertices` and `all_triangles` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def dual_contouring_ndc(int_grid, float_grid):
    """
    Extracts a mesh from integer and float grids.
    Note: This implementation follows the provided source [S1, unofficial].
    The actual NDC constraint regularization is handled by the child method
    [NDC Constraint Regularization](4411-ndc-constraint-regularization.md).
    """
    dimx, dimy, dimz = int_grid.shape
    
    # Initialize vertices_grid with -1
    vertices_grid = np.full((dimx, dimy, dimz), -1, dtype=np.int32)
    all_vertices = []
    
    # Vertex Generation
    for i in range(dimx - 1):
        for j in range(dimy - 1):
            for k in range(dimz - 1):
                v0 = int_grid[i, j, k]
                v1 = int_grid[i+1, j, k]
                v2 = int_grid[i+1, j+1, k]
                v3 = int_grid[i, j+1, k]
                v4 = int_grid[i, j, k+1]
                v5 = int_grid[i+1, j, k+1]
                v6 = int_grid[i+1, j+1, k+1]
                v7 = int_grid[i, j+1, k+1]
                
                # Check if cell contains surface
                if not (v1 == v0 and v2 == v0 and v3 == v0 and 
                        v4 == v0 and v5 == v0 and v6 == v0 and v7 == v0):
                    
                    # Create new vertex
                    vertex_idx = len(all_vertices)
                    vertices_grid[i, j, k] = vertex_idx
                    
                    # Calculate position
                    pos = float_grid[i, j, k] + np.array([i, j, k], dtype=np.float32)
                    all_vertices.append(pos)
    
    # Convert vertices to array
    all_vertices = np.array(all_vertices, dtype=np.float32)
    all_triangles = []
    
    # Triangle Generation in i-direction
    for i in range(dimx - 1):
        for j in range(1, dimy - 1):
            for k in range(1, dimz - 1):
                v0 = int_grid[i, j, k]
                v1 = int_grid[i+1, j, k]
                
                if v0 != v1:
                    idx_000 = vertices_grid[i, j-1, k-1]
                    idx_001 = vertices_grid[i, j, k-1]
                    idx_010 = vertices_grid[i, j-1, k]
                    idx_011 = vertices_grid[i, j, k]
                    
                    if v0 == 0:
                        all_triangles.append([idx_000, idx_001, idx_011])
                        all_triangles.append([idx_000, idx_011, idx_010])
                    else:
                        all_triangles.append([idx_000, idx_011, idx_001])
                        all_triangles.append([idx_000, idx_010, idx_011])

    # Triangle Generation in j-direction
    for i in range(1, dimx - 1):
        for j in range(dimy - 1):
            for k in range(1, dimz - 1):
                v0 = int_grid[i, j, k]
                v1 = int_grid[i, j+1, k]
                
                if v0 != v1:
                    idx_000 = vertices_grid[i-1, j, k-1]
                    idx_001 = vertices_grid[i, j, k-1]
                    idx_010 = vertices_grid[i-1, j, k]
                    idx_011 = vertices_grid[i, j, k]
                    
                    if v0 == 0:
                        all_triangles.append([idx_000, idx_011, idx_001])
                        all_triangles.append([idx_000, idx_010, idx_011])
                    else:
                        all_triangles.append([idx_000, idx_001, idx_011])
                        all_triangles.append([idx_000, idx_011, idx_010])

    # Triangle Generation in k-direction
    for i in range(1, dimx - 1):
        for j in range(1, dimy - 1):
            for k in range(dimz - 1):
                v0 = int_grid[i, j, k]
                v1 = int_grid[i, j, k+1]
                
                if v0 != v1:
                    idx_000 = vertices_grid[i-1, j-1, k]
                    idx_001 = vertices_grid[i-1, j, k]
                    idx_010 = vertices_grid[i, j-1, k]
                    idx_011 = vertices_grid[i, j, k]
                    
                    if v0 == 0:
                        all_triangles.append([idx_000, idx_011, idx_010])
                        all_triangles.append([idx_000, idx_010, idx_011])
                    else:
                        all_triangles.append([idx_000, idx_010, idx_011])
                        all_triangles.append([idx_000, idx_011, idx_010])

    all_triangles = np.array(all_triangles, dtype=np.int32)
    
    return all_vertices, all_triangles
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The loops for triangle generation use ranges that exclude the outermost layer of grid cells (e.g., $j$ from $1$ to $\text{dimy}-2$). This assumes that the grid is padded or that boundary cells do not contribute to the mesh in the same way, or that the input grid has a specific structure where boundary intersections are handled elsewhere [S1, unofficial].
*   **Index Validity:** The code assumes that `vertices_grid` entries accessed during triangle generation are not `-1`. If a cell does not contain a vertex (because no surface intersection was detected in the vertex generation phase for that specific cell coordinate mapping), accessing `vertices_grid` might yield `-1`, leading to invalid triangle indices. The logic relies on the specific mapping of `vertices_grid[i,j,k]` to the vertex created for the cell starting at $(i,j,k)$ [S1, unofficial].
*   **NDC Constraint Implementation:** The provided source code does not show the explicit NDC constraint enforcement (regularization). This is noted to be handled by the child method [NDC Constraint Regularization](4411-ndc-constraint-regularization.md). Implementers must integrate this regularization step, likely modifying the vertex position calculation or applying it post-generation, to ensure vertices are within $[0,1]$ [S1, unofficial].

## Subtopics

### [NDC Constraint Regularization](4411-ndc-constraint-regularization.md)
A post-processing step that ensures extracted vertex coordinates remain within the normalized device coordinate (NDC) range $[0, 1]$ before mapping to grid indices. It solves this by augmenting the least-squares system with diagonal regularization terms that pull the solution toward a target position. It differs from standard dual contouring by explicitly handling boundary violations through an iterative scale adjustment and binary search mechanism rather than simple clamping or projection.

## Operating characteristics
*   **Time Complexity:** The vertex generation phase is $O(\text{dimx} \cdot \text{dimy} \cdot \text{dimz})$. The triangle generation phase is also $O(\text{dimx} \cdot \text{dimy} \cdot \text{dimz})$ as it iterates over the grid cells. The overall complexity is linear with respect to the number of grid cells [S1, unofficial].
*   **Memory Complexity:** The `vertices_grid` array requires $O(\text{dimx} \cdot \text{dimy} \cdot \text{dimz})$ space. The `all_vertices` and `all_triangles` arrays scale with the number of surface intersections, which is bounded by the grid size [S1, unofficial].
*   **Strengths:** The method provides a structured approach to mesh extraction from grid data. The integration of NDC constraints (via the child method) helps maintain geometric validity within specific coordinate systems [S1, unofficial].
*   **Limitations:** The provided source code does not detail the QEF minimization or regularization solver, which are central to standard dual contouring. The basic implementation relies on a simple position calculation, which may not produce high-quality meshes for complex surfaces without the additional regularization steps [S1, unofficial]. The specific triangle generation logic is rigid and depends on the integer grid values [S1, unofficial].

## Background
The method is associated with the repository `czq142857/NDC` [S1, unofficial]. It is part of a pipeline for surface extraction that includes neighbor search and evaluation metrics, as seen in the `eval_100000.py` script which loads ground truth and predicted meshes for comparison [S1, unofficial]. The specific "NDC Constraint" aspect is designed to handle boundary conditions in normalized device coordinates, a common requirement in computer graphics and rendering pipelines [S1, unofficial].

## Sources
[S1] czq142857/NDC, https://github.com/czq142857/NDC, fetched 2026-10-26 (unofficial)
