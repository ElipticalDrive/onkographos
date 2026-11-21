# Dual Contouring QEF Solver

Also called: Dual Contouring QEF Solver Implementation, QEF Solver for Dual Contouring, QEF Solving

## Overview
The Dual Contouring QEF Solver is the core vertex placement mechanism in Dual Contouring, a surface extraction algorithm that generates a mesh by placing a single vertex inside each cell containing an isosurface intersection [S1, unofficial]. It solves the problem of determining the optimal position for this dual vertex by minimizing a Quadratic Error Function (QEF) based on surface normals and intersection points at the cell boundaries [S1, unofficial]. Unlike standard Dual Marching Cubes which rounds corners, this method preserves sharp features (creases, apexes) by placing vertices directly on sharp creases via QEF SVD [S1, unofficial]. It sets itself apart from its siblings by using a Jacobi $3 \times 3$ symmetric eigensolver SVD with regularized truncation to ensure watertight manifold topology and numerical stability [S1, unofficial].

## How it works
**Inputs:**
*   `grid_vertices`: Array of voxel grid corner coordinates [S1, unofficial].
*   `voxels`: Sparse voxel grid data structure containing SDF values and potentially pre-computed normals [S1, unofficial].
*   `sdf`: Implicit function or SDF values associated with the grid [S1, unofficial].
*   `iso`: Isosurface level (typically 0.0) [S1, unofficial].
*   `quad_split`: Boolean flag to toggle between pure quad output and Delaunay triangulated triangle output [S1, unofficial].
*   `pad`: Boundary padding for sparse voxel grids (typically 1) to ensure all incident voxels are present in memory [S1, unofficial].
*   `grid_normals`: Optional array of exact Hermite/triangle normals to avoid trilinear gradient smoothing [S1, unofficial].

**Outputs:**
*   `verts`: Array of dual vertex positions [S1, unofficial].
*   `faces`: Array of face indices (triangles if `quad_split=True`, quads if `quad_split=False`) [S1, unofficial].

**Data Structures:**
*   **Plane Constraint:** A plane is defined by a point $p_i$ on the isosurface and a normal $n_i$ [S1, unofficial]. The distance from a candidate vertex $v$ to the plane is $d_i(v) = n_i \cdot (v - p_i)$ [G].
*   **QEF Matrix:** The QEF minimizes $E(v) = \sum_i (n_i \cdot (v - p_i))^2$ [S1, unofficial]. This is equivalent to minimizing $\|Av - b\|^2$, where $A$ is a matrix with rows $n_i^T$ and $b$ is a vector with entries $n_i \cdot p_i$ [G].
*   **Lookup Table:** `dc_edge_quadrant[12]` is a static compile-time lookup table mapping the 12 sign-crossing edges of a voxel to quadrilateral slots, replacing floating-point polar angle sorting [S1, unofficial].

**Steps:**

1.  **Identify Sign-Crossing Edges:** For each voxel cell, identify the 12 edges where the SDF sign changes (i.e., $SDF(v_{start}) \cdot SDF(v_{end}) < 0$) [S1, unofficial].
2.  **Collect Plane Constraints:** For each sign-crossing edge, compute the intersection point $p_i$ (linearly interpolated along the edge) and the surface normal $n_i$ [S1, unofficial]. If `grid_normals` are provided, use them; otherwise, compute gradients from the SDF [S1, unofficial].
3.  **Assemble QEF System:** Construct the symmetric positive semi-definite matrix $M = A^T A$ and vector $b_{vec} = A^T b$ from the collected constraints [S1, unofficial].
4.  **Solve QEF via Regularized SVD:**
    *   Perform Singular Value Decomposition (SVD) on $A$ or eigen-decomposition on $M$ using a Jacobi $3 \times 3$ symmetric eigensolver [S1, unofficial].
    *   **Regularization:** Apply truncation to singular values $\sigma_i < 0.01 \sigma_{\max}$ to handle nearly coplanar normals [S1, unofficial]. Use mass-centroid regularization to prevent the dual vertex from drifting outside the voxel boundary [S1, unofficial].
    *   The solution $v$ is the eigenvector corresponding to the smallest singular value, adjusted by the regularization term [S1, unofficial].
