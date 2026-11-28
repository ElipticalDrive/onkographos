# Recursive Parameter Space Voxelization for Catmull-Clark Subdivision Surfaces

Also called: CCSS Voxelization, Voxelization of Solids Represented by CCSSs

## Overview
This method converts a free-form object represented by a Catmull-Clark subdivision surface (CCSS) into a voxel grid by recursively subdividing the 2D parameter space of surface patches and sampling 3D points, rather than subdividing 3D object space [S1]. It ensures leak-free results for closed objects when a 3D flooding operation is performed by guaranteeing that voxels generated from a single patch satisfy specific N-adjacency properties [S1]. Unlike 3D space subdivision techniques which are computationally expensive due to distance computations between 3D points, this method performs subdivision in 2D parameter space, avoiding expensive 3D distance computations [S1].

## How it works
**Inputs:**
*   A control mesh $S$ defining a Catmull-Clark subdivision surface [S1].
*   Output voxel grid resolution $M_1 \times M_2 \times M_3$ [S1].
*   Adjacency requirement $N \in \{6, 18, 26\}$, specifying the connectivity of voxels generated from a subpatch (6 for face-sharing, 18 for edge/face-sharing, 26 for vertex/edge/face-sharing) [S1].
*   Initial subdivision level $k$, an integer specifying that the parameter space $[0,1] \times [0,1]$ is divided into $k \times k$ subpatches before recursive processing begins [S1].

**Outputs:**
*   A voxel grid of resolution $M_1 \times M_2 \times M_3$ containing the voxelized representation of the CCSS [S1].

**Data Structures:**
*   The control mesh $S$ is stored as a collection of patches [S1].
*   Each patch `pid` is associated with a local 2D parameter domain $[0,1] \times [0,1]$ [S1].
*   Parametrization techniques allow the explicit calculation of any 3D point $\mathbf{S}(u,v)$, its position, normal, and partial derivatives on the limit surface given a patch ID and parameter coordinates $(u,v)$ [S1].

**Steps:**

1.  **Normalization:** Normalize the control mesh $S$ so that its bounding box aligns with the voxel grid coordinates $[0, M_1-1] \times [0, M_2-1] \times [0, M_3-1]$ [S1].
2.  **Initialization:** Iterate through each patch `pid` in the mesh $S$ [S1].
3.  **Initial Grid:** For the parameter space of the patch, iterate $u$ from $1/k$ to $1$ with step $1/k$, and $v$ from $1/k$ to $1$ with step $1/k$ [S1].
4.  **Recursive Processing:** For each initial subpatch defined by parameter ranges $[u-1/k, u]$ and $[v-1/k, v]$, call `VoxelizeSubPatch(N, pid, u - 1/k, u, v - 1/k, v)` [S1].
5.  **Subpatch Evaluation:** Inside `VoxelizeSubPatch`, evaluate the 3D surface positions for the four corners of the current subpatch defined by $(u_1, v_1)$, $(u_2, v_1)$, $(u_2, v_2)$, and $(u_1, v_2)$ using the patch ID `pid` and the parametrization function $\mathbf{S}(u,v)$ [S1]. Let these points be $\mathbf{V}_1, \mathbf{V}_2, \mathbf{V}_3, \mathbf{V}_4$ [S1].
6.  **Voxel Conversion:** Convert the 3D surface coordinates to voxel indices $(i,j,k)$ using the formula:
    $$ i = \lfloor \mathbf{S}(u, v).x + 0.5 \rfloor, \quad j = \lfloor \mathbf{S}(u, v).y + 0.5 \rfloor, \quad k = \lfloor \mathbf{S}(u, v).z + 0.5 \rfloor $$
    where $\lfloor \cdot \rfloor$ denotes the floor function, effectively rounding to the nearest integer [S1]. Mark these voxels in the output grid [S1].
