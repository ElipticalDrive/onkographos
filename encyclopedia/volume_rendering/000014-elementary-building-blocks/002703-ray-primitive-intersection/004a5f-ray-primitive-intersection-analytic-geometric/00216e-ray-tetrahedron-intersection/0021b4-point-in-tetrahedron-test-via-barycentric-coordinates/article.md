# Point-in-Tetrahedron Test via Barycentric Coordinates

Also called: Barycentric Point-in-Tetrahedron Test

## Overview
This method determines whether a set of query points lies inside a set of tetrahedra by solving for barycentric coordinates [S1, unofficial]. It serves as the core geometric primitive for [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md), allowing volume rendering engines to identify which cells contain sample points for density accumulation [G]. Unlike [Plücker-based Tetrahedral Cell Intersection](1767-pl-cker-based-tetrahedral-cell-intersection.md) which operates on ray primitives, this method operates on explicit point positions, making it suitable for post-intersection validation or direct point-in-volume queries [G].

## How it works
### Inputs and Outputs
**Inputs:**
- `points`: Tensor of shape $(P, 3)$ containing $P$ query points in 3D space [S1, unofficial].
- `vertices`: Tensor of shape $(V, 3)$ containing the vertices of the tetrahedral mesh [S1, unofficial].
- `tetrahedra`: Tensor of shape $(T, 4)$ containing indices into `vertices` defining $T$ tetrahedra [S1, unofficial].

**Outputs:**
- `inside_tet`: Tensor of shape $(P,)$ storing the index of the first tetrahedron containing each point, or $-1$ if no containing tetrahedron exists [S1, unofficial].
- `barycentric_coords`: Tensor of shape $(P, 4)$ storing the barycentric weights $(\lambda_0, \lambda_1, \lambda_2, \lambda_3)$ for each point within its containing tetrahedron, or zeros if outside [S1, unofficial].

### Data Structures
The implementation relies on dense tensor layouts optimized for batched linear algebra operations [G].
- `mat`: A $(T, 3, 3)$ tensor representing the system of linear equations for a chunk of points, where columns are the edge vectors [S1, unofficial].
- `d`: A $(K, T, 3)$ tensor representing the vector from the reference vertex to the query points in the current chunk [S1, unofficial].

