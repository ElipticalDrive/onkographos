# Quadric Error Function (QEF) for Sharp Features

Also called: QEF Solver

## Overview
This method computes the optimal vertex position for sharp features (corners and edges) in dual contouring by minimizing the distance to multiple intersecting planes defined by sample points and their normals [S1, unofficial]. It distinguishes itself from standard dual contouring by explicitly handling underdetermined systems (sharp edges) via least-squares minimization, ensuring the resulting vertex lies equidistant from all contributing planes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $k$ intersection points $\{P_i\}$ and their corresponding surface normals $\{N_i\}$ within a voxel cell [S1, unofficial].
*   **Output:** A single 3D point $x$ representing the sharp feature [S1, unofficial].

**Data Structures:**
The method constructs a linear system $Ax = B$ (or $Ax \approx B$ for QEF) using the following layouts [S1, unofficial]:
*   `matrix_A`: A dense matrix of size $k \times 3$. Each row $i$ stores the normal vector components $(n_{ix}, n_{iy}, n_{iz})$ [S1, unofficial].
*   `vector_B`: A dense vector of size $k$. Each entry $i$ stores the dot product $N_i \cdot P_i$ [S1, unofficial].

**Steps:**
1.  **Collect Intersections:** Identify all grid edges intersecting the isosurface within the voxel and compute the point $P_i$ and normal $N_i$ at each intersection [G].
2.  **Populate Matrix A and Vector B:** For each intersection point $i$ from $0$ to $k-1$:
    *   Set `matrix_A[i][0] = N_i.x`
    *   Set `matrix_A[i][1] = N_i.y`
    *   Set `matrix_A[i][2] = N_i.z`
    *   Set `vector_B[i] = P_i.x * N_i.x + P_i.y * N_i.y + P_i.z * N_i.z` [S1, unofficial].
3.  **Solve via SVD:** Perform Singular Value Decomposition on $A$ to obtain $U, D, V^T$ such that $A = U D V^T$ [G]. Solve the system $U D V^T x = B$ for $x$ [S1, unofficial].

**Formulas:**
For a sharp corner (3 points), the exact solution satisfies:
$$ N_i \cdot (x - P_i) = 0 \quad \text{for } i=1,2,3 $$
Which expands to the linear system $Ax = B$ where:
$$ A = \begin{bmatrix} N_1^T \\ N_2^T \\ N_3^T \end{bmatrix}, \quad B = \begin{bmatrix} N_1 \cdot P_1 \\ N_2 \cdot P_2 \\ N_3 \cdot P_3 \end{bmatrix} $$
[S1, unofficial].

For a sharp edge (underdetermined system, e.g., 4 points defining 2 planes), the system is solved via QEF:
$$ E[x] = \|Ax - B\|^2 $$
The solution is found using SVD:
$$ x = V D^{-1} U^T B $$
*(Note: $D^{-1}$ involves inverting non-zero singular values; zero singular values are treated as zero to find the minimum norm solution) [S1, unofficial].*

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Degenerate Normals:** If normals are collinear or nearly parallel, the matrix $A$ becomes ill-conditioned [G].
*   **Underdetermined Systems:** Standard linear solvers fail for sharp edges (fewer independent constraints than dimensions) [S1, unofficial].
*   **Boundary Violations:** The unconstrained QEF solution may lie outside the voxel cell [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solve_qef_sharp_features(intersection_points, normals):
    """
    Solves for the optimal vertex position using QEF for sharp features.
    
    Args:
        intersection_points: List of 3D vectors [P0, P1, ..., Pk-1]
        normals: List of 3D vectors [N0, N1, ..., Nk-1]
        
    Returns:
        newPoint: 3D vector representing the sharp feature location
    """
    k = len(intersection_points)
    if k == 0:
        return None
        
    # Initialize matrix A (k x 3) and vector B (k)
    matrix_A = [[0.0, 0.0, 0.0] for _ in range(k)]
    vector_B = [0.0] * k
    
    # Populate A and B
    for i in range(k):
        P = intersection_points[i]
        N = normals[i]
        
        # Row i of A is the normal vector
        matrix_A[i][0] = N.x
        matrix_A[i][1] = N.y
        matrix_A[i][2] = N.z
        
        # Entry i of B is the dot product N . P
        vector_B[i] = P.x * N.x + P.y * N.y + P.z * N.z
        
    # Standard SVD routine [G]
    # A = U @ diag(D) @ Vt
    U, D, Vt = svd(matrix_A)
    
    # Construct D^-1 (pseudoinverse of diagonal matrix D)
    # Handle zero singular values to avoid division by zero
    D_inv = [[0.0] * 3 for _ in range(3)]
    for i in range(3):
        if D[i] > 1e-10:
            D_inv[i][i] = 1.0 / D[i]
        else:
            D_inv[i][i] = 0.0
            
    # Compute x = V @ D_inv @ U^T @ B
    # Step 1: U^T @ B
    UtB = [0.0] * 3
    for i in range(3):
        for j in range(k):
            UtB[i] += U[j][i] * vector_B[j]
            
    # Step 2: D_inv @ (U^T @ B)
    D_inv_UtB = [0.0] * 3
    for i in range(3):
        D_inv_UtB[i] = D_inv[i][i] * UtB[i]
        
    # Step 3: V @ (D_inv @ U^T @ B)
    newPoint = [0.0] * 3
    for i in range(3):
        for j in range(3):
            newPoint[i] += Vt[j][i] * D_inv_UtB[j]
            
    return newPoint
```

## Subtopics

### [QEF Boundary Constrained Vertex Placement](1740-qef-boundary-constrained-vertex-placement.md)
A fallback strategy for vertex placement when the QEF solution lies outside the voxel cell. It constrains the search space hierarchically (interior -> faces -> edges -> corners) to find a valid point within boundaries. Choose this when the unconstrained QEF result is invalid for mesh topology.

## Operating characteristics
*   **Time Complexity:** Dominated by SVD, typically $O(k^3)$ for small $k$ [G].
*   **Memory Complexity:** $O(k)$ for storing $A$ and $B$ [G].
*   **Strengths:** Robustly handles sharp features (edges/corners) that standard linear solvers cannot [G].
*   **Limitations:** Does not guarantee the result lies within the voxel cell [G].
*   **When to Use:** When extracting meshes from implicit surfaces with sharp features (e.g., boxes, cylinders) using dual contouring [G].
*   **When Not to Use:** For smooth surfaces where standard dual contouring or marching cubes suffices, or when strict boundary constraints are required without additional fallback logic [G].

## Background
The Quadric Error Function (QEF) for sharp features was introduced in the context of dual contouring for isosurface extraction [S1, unofficial]. The method was described in an unofficial source detailing the implementation of an isosurface polygonizer, specifically addressing the limitation of linear systems when dealing with sharp edges where the system is underdetermined [S1, unofficial]. It builds on the general dual contouring framework [Parent](1424-dual-contouring-qef-solver.md).

## Sources
[S1] http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, fetched 2026-10-30 (unofficial)
