# Point-in-Tetrahedron Test via Barycentric Coordinates

Also called: Barycentric Point-in-Tetrahedron Test

## Overview
This method determines whether a set of query points lies inside a set of tetrahedra by solving for barycentric coordinates [S1, unofficial]. It serves as the core geometric primitive for [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md), allowing volume rendering engines to identify which cells contain sample points for density accumulation [G]. Unlike [Plücker-based Tetrahedral Cell Intersection](1767-pl-cker-based-tetrahedral-cell-intersection.md) which operates on ray primitives, this method operates on explicit point positions, making it suitable for post-intersection validation or direct point-in-volume queries [G].

## How it works
### Inputs and Outputs
**Inputs:**
- `points`: Tensor of shape $(P, 3)$ containing $P$ query points in 3D space.
- `vertices`: Tensor of shape $(V, 3)$ containing the vertices of the tetrahedral mesh.
- `tetrahedra`: Tensor of shape $(T, 4)$ containing indices into `vertices` defining $T$ tetrahedra.

**Outputs:**
- `inside_tet`: Tensor of shape $(P,)$ storing the index of the first tetrahedron containing each point, or $-1$ if no containing tetrahedron exists [S1, unofficial].
- `barycentric_coords`: Tensor of shape $(P, 4)$ storing the barycentric weights $(\lambda_0, \lambda_1, \lambda_2, \lambda_3)$ for each point within its containing tetrahedron, or zeros if outside [S1, unofficial].

### Data Structures
The implementation relies on dense tensor layouts optimized for batched linear algebra operations [G].
- `mat`: A $(T, 3, 3)$ tensor representing the system of linear equations for a chunk of points, where columns are the edge vectors.
- `d`: A $(T, K, 3)$ tensor representing the vector from the reference vertex to the query points in the current chunk.

### Algorithm Steps
1. **Initialization**: Initialize `inside_tet` with $-1$ and `barycentric_coords` with $0$ [S1, unofficial].
2. **Chunking**: To manage memory efficiency, process points in chunks of size `chunk_size` [S1, unofficial]. Iterate `start` from $0$ to $P$ with step `chunk_size`. Let `end = min(start + chunk_size, P)`.
3. **Vertex Extraction**: For the current chunk, extract the vertices of all tetrahedra. Let $v_0, v_1, v_2, v_3$ be the vertices of a specific tetrahedron $t$ [S1, unofficial].
4. **Edge Vector Computation**: Compute edge vectors relative to vertex $v_3$ (the reference vertex):
   $$ \mathbf{a} = \mathbf{v}_0 - \mathbf{v}_3 $$
   $$ \mathbf{b} = \mathbf{v}_1 - \mathbf{v}_3 $$
   $$ \mathbf{c} = \mathbf{v}_2 - \mathbf{v}_3 $$
5. **Point Vector Computation**: Compute the vector from $v_3$ to the query point $\mathbf{p}$:
   $$ \mathbf{d} = \mathbf{p} - \mathbf{v}_3 $$
6. **Barycentric Solving**: Solve the linear system $\mathbf{M} \boldsymbol{\lambda}_{0:2} = \mathbf{d}$ for the first three barycentric coordinates, where $\mathbf{M} = [\mathbf{a}, \mathbf{b}, \mathbf{c}]$ [S1, unofficial].
   $$ \begin{bmatrix} \lambda_0 \\ \lambda_1 \\ \lambda_2 \end{bmatrix} = \mathbf{M}^{-1} \mathbf{d} $$
   The fourth coordinate is derived by the constraint $\sum \lambda_i = 1$:
   $$ \lambda_3 = 1 - \lambda_0 - \lambda_1 - \lambda_2 $$
7. **Inside Test**: A point is inside the tetrahedron if all barycentric coordinates are non-negative:
   $$ \text{inside} = (\lambda_0 \ge 0) \land (\lambda_1 \ge 0) \land (\lambda_2 \ge 0) \land (\lambda_3 \ge 0) $$