7.  **Termination Check:**
    *   If the subpatch size is smaller than a voxel, return [S1].
    *   Calculate the maximum differences in voxel indices among the four corners:
        $$ \Delta i = \max_{a,b \in \{1,2,3,4\}} |i_a - i_b| $$
        $$ \Delta j = \max_{a,b \in \{1,2,3,4\}} |j_a - j_b| $$
        $$ \Delta k = \max_{a,b \in \{1,2,3,4\}} |k_a - k_b| $$
    *   If $N=6$ and $\Delta i + \Delta j + \Delta k \le 1$, return (subpatch is small enough for 6-adjacency) [S1].
    *   If $N=18$ and $\Delta i \le 1$, $\Delta j \le 1$, $\Delta k \le 1$, and $\Delta i + \Delta j + \Delta k \le 2$, return (subpatch is small enough for 18-adjacency) [S1].
    *   If $N=26$ and $\Delta i \le 1$, $\Delta j \le 1$, $\Delta k \le 1$, return (subpatch is small enough for 26-adjacency) [S1].
8.  **Recursive Subdivision:** If the subpatch is not small enough, compute midpoints $u_{12} = (u_1 + u_2)/2$ and $v_{12} = (v_1 + v_2)/2$ [S1]. Recursively call `VoxelizeSubPatch` for the four quadrants:
    *   $[u_1, u_{12}] \times [v_1, v_{12}]$
    *   $[u_{12}, u_2] \times [v_1, v_{12}]$
    *   $[u_{12}, u_2] \times [v_{12}, v_2]$
    *   $[u_1, u_{12}] \times [v_{12}, v_2]$ [S1]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def voxelize_ccss(control_mesh, N, M1, M2, M3, k):
    """
    Voxelizes a Catmull-Clark Subdivision Surface.
    
    Args:
        control_mesh: List of patches, each patch has an ID and parametrization function.
        N: Adjacency requirement (6, 18, or 26).
        M1, M2, M3: Resolution of the output voxel grid.
        k: Initial subdivision level for parameter space.
        
    Returns:
        3D boolean array of shape (M1, M2, M3).
    """
    # Initialize voxel grid
    voxel_grid = [[[False for _ in range(M3)] for _ in range(M2)] for _ in range(M1)]
    
    # [G] Normalize control mesh bounding box to [0, M1-1] x [0, M2-1] x [0, M3-1].
    normalized_mesh = control_mesh  # Placeholder for normalization step
    
    # Iterate through each patch
    for patch in normalized_mesh:
        pid = patch.id
        # Initial parameter space subdivision
        step = 1.0 / k
        for i in range(1, k + 1):
            u = i * step
            for j in range(1, k + 1):
                v = j * step
                # Call recursive voxelization for the initial subpatch
                voxelize_subpatch(pid, u - step, u, v - step, v, N, voxel_grid)
            
    return voxel_grid

def voxelize_subpatch(pid, u1, u2, v1, v2, N, voxel_grid):
    """
    Recursively voxelizes a subpatch of a CCSS.
    """
    # [G] Evaluate surface point S(u,v) for patch pid using parametrization techniques [S1].
    v1_pt = (u1, v1)  # Placeholder for 3D point evaluation
    v2_pt = (u2, v1)
    v3_pt = (u2, v2)
    v4_pt = (u1, v2)
    
    # Convert to voxel indices
    i1 = math.floor(v1_pt[0] + 0.5)
    j1 = math.floor(v1_pt[1] + 0.5)
    k1 = math.floor(v1_pt[2] + 0.5)
    
    i2 = math.floor(v2_pt[0] + 0.5)
    j2 = math.floor(v2_pt[1] + 0.5)
    k2 = math.floor(v2_pt[2] + 0.5)
    
    i3 = math.floor(v3_pt[0] + 0.5)
    j3 = math.floor(v3_pt[1] + 0.5)
    k3 = math.floor(v3_pt[2] + 0.5)
    
    i4 = math.floor(v4_pt[0] + 0.5)
    j4 = math.floor(v4_pt[1] + 0.5)
    k4 = math.floor(v4_pt[2] + 0.5)
    
    # Mark voxels in the grid
    if 0 <= i1 < M1 and 0 <= j1 < M2 and 0 <= k1 < M3: voxel_grid[i1][j1][k1] = True
    if 0 <= i2 < M1 and 0 <= j2 < M2 and 0 <= k2 < M3: voxel_grid[i2][j2][k2] = True
    if 0 <= i3 < M1 and 0 <= j3 < M2 and 0 <= k3 < M3: voxel_grid[i3][j3][k3] = True
    if 0 <= i4 < M1 and 0 <= j4 < M2 and 0 <= k4 < M3: voxel_grid[i4][j4][k4] = True
    
    # Check if subpatch is small enough
    # [G] Check if subpatch bounding box in 3D is smaller than a voxel; if so, return.
    
    # Calculate max differences in voxel indices
    indices_i = [i1, i2, i3, i4]
    indices_j = [j1, j2, j3, j4]
    indices_k = [k1, k2, k3, k4]
    
    delta_i = max(indices_i) - min(indices_i)
    delta_j = max(indices_j) - min(indices_j)
    delta_k = max(indices_k) - min(indices_k)
    
    # Termination conditions based on N
    if N == 6 and (delta_i + delta_j + delta_k) <= 1:
        return
    if N == 18 and delta_i <= 1 and delta_j <= 1 and delta_k <= 1 and (delta_i + delta_j + delta_k) <= 2:
        return
    if N == 26 and delta_i <= 1 and delta_j <= 1 and delta_k <= 1:
        return
        
    # Recursive subdivision
    u12 = (u1 + u2) / 2.0
    v12 = (v1 + v2) / 2.0
    
    voxelize_subpatch(pid, u1, u12, v1, v12, N, voxel_grid)
    voxelize_subpatch(pid, u12, u2, v1, v12, N, voxel_grid)
    voxelize_subpatch(pid, u12, u2, v12, v2, N, voxel_grid)
    voxelize_subpatch(pid, u1, u12, v12, v2, N, voxel_grid)
