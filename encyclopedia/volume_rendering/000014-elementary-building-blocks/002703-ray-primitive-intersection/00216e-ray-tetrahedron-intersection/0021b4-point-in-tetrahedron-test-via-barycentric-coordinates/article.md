# Point-in-Tetrahedron Test via Barycentric Coordinates

Also called: Barycentric Point-in-Tetrahedron Test

## Overview
This method determines whether a set of query points lies inside a set of tetrahedra by solving for barycentric coordinates. It serves as the geometric core for the [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md) algorithm, specifically handling the containment check after a ray has potentially intersected the tetrahedral volume. Unlike sibling methods that may rely on edge-based or Plücker coordinate formulations, this approach uses linear algebra to solve for the four barycentric weights directly, allowing for efficient vectorized processing of point clouds.

## How it works
### Inputs and Outputs
**Inputs:**
- `points`: A tensor of shape $(P, 3)$ containing $P$ query points in 3D space.
- `vertices`: A tensor of shape $(V, 3)$ containing the 3D coordinates of the mesh vertices.
- `tetrahedra`: A tensor of shape $(T, 4)$ containing indices into `vertices`, defining $T$ tetrahedra. Each row $[i_0, i_1, i_2, i_3]$ refers to vertices $v_0, v_1, v_2, v_3$.

**Outputs:**
- `inside_tet`: A tensor of shape $(P,)$ where each element is the index of the first tetrahedron containing the corresponding point, or $-1$ if no containing tetrahedron is found.
- `barycentric_coords`: A tensor of shape $(P, 4)$ containing the barycentric weights $[\lambda_0, \lambda_1, \lambda_2, \lambda_3]$ for each point relative to its containing tetrahedron. If a point is not inside any tetrahedron, these values are zero.

### Data Structures
The implementation processes points in chunks to manage memory usage.
- `chunk_size`: The number of points processed in a single batch.
- `mat`: A temporary tensor of shape $(3, 3, \text{chunk\_size})$ representing the edge vectors of the tetrahedra.
- `d`: A temporary tensor of shape $(3, \text{chunk\_size})$ representing the vector from the reference vertex to the query points.

### Algorithm Steps
1.  **Initialization**: Initialize `inside_tet` with $-1$ and `barycentric_coords` with $0$.
2.  **Chunking**: Iterate over the points in chunks of size `chunk_size`. For each chunk, determine the start and end indices.
3.  **Vertex Retrieval**: Retrieve the vertices for all tetrahedra using the `tetrahedra` index tensor. Let the vertices for a specific tetrahedron be $v_0, v_1, v_2, v_3$.
4.  **Edge Vector Computation**: Compute the edge vectors originating from $v_3$:
    $$ a = v_0 - v_3 $$
    $$ b = v_1 - v_3 $$
    $$ c = v_2 - v_3 $$
5.  **Point Vector Computation**: Compute the vector from $v_3$ to each query point $p$ in the current chunk:
    $$ d = p - v_3 $$
6.  **Barycentric Solve**: The barycentric coordinates $\lambda_0, \lambda_1, \lambda_2$ (relative to edges $a, b, c$) satisfy the linear system:
    $$ \lambda_0 a + \lambda_1 b + \lambda_2 c = d $$
    This is solved by constructing the matrix $M = [a, b, c]$ and computing:
    $$ [\lambda_0, \lambda_1, \lambda_2]^T = M^{-1} d $$
    In practice, this is done using a linear solver (e.g., `torch.linalg.solve`).
7.  **Lambda Extraction**: Extract $\lambda_0, \lambda_1, \lambda_2$ from the solution. Compute the fourth barycentric coordinate:
    $$ \lambda_3 = 1 - \lambda_0 - \lambda_1 - \lambda_2 $$
8.  **Containment Check**: A point is inside the tetrahedron if and only if all barycentric coordinates are non-negative:
    $$ \text{inside} = (\lambda_0 \ge 0) \land (\lambda_1 \ge 0) \land (\lambda_2 \ge 0) \land (\lambda_3 \ge 0) $$