8. **Selection**: For each point, identify the first tetrahedron satisfying the inside condition. If no tetrahedron contains the point, the result is $-1$ [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_in_tetrahedron(points, vertices, tetrahedra, chunk_size):
    """
    Determines if points lie inside tetrahedra using barycentric coordinates.
    
    Args:
        points: Array (P, 3)
        vertices: Array (V, 3)
        tetrahedra: Array (T, 4)
        chunk_size: Integer, number of points to process per batch
        
    Returns:
        inside_tet: Array (P,) index of containing tetrahedron or -1
        barycentric_coords: Array (P, 4) barycentric weights
    """
    P = len(points)
    inside_tet = [-1] * P
    barycentric_coords = [[0.0, 0.0, 0.0, 0.0] for _ in range(P)]
    
    # Pre-fetch tetrahedron vertices: shape (T, 4, 3)
    tet_vertices = [vertices[tetrahedra[t][v]] for t in range(len(tetrahedra)) for v in range(4)]
    tet_vertices = reshape(tet_vertices, (len(tetrahedra), 4, 3))
    
    for start in range(0, P, chunk_size):
        end = min(start + chunk_size, P)
        chunk_points = points[start:end]
        K = len(chunk_points)
        
        # Extract vertices v0..v3 for all T: shape (T, 3)
        v0 = tet_vertices[:, 0, :]
        v1 = tet_vertices[:, 1, :]
        v2 = tet_vertices[:, 2, :]
        v3 = tet_vertices[:, 3, :]
        
        # Edge vectors: shape (T, 3)
        a = v0 - v3
        b = v1 - v3
        c = v2 - v3
        
        # Matrix M: shape (T, 3, 3) where columns are a, b, c
        M = stack([a, b, c], axis=-1)
        
        # Point vectors relative to v3: shape (T, K, 3)
        d = chunk_points - v3[:, None, :]
        
        # Solve M @ lambda = d^T for each T, K
        # Transpose d to (T, 3, K) to match solve signature
        d_T = transpose(d, axes=(0, 2, 1))
        lambda_012 = solve_linear_system(M, d_T) # shape (T, 3, K)
        lambda_012 = transpose(lambda_012, axes=(0, 2, 1)) # shape (T, K, 3)
        
        # Extract lambdas
        lambda_0 = lambda_012[:, :, 0]
        lambda_1 = lambda_012[:, :, 1]
        lambda_2 = lambda_012[:, :, 2]
        lambda_3 = 1.0 - lambda_0 - lambda_1 - lambda_2
        
        # Inside condition: shape (T, K)
        inside = (lambda_0 >= 0) & (lambda_1 >= 0) & (lambda_2 >= 0) & (lambda_3 >= 0)
        
        # Find first containing tetrahedron for each point
        valid_tet_exists = any(inside, axis=0) # shape (K,)
        tet_indices = argmax(inside.astype(float), axis=0) # shape (K,)
        chunk_inside_tet = where(valid_tet_exists, tet_indices, -1)
        
        # Update outputs
        chunk_valid_mask = nonzero(valid_tet_exists)
        valid_tet_idx = chunk_inside_tet[chunk_valid_mask]
        
        # Gather barycentric coords for valid points
        gathered_lambdas = gather(all_lambdas, valid_tet_idx, chunk_valid_mask)
        
        for i, k in enumerate(chunk_valid_mask):
            global_idx = start + k
            inside_tet[global_idx] = chunk_inside_tet[k]
            barycentric_coords[global_idx] = gathered_lambdas[i]
            
    return inside_tet, barycentric_coords
```

### Parameters
- `chunk_size`: Controls memory usage during the batched solve. Not given in the sources.

### Edge Cases and Pitfalls
- **Degenerate Tetrahedra**: If the volume of the tetrahedron is zero (coplanar vertices), the matrix $\mathbf{M}$ is singular and the solve will fail or produce NaNs [G]. Implementers should check for zero volume or use a pseudo-inverse with a tolerance threshold.
- **Precision**: Barycentric coordinates sum to 1. Floating-point errors may cause $\sum \lambda_i \ne 1$ exactly [G]. It is recommended to renormalize $\lambda_3 = 1 - \lambda_0 - \lambda_1 - \lambda_2$ after solving for the first three, as done in the algorithm, to ensure the constraint is satisfied.
- **Boundary Conditions**: Points exactly on the face or edge have $\lambda_i = 0$ [G]. The condition $\lambda_i \ge 0$ includes boundaries [G].

## Operating characteristics
- **Time Complexity**: $O(P \cdot T)$ for a naive implementation checking all tetrahedra, where $P$ is the number of points and $T$ is the number of tetrahedra. With spatial acceleration (e.g., BVH), this reduces to $O(P \log T)$ or better [G]. The linear algebra solve is $O(T \cdot K)$ per chunk for $K$ points [G].
- **Memory Complexity**: $O(T \cdot K)$ for the intermediate matrices in the chunked solve [G].
- **Strengths**: Simple to implement, numerically stable for non-degenerate tetrahedra, easily vectorizable [G].
- **Limitations**: Naive search is slow for large $T$. Requires a spatial accelerator for performance in dense volumes [G].

## Background
This method is a standard geometric technique used in computer graphics and scientific visualization. It is employed in [TetWeave](https://github.com/AlexandreBinninger/TetWeave) for resampling and interpolation in 3D reconstruction pipelines [S1, unofficial].

## Sources
[S1] AlexandreBinninger/TetWeave, https://github.com/AlexandreBinninger/TetWeave, fetched 2026-10-27 (unofficial)