```

**Edge Cases and Pitfalls:**
*   **Holes:** Holes appear if the voxels generated from the four corners of a subpatch are not N-adjacent [S1]. The algorithm prevents this by recursively subdividing until the N-adjacency condition is met [S1].
*   **Shared Boundaries:** Shared boundaries between patches or subpatches must map to the same voxel to ensure leak-free results [S1]. The algorithm guarantees this because shared boundary points are evaluated at the same parameter coordinates [S1].
*   **Precision:** The voxel coordinate conversion uses `floor(S(u,v).x + 0.5)` [S1]. Floating-point precision errors in surface evaluation could theoretically cause boundary voxels to be assigned incorrectly, but the recursive subdivision ensures that adjacent patches share evaluation points, mitigating gaps [S1].

## Operating characteristics
*   **Time Complexity:** The method avoids the exponential increase in faces associated with uniformly refining the control mesh, making it more efficient than direct sampling on subdivided surfaces [S1]. The complexity depends on the curvature of the surface and the chosen adjacency $N$; flatter regions terminate earlier, while highly curved regions require deeper recursion [S1].
*   **Memory Complexity:** The output is a voxel grid of size $M_1 \times M_2 \times M_3$ [S1]. The recursion depth is limited by the resolution and curvature, but the memory overhead for the recursion stack is generally low compared to storing a highly subdivided mesh [S1].
*   **Strengths:**
    *   Guarantees leak-free voxelization for closed objects when followed by 3D flooding [S1].
    *   Works directly on parametric surfaces of arbitrary topology without requiring explicit polygonization first [S1].
    *   Avoids expensive 3D distance computations inherent in 3D space subdivision techniques [S1].
*   **Limitations:**
    *   Requires efficient parametrization techniques to explicitly calculate surface points [S1].
    *   Performance is sensitive to the initial subdivision level $k$ and adjacency $N$; higher $N$ or smaller $k$ may increase computation time [S1].
*   **When to use:** Volume graphics applications involving free-form solids with arbitrary topology represented by Catmull-Clark subdivision surfaces [S1].

## Background
*   **Origin:** The method was presented in a paper titled "Voxelization of Free-Form Solids Represented by Catmull-Clark Subdivision Surfaces" [S1].
*   **History:** Subdivision surfaces have become popular in graphical modeling, visualization, and animation due to their capability in modeling complex shapes of arbitrary topology and high visual quality [S1]. Traditional 3D scan-conversion methods extend 2D scan-line algorithms, but this method works directly on parametric surfaces [S1]. Recent parametrization techniques allow explicit calculation of points on the limit surface, enabling this efficient voxelization approach [S1].
*   **Usage:** Used for volume graphics applications [S1].

## Sources
[S1] Voxelization of Free-Form Solids Represented by Catmull ..., https://cs.uky.edu/~cheng/PUBL/voxel_gmp06_final.pdf, fetched 2026-10-29