### Algorithm Steps
1. **Initialization**: Initialize `inside_tet` with $-1$ and `barycentric_coords` with $0$ [S1, unofficial].
2. **Chunking**: To manage memory efficiency, process points in chunks of size `chunk_size` [S1, unofficial]. Iterate `start` from $0$ to $P$ with step `chunk_size` [S1, unofficial]. Let `end = min(start + chunk_size, P)` [S1, unofficial].
3. **Vertex Extraction**: For the current chunk, extract the vertices of all tetrahedra [S1, unofficial]. Let $v_0, v_1, v_2, v_3$ be the vertices of a specific tetrahedron $t$ [S1, unofficial].
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
8. **Selection**: For each point, identify the first tetrahedron satisfying the inside condition [S1, unofficial]. If no tetrahedron contains the point, the result is $-1$ [S1, unofficial].

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
    tet_vertices = [[vertices[tetrahedra[t][v]] for v in range(4)] for t in range(len(tetrahedra))]
    
    for start in range(0, P, chunk_size):
        end = min(start + chunk_size, P)
        chunk_points = points[start:end]
        K = len(chunk_points)
        
        # Extract vertices v0..v3 for all T: shape (T, 3)
        v0 = [row[0] for row in tet_vertices]
        v1 = [row[1] for row in tet_vertices]
        v2 = [row[2] for row in tet_vertices]
        v3 = [row[3] for row in tet_vertices]
        
        # Edge vectors: shape (T, 3)
        a = [[v0[t][i] - v3[t][i] for i in range(3)] for t in range(len(tet_vertices))]
        b = [[v1[t][i] - v3[t][i] for i in range(3)] for t in range(len(tet_vertices))]
        c = [[v2[t][i] - v3[t][i] for i in range(3)] for t in range(len(tet_vertices))]
        
        # Matrix M: shape (T, 3, 3) where columns are a, b, c
        M = [[[a[t][i][j] for i in range(3)] for j in range(3)] for t in range(len(tet_vertices))]
        
        # Point vectors relative to v3: shape (K, T, 3)
        d = [[[chunk_points[k][i] - v3[t][i] for i in range(3)] for t in range(len(tet_vertices))] for k in range(K)]
        
        # Transpose d to (T, 3, K) to match solve signature
        d_T = [[[d[k][t][i] for k in range(K)] for i in range(3)] for t in range(len(tet_vertices))]
        
        # Solve M @ lambda = d_T for each T using standard linear algebra routine
        # sol shape: (T, 3, K)
        sol = solve_linear_system(M, d_T)
        
        # Transpose sol to (K, T, 3)
        chunk_bary = [[[sol[t][i][k] for i in range(3)] for t in range(len(tet_vertices))] for k in range(K)]
        
        # Extract lambdas
        l0 = [[chunk_bary[k][t][0] for t in range(len(tet_vertices))] for k in range(K)]
        l1 = [[chunk_bary[k][t][1] for t in range(len(tet_vertices))] for k in range(K)]
        l2 = [[chunk_bary[k][t][2] for t in range(len(tet_vertices))] for k in range(K)]
        l3 = [[1.0 - l0[k][t] - l1[k][t] - l2[k][t] for t in range(len(tet_vertices))] for k in range(K)]
        
        # Inside condition: shape (K, T)
        inside = [[(l0[k][t] >= 0) and (l1[k][t] >= 0) and (l2[k][t] >= 0) and (l3[k][t] >= 0) for t in range(len(tet_vertices))] for k in range(K)]
        
        # Find first containing tetrahedron for each point
        valid_exists = [any(inside[k]) for k in range(K)]
        tet_idx = []
        for k in range(K):
            if valid_exists[k]:
                # argmax on float representation of boolean array
                scores = [1.0 if inside[k][t] else 0.0 for t in range(len(tet_vertices))]
                tet_idx.append(scores.index(max(scores)))
            else:
                tet_idx.append(-1)
        
        # Update outputs for valid points
        valid_mask = [k for k in range(K) if valid_exists[k]]
        for k in valid_mask:
            global_idx = start + k
            t = tet_idx[k]
            inside_tet[global_idx] = t
            barycentric_coords[global_idx][0:3] = [l0[k][t], l1[k][t], l2[k][t]]
            
    return inside_tet, barycentric_coords
```

### Parameters
- `chunk_size`: Controls memory usage during the batched solve. Not given in the sources.

### Edge Cases and Pitfalls
- **Degenerate Tetrahedra**: If the volume of the tetrahedron is zero (coplanar vertices), the matrix $\mathbf{M}$ is singular and the solve will fail or produce NaNs [G]. Implementers should check for zero volume or use a pseudo-inverse with a tolerance threshold [G].
- **Precision**: Barycentric coordinates sum to 1. Floating-point errors may cause $\sum \lambda_i \ne 1$ exactly [G]. It is recommended to renormalize $\lambda_3 = 1 - \lambda_0 - \lambda_1 - \lambda_2$ after solving for the first three, as done in the algorithm, to ensure the constraint is satisfied [G].
- **Boundary Conditions**: Points exactly on the face or edge have $\lambda_i = 0$ [G]. The condition $\lambda_i \ge 0$ includes boundaries [G].

## Operating characteristics
- **Time Complexity**: $O(P \cdot T)$ for a naive implementation checking all tetrahedra, where $P$ is the number of points and $T$ is the number of tetrahedra [G]. With spatial acceleration (e.g., BVH), this reduces to $O(P \log T)$ or better [G]. The linear algebra solve is $O(T \cdot K)$ per chunk for $K$ points [G].
- **Memory Complexity**: $O(T \cdot K)$ for the intermediate matrices in the chunked solve [G].
- **Strengths**: Simple to implement, numerically stable for non-degenerate tetrahedra, easily vectorizable [G].
- **Limitations**: Naive search is slow for large $T$. Requires a spatial accelerator for performance in dense volumes [G].

## Background
This method is a standard geometric technique used in computer graphics and scientific visualization [G]. It is employed in [TetWeave](https://github.com/AlexandreBinninger/TetWeave) for resampling and interpolation in 3D reconstruction pipelines [S1, unofficial].

## Sources
[S1] AlexandreBinninger/TetWeave, https://github.com/AlexandreBinninger/TetWeave, fetched 2026-10-27 (unofficial)