9.  **Selection**: For each point, if it is inside any tetrahedron, select the first one (lowest index). Update `inside_tet` and `barycentric_coords` for the valid points in the chunk.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_in_tetrahedron(points, vertices, tetrahedra, chunk_size=1024):
    """
    Determine if points lie inside tetrahedra using barycentric coordinates.
    
    Args:
        points: Tensor of shape (P, 3)
        vertices: Tensor of shape (V, 3)
        tetrahedra: Tensor of shape (T, 4)
        chunk_size: Integer, number of points to process at once
        
    Returns:
        inside_tet: Tensor of shape (P,) with tetrahedron index or -1
        barycentric_coords: Tensor of shape (P, 4) with barycentric weights
    """
    P = points.shape[0]
    inside_tet = torch.full((P,), -1, dtype=torch.int64)
    barycentric_coords = torch.zeros((P, 4), dtype=torch.float32)
    
    num_tets = tetrahedra.shape[0]
    # Pre-fetch all tetrahedron vertices to avoid repeated indexing
    # Shape: (T, 4, 3)
    tets_vertices = vertices[tetrahedra] 
    
    for start in range(0, P, chunk_size):
        end = min(start + chunk_size, P)
        chunk_points = points[start:end] # Shape: (chunk_size, 3)
        
        # Get vertices for all tetrahedra: (T, 4, 3)
        # Unpack into v0, v1, v2, v3: each (T, 3)
        v0, v1, v2, v3 = tets_vertices[:, 0], tets_vertices[:, 1], tets_vertices[:, 2], tets_vertices[:, 3]
        
        # Compute edge vectors: each (T, 3)
        a = v0 - v3
        b = v1 - v3
        c = v2 - v3
        
        # Compute point vectors relative to v3: (chunk_size, 3)
        # We need to broadcast v3 to match chunk_size for the subtraction
        # v3 is (T, 3), chunk_points is (chunk_size, 3)
        # We want d to be (T, chunk_size, 3) where d[t, p] = chunk_points[p] - v3[t]
        d = chunk_points.unsqueeze(0) - v3.unsqueeze(1) # Shape: (T, chunk_size, 3)
        
        # Stack edge vectors to form matrix M: (T, 3, 3)
        # M[t] = [a[t], b[t], c[t]]
        M = torch.stack([a, b, c], dim=1) # Shape: (T, 3, 3)
        
        # Solve M * lambda = d for each point
        # d is (T, chunk_size, 3), we need to solve for each point independently
        # Reshape d to (T * chunk_size, 3) for batched solve, or use einsum/solve appropriately
        # torch.linalg.solve expects (..., N, N) and (..., N, K)
        # We want to solve for each of the T tetrahedra for each of the chunk_size points
        
        # Reshape M to (T, 3, 3) and d to (T, chunk_size, 3)
        # We need to solve M[t] @ x = d[t, p] for each t, p
        # This can be done by reshaping to (T*chunk_size, 3, 3) and (T*chunk_size, 3, 1)
        
        M_flat = M.reshape(-1, 3, 3) # (T*chunk_size, 3, 3)
        d_flat = d.permute(0, 2, 1).reshape(-1, 3, 1) # (T*chunk_size, 3, 1)
        
        # Solve linear system
        sol = torch.linalg.solve(M_flat, d_flat) # (T*chunk_size, 3, 1)
        
        # Reshape back to (T, chunk_size, 3)
        sol = sol.squeeze(-1).permute(1, 0, 2) # (chunk_size, T, 3)
        
        # Extract lambdas: (chunk_size, T)
        lambda_0 = sol[:, :, 0]
        lambda_1 = sol[:, :, 1]
        lambda_2 = sol[:, :, 2]
        
        # Compute lambda_3
        lambda_3 = 1.0 - lambda_0 - lambda_1 - lambda_2
        
        # Check inside condition: (chunk_size, T)
        inside = (lambda_0 >= 0) & (lambda_1 >= 0) & (lambda_2 >= 0) & (lambda_3 >= 0)
        
        # Find first containing tetrahedron for each point
        # inside.any(dim=1) gives (chunk_size,) boolean mask for valid points
        valid_tet_exists = inside.any(dim=1)
        
        # Get indices of valid points in the chunk
        chunk_valid_mask = valid_tet_exists.nonzero(as_tuple=True)[0]
        
        if len(chunk_valid_mask) > 0:
            # For valid points, find the first tetrahedron index
            # inside.float() is (chunk_size, T)
            # argmax(dim=1) returns the first max index (which is the first True)
            chunk_inside_tet = torch.where(
                valid_tet_exists, 
                inside.float().argmax(dim=1), 
                torch.full((chunk_size,), -1, dtype=torch.float32)
            ).long()
            
            # Get the tetrahedron indices for valid points
            valid_tet_idx = chunk_inside_tet[chunk_valid_mask]
            
            # Update inside_tet
            inside_tet[start:end][chunk_valid_mask] = valid_tet_idx
            
            # Update barycentric_coords
            # We need the lambdas corresponding to the selected tetrahedra
            # valid_tet_idx is (num_valid,)
            # We need to gather lambda_0, lambda_1, lambda_2 for these indices
            
            # Gather indices for each lambda
            # lambda_0 shape: (chunk_size, T)
            # We want lambda_0[chunk_valid_mask, valid_tet_idx]
            
            # Create a grid of indices for gathering
            row_indices = torch.arange(len(chunk_valid_mask), device=inside.device)
            col_indices = valid_tet_idx
            
            # Gather values
            l0 = lambda_0[chunk_valid_mask, col_indices]
            l1 = lambda_1[chunk_valid_mask, col_indices]
            l2 = lambda_2[chunk_valid_mask, col_indices]
            l3 = lambda_3[chunk_valid_mask, col_indices]
            
            barycentric_coords[start:end, 0][chunk_valid_mask] = l0
            barycentric_coords[start:end, 1][chunk_valid_mask] = l1
            barycentric_coords[start:end, 2][chunk_valid_mask] = l2
            barycentric_coords[start:end, 3][chunk_valid_mask] = l3
            
    return inside_tet, barycentric_coords