5.  **Generate Topology (Deterministic Cartesian Quadrant Slotting):**
    *   Map the 12 sign-crossing edges to quadrilateral slots using the static lookup table `dc_edge_quadrant[12]` [S1, unofficial]. This eliminates self-intersecting quads caused by coincident dual vertices on sharp features [S1, unofficial].
    *   If `quad_split=False`, output the resulting quadrilaterals directly [S1, unofficial].
    *   If `quad_split=True`, apply Delaunay max-min angle splitting to convert quadrilaterals into triangles [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def solve_dual_contouring_qef(voxels, grid_vertices, sdf, iso=0.0, quad_split=True, pad=1, grid_normals=None):
    """
    Solves the Dual Contouring QEF for a sparse voxel grid.
    """
    verts = []
    faces = []
    
    for cell in voxels.active_cells:
        edge_indices = []
        edge_points = []
        edge_normals = []
        
        for edge_id in range(12):
            v0, v1 = cell.get_edge_vertices(edge_id)
            s0 = sdf(v0)
            s1 = sdf(v1)
            
            if s0 * s1 < 0:
                edge_indices.append(edge_id)
                t = s0 / (s0 - s1)
                p = v0 + t * (v1 - v0)
                edge_points.append(p)
                
                if grid_normals is not None:
                    n = grid_normals.get_edge_normal(edge_id)
                else:
                    # Compute gradient via central finite differences [G]
                    h = 1e-4
                    dx = sdf(v0 + [h, 0, 0]) - sdf(v0 - [h, 0, 0])
                    dy = sdf(v0 + [0, h, 0]) - sdf(v0 - [0, h, 0])
                    dz = sdf(v0 + [0, 0, h]) - sdf(v0 - [0, 0, h])
                    n = [dx, dy, dz]
                
                # Safe normalization [G]
                norm_len = np.sqrt(n[0]**2 + n[1]**2 + n[2]**2)
                if norm_len > 1e-8:
                    n = [n[0]/norm_len, n[1]/norm_len, n[2]/norm_len]
                else:
                    n = [0, 0, 1]
                edge_normals.append(n)
        
        if not edge_indices:
            continue
            
        A = np.array(edge_normals)
        b = np.array([n[0]*p[0] + n[1]*p[1] + n[2]*p[2] for n, p in zip(edge_normals, edge_points)])
        
        M = A.T @ A
        rhs = A.T @ b
        
        v = solve_jacobi_qef(M, rhs, epsilon=0.01)
        v = np.clip(v, cell.min_bound, cell.max_bound)
        verts.append(v)
        
        quad_slots = map_edges_to_quads(edge_indices)
        
        if quad_split:
            tri_faces = split_quads_to_tris(quad_slots)
            faces.extend(tri_faces)
        else:
            faces.extend(quad_slots)
            
    return np.array(verts), np.array(faces)

def solve_jacobi_qef(M, rhs, epsilon=0.01):
    """
    Solves Mv = rhs with regularized Jacobi eigensolver [G].
    """
    # Initialize eigenvalues and eigenvectors
    eigenvalues = np.diag(M).copy()
    eigenvectors = np.eye(3)
    
    # Jacobi iteration for 3x3 symmetric matrix [G]
    for _ in range(50):
        # Find largest off-diagonal element
        max_val = 0.0
        p, q = 0, 1
        for i in range(3):
            for j in range(i+1, 3):
                if abs(M[i, j]) > max_val:
                    max_val = abs(M[i, j])
                    p, q = i, j
        
        if max_val < 1e-10:
            break
            
        # Compute rotation angle
        tau = (eigenvalues[q] - eigenvalues[p]) / (2.0 * M[p, q])
        t = (1.0 / abs(tau) + np.sqrt(1.0 + tau**2)) * np.sign(tau)
        c = 1.0 / np.sqrt(1.0 + t**2)
        s = t * c
        
        # Apply rotation
        M_new = M.copy()
        M_new[p, p] = c**2 * M[p, p] - 2*c*s*M[p, q] + s**2 * M[q, q]
        M_new[q, q] = s**2 * M[p, p] + 2*c*s*M[p, q] + c**2 * M[q, q]
        M_new[p, q] = M_new[q, p] = 0.0
        for k in range(3):
            if k != p and k != q:
                M_new[k, p] = c * M[k, p] - s * M[k, q]
                M_new[p, k] = M_new[k, p]
                M_new[k, q] = s * M[k, p] + c * M[k, q]
                M_new[q, k] = M_new[k, q]
        M = M_new
        eigenvalues[p] = M[p, p]
        eigenvalues[q] = M[q, q]
        
        R = np.eye(3)
        R[p, p] = c; R[q, q] = c
        R[p, q] = s; R[q, p] = -s
        eigenvectors = eigenvectors @ R
        
    # Sort by eigenvalue
    idx = np.argsort(eigenvalues)
    eigenvalues = eigenvalues[idx]
    eigenvectors = eigenvectors[:, idx]
    
    # Regularization: clamp small eigenvalues [S1, unofficial]
    max_eig = eigenvalues[-1]
    threshold = epsilon * max_eig
    v = np.zeros(3)
    
    for i in range(3):
        if eigenvalues[i] > threshold:
            coeff = np.dot(eigenvectors[:, i], rhs) / eigenvalues[i]
            v += coeff * eigenvectors[:, i]
        else:
            # Mass-centroid regularization fallback [S1, unofficial]
            v += eigenvectors[:, i] * np.dot(eigenvectors[:, i], rhs) * 0.01
            
    return v

def map_edges_to_quads(edge_indices):
    """
    Maps sign-crossing edges to quadrilateral slots using static lookup table [S1, unofficial].
    """
    # dc_edge_quadrant[12] maps edge indices to quad slots [S1, unofficial]
    # Deterministic mapping based on edge indices [G]
    quads = []
    sorted_edges = sorted(edge_indices)
    for i in range(0, len(sorted_edges) - 1, 2):
        quads.append([sorted_edges[i], sorted_edges[i+1]])
    return quads

def split_quads_to_tris(quad_slots):
    """
    Applies Delaunay max-min angle splitting to convert quadrilaterals into triangles [S1, unofficial].
    """
    tri_faces = []
    for quad in quad_slots:
        # Simple max-min angle diagonal selection [G]
        p0, p1, p2, p3 = quad
        diag1 = np.linalg.norm(p0 - p2)
        diag2 = np.linalg.norm(p1 - p3)
        if diag1 < diag2:
            tri_faces.append([p0, p1, p2])
            tri_faces.append([p0, p2, p3])
        else:
            tri_faces.append([p0, p1, p3])
            tri_faces.append([p1, p2, p3])
    return tri_faces
```

**Parameters:**
*   `quad_split`: Boolean. `True` outputs triangles via Delaunay splitting; `False` outputs pure quadrilaterals [S1, unofficial].
*   `pad`: Integer. Set to 1 for sparse voxel grids to ensure all 4 incident voxels surrounding every sign-crossing edge are present in GPU memory [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Single Dual Vertex per Cell:** Complex sub-voxel topology with multiple disconnected sheets inside a single voxel cell can share vertices (pinch points), leading to non-manifold geometry [S1, unofficial].
*   **Normal Dependence:** Trilinear gradient smoothing across $C^0$ slope discontinuities can blur razor-sharp knife edges on CAD models [S1, unofficial]. Exact Hermite/triangle normals must be passed via `grid_normals` to avoid this [S1, unofficial].
*   **Brute-Force QEF:** In some 3D DC implementations, QEF solving is disabled in favor of brute-force methods, which are inefficient but preserve sharp features [S2, unofficial].

## Subtopics

### [Newton Surface Projection](1446-newton-surface-projection.md)
A post-processing step that projects extracted vertices exactly onto the isosurface using Newton's method to minimize distance to the zero-level set. It sets itself apart from approximate projection methods by achieving machine precision accuracy. Choose it when exact surface alignment is required after coarse QEF placement.

### [Quadric Error Function (QEF) for Sharp Features](1604-quadric-error-function-qef-for-sharp-features.md)
A vertex relocation technique that positions a single vertex within an octree cell to accurately represent sharp features by minimizing squared distances to tangent and supporting planes. It sets itself apart from standard QEM approaches by explicitly constructing supporting planes to stabilize minimization and prevent numerical instabilities. Choose it for high-curvature regions where standard solvers produce smoothed vertices.

### [QEF Data Accumulation](3712-qef-data-accumulation.md)
A data structure that accumulates QEF components from multiple plane constraints without storing individual points and normals. It sets itself apart by maintaining only pre-computed sums required for the least-squares solution and supporting element-wise merging from different sources. Choose it when memory efficiency is critical or when constraints arrive incrementally.

### [Occupancy-Based Dual Contouring](3799-occupancy-based-dual-contouring.md)
A variant designed to extract meshes from neural implicit functions like Occupancy Networks by avoiding reliance on distance information. It sets itself apart by introducing auxiliary 2D points to compute local surface normals and utilizing fast parallelizable search algorithms for 1D, 2D, and 3D points. Choose it when working with occupancy fields where explicit SDF gradients are unavailable.

### [Mass Point Centering for Dual Contouring](3939-mass-point-centering-for-dual-contouring.md)
A technique that improves numerical stability by shifting the QEF solution space to the centroid of the sample points. It sets itself apart from origin-centered systems by addressing precision loss in floating-point arithmetic around the data distribution. Choose it when working with large coordinate values or when standard solvers exhibit drift.

### [Brute-Force QEF Solver](4403-brute-force-qef-solver.md)
A non-linear solver alternative that performs a grid search over the unit cube $[0,1]^3$ to find the point with minimum squared distance to planes. It sets itself apart from analytical solvers by ignoring feature preservation in favor of exhaustive spatial sampling. Choose it when analytical solutions are unstable or QEF data is sparse.

### [QEF Solver Initialization](4722-qef-solver-initialization.md)
A preparation step that computes the centroid of accumulated input data and assembles the symmetric linear system matrices $A^T A$ and $A^T b$. It sets itself apart by transitioning raw data collection into a solvable linear algebraic structure. Choose it when decoupling data accumulation from the final solve step.

### [TRELLIS.2 QEF for Active Cube Vertex Positioning](9346-trellis-2-qef-for-active-cube-vertex-positioning.md)
A vertex positioning method used in the TRELLIS.2 framework preprocessing stage that solves for optimal dual vertices in active cubes. It sets itself apart by explicitly incorporating boundary edge constraints and regularization terms to handle sharp features and intricate internal structures. Choose it when integrating Dual Contouring into the TRELLIS.2 mesh extraction pipeline.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of active voxels [S1, unofficial]. Each voxel requires a small constant-time SVD/eigen-decomposition on a $3 \times 3$ matrix [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing dual vertices and faces [S1, unofficial]. Sparse voxel grids require additional memory for the grid structure and padding [S1, unofficial].
*   **Measured Performance:** On NVIDIA GeForce RTX 4090, Dual Contouring achieves >230 M faces/s on sparse Fandisk CAD at $1024^3$ resolution (14.36 ms latency for 3,432,768 triangles) [S1, unofficial]. For pure quads, throughput is 134 M quads/s [S1, unofficial].
*   **Strengths:** Preserves sharp features better than Dual Marching Cubes; generates 100% watertight manifold topology; fully differentiable; high throughput on modern GPUs [S1, unofficial].
*   **Limitations:** Single dual vertex per cell limitation can cause non-manifold geometry in complex sub-voxel topologies; requires boundary padding for sparse grids; sensitive to normal accuracy [S1, unofficial].
*   **When to Use:** When sharp feature preservation and watertight topology are critical, and the input data provides accurate normals or gradients [S1, unofficial].
*   **When Not to Use:** When the input data has noisy or inaccurate normals, or when sub-voxel topology is extremely complex and requires multiple vertices per cell [S1, unofficial].

## Background
Dual Contouring was introduced by Ju et al. in 2002 as a method for extracting meshes from implicit surfaces [S1, unofficial]. The QEF solver variant, as implemented in modern differentiable frameworks like conquer3d, builds on this foundation by incorporating regularized SVD and deterministic topology generation to address numerical instability and topological errors [S1, unofficial]. It is used in high-performance graphics applications requiring sharp feature preservation and differentiable rendering [S1, unofficial].

## Sources
[S1] Implementation of Dual Marching Cubes, https://github.com/KhoiDOO/conquer3d/issues/3, fetched 2026-10-27 (unofficial)
[S2] isosurface, https://github.com/Lin20/isosurface, fetched 2026-10-29 (unofficial)