```

### Parameters
- `chunk_size`: Controls memory usage. Larger values improve throughput but increase peak memory. Typical values range from 1024 to 8192 depending on available GPU memory [S1, unofficial].

### Edge Cases and Pitfalls
- **Degenerate Tetrahedra**: If the tetrahedron volume is zero (coplanar vertices), the matrix $M$ is singular and the linear solver will fail or produce NaNs. [G] Implementers should check for zero volume or use a pseudo-inverse if degenerate cases are expected.
- **Precision**: Barycentric coordinates should sum to 1.0. Floating-point errors may cause slight deviations. The condition $\lambda_i \ge 0$ is robust, but strict equality checks for $\sum \lambda_i = 1$ are not required if $\lambda_3$ is computed as $1 - \sum_{i=0}^2 \lambda_i$.
- **Boundary Conditions**: Points exactly on the boundary have $\lambda_i = 0$. The condition $\ge 0$ includes them as "inside".

## Subtopics

### [Plücker-based Tetrahedral Cell Intersection](1767-pl-cker-based-tetrahedral-cell-intersection.md)
Uses Plücker coordinates for intersection tests. Choose when working with line geometry primitives directly.

### [IRIS Ray-Geometry Intersection](3250-iris-ray-geometry-intersection.md)
A specific ray-intersection algorithm. Choose when using the IRIS framework.

### [Skeleton Parameter Minimization for Ray Intersection](3975-skeleton-parameter-minimization-for-ray-intersection.md)
Optimizes ray intersection via skeleton parameters. Choose for skeleton-based representations.

### [Ray-Cylinder Intersection](5014-ray-cylinder-intersection.md)
Intersects rays with cylinders. Choose for cylindrical primitives.

### [Mesh Shader Tetrahedron Intersection Optimization](5174-mesh-shader-tetrahedron-intersection-optimization.md)
Optimizes tetrahedron intersection on mesh shaders. Choose for GPU mesh shader pipelines.

### [Raytracing Cell-Organized Irregular Volumes](5257-raytracing-cell-organized-irregular-volumes.md)
Handles cell-organized irregular volumes. Choose for structured irregular grids.

## Operating characteristics
- **Time Complexity**: $O(P \cdot T)$ in the worst case if checking all tetrahedra, but typically $O(P)$ if spatial acceleration structures (like BVH or grid) are used to limit $T$ to a small constant. The linear solve is $O(1)$ per point per tetrahedron.
- **Memory Complexity**: $O(P + T)$ for storing inputs and outputs. Chunking reduces peak memory to $O(\text{chunk\_size} \cdot T)$.
- **Strengths**: Vectorizable, numerically stable for non-degenerate tetrahedra, easy to implement with standard linear algebra libraries.
- **Limitations**: Requires solving a linear system for each tetrahedron-point pair, which can be expensive without acceleration structures. Degenerate tetrahedra cause numerical instability.
- **When to Use**: When high precision is needed for point containment in tetrahedral meshes, and acceleration structures are available to limit the search space.
- **When Not to Use**: For degenerate tetrahedra or when memory is extremely constrained and chunking is not feasible.

## Background
This method is derived from the TetWeave project, which uses barycentric coordinates for resampling and optimization in tetrahedral grids [S1, unofficial]. It is a standard technique in computational geometry for point-in-polyhedron tests.

## Sources
[S1] AlexandreBinninger/TetWeave, https://github.com/AlexandreBinninger/TetWeave, fetched 2026-10-27 (unofficial)
