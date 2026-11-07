# Dual Contouring QEF Solver

Also called: Dual Contouring QEF Solver Implementation, QEF Solver for Dual Contouring, QEF Solving

## Overview
The Dual Contouring QEF Solver is the core optimization step in Dual Contouring that determines the position of a single dual vertex within each voxel cell containing an isosurface intersection [S1, unofficial]. It solves the problem of placing a vertex that best satisfies the geometric constraints (normals and intersection points) of the intersecting surface by minimizing a Quadratic Error Function (QEF) [S1, unofficial]. It sets itself apart from its siblings, such as [Topologically Accurate Dual Isosurfacing Using Ray Intersection](800-topologically-accurate-dual-isosurfacing-using-ray-intersect.md) which relies on ray casting for topology, and [Linear Edge Intersection for Dual Contouring](926-linear-edge-intersection-for-dual-contouring.md) which uses linear approximations, by explicitly constructing and solving a least-squares system via Singular Value Decomposition (SVD) to preserve sharp features and creases [S1, unofficial].

## How it works
The solver operates on a per-voxel basis. For each voxel cell, it collects plane constraints from edges where the scalar field changes sign, then solves for the optimal vertex position that minimizes the distance to these planes.

### Inputs and Outputs
**Inputs:**
*   `voxel_cell`: The 3D grid cell index $(i, j, k)$.
*   `grid_vertices`: Array of voxel corner coordinates.
*   `sdf`: Implicit function values at grid vertices.
*   `grid_normals`: Normals at grid vertices. If not provided, computed via trilinear gradient of `sdf` [S1, unofficial].
*   `quad_split`: Boolean flag. If `True`, output triangles via Delaunay splitting; if `False`, output quads [S1, unofficial].
*   `pad`: Boundary padding for sparse voxel grids. Set to 1 to ensure all 4 incident voxels surrounding every sign-crossing edge are present in GPU memory [S1, unofficial].

**Outputs:**
*   `dual_vertex`: Position $(x, y, z)$ of the new vertex inside the cell.
*   `mesh_topology`: Indices for edges forming quadrilaterals or triangles, determined by Deterministic Cartesian Quadrant Slotting.

### Data Structures
**Plane Constraint:**
Each sign-crossing edge provides a plane constraint defined by a point $p_i$ and a normal $n_i$.
*   $p_i$: Intersection point on the edge.
*   $n_i$: Normal vector at the intersection point.

**QEF Matrix System:**
The QEF is defined as minimizing $\sum_i (n_i \cdot (x - p_i))^2$. This expands to the linear system $Ax = b$, where:
*   $A$ is a matrix where each row is the normal vector $n_i^T$.
*   $b$ is a vector where each element is $n_i \cdot p_i$.
*   The solution minimizes $\|Ax - b\|^2$.

### Numbered Steps

1.  **Identify Sign-Crossing Edges:**
    For the current voxel cell, examine the 12 edges. For each edge, check if the scalar field values at the two endpoints have opposite signs (or one is zero). If a sign change exists, the isosurface intersects this edge.

2.  **Collect Plane Constraints:**
    For each intersecting edge:
    *   Compute the intersection point $p_i$ using linear interpolation between the edge endpoints based on the scalar field values.
    *   Compute the normal $n_i$. If `grid_normals` are provided, interpolate them. Otherwise, compute the gradient of the SDF at the intersection point [S1, unofficial].
    *   Accumulate the constraint $(p_i, n_i)$.

3.  **Solve the QEF:**
    Construct the matrix $A$ (size $N \times 3$, where $N$ is the number of intersecting edges) and vector $b$ (size $N$).
    *   Apply **Regularized SVD Truncation**: Perform Singular Value Decomposition on $A$. Let $\sigma_{\max}$ be the largest singular value. Singular values $\sigma_i < 0.01 \sigma_{\max}$ are clamped or truncated. This prevents dual vertices from drifting outside the voxel boundary when normals are nearly coplanar [S1, unofficial].
    *   Use **Mass-Centroid Regularization**: Shift the solution space to the centroid of the sample points to improve numerical stability [S1, unofficial].
    *   Solve for $x$ using a Jacobi $3 \times 3$ symmetric eigensolver SVD [S1, unofficial].

4.  **Generate Output Mesh Topology:**
    Use **Deterministic Cartesian Quadrant Slotting** to map the intersecting edges to quadrilateral slots.
    *   Instead of floating-point polar angle sorting (`atan2f`), use a static compile-time lookup table `dc_edge_quadrant[12]` [S1, unofficial].
    *   This eliminates self-intersecting "bowtie" quads caused by coincident dual vertices on sharp features [S1, unofficial].

5.  **Split Quads (Optional):**
    *   If `quad_split=True`, apply Delaunay max-min angle splitting to convert the generated quadrilaterals into triangles [S1, unofficial].
    *   If `quad_split=False`, output pure quadrilateral meshes [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def solve_qef(intersections, normals):
    """
    Solves the QEF for a set of intersection points and normals.
    
    Args:
        intersections: List of (x, y, z) points on the edges.
        normals: List of (nx, ny, nz) normals at those points.
        
    Returns:
        vertex: The optimal dual vertex position.
    """
    if not intersections:
        return None

    # Convert to numpy arrays
    P = np.array(intersections)  # Shape (N, 3)
    N = np.array(normals)        # Shape (N, 3)

    # Construct A and b for Ax = b
    # The QEF minimizes sum((N_i . (x - P_i))^2)
    # This is equivalent to minimizing ||Nx - d||^2 where d_i = N_i . P_i
    A = N
    b = np.dot(N, P.T)  # Shape (N,)

    # Mass centroid regularization: shift coordinates to centroid
    centroid = np.mean(P, axis=0)
    P_shifted = P - centroid
    
    # Recompute b with shifted points
    b_shifted = np.dot(N, P_shifted.T)

    # Solve using SVD
    # A = U @ S @ Vt
    U, S, Vt = np.linalg.svd(A, full_matrices=False)
    
    # Regularized SVD Truncation
    # Singular values < 0.01 * max(S) are clamped
    sigma_max = np.max(S)
    threshold = 0.01 * sigma_max
    
    # Create regularization matrix
    # We want to solve min ||Ax - b||^2 + lambda * ||x||^2 if ill-conditioned
    # Here we clamp small singular values in the pseudo-inverse
    S_inv = np.zeros_like(S)
    for i, s in enumerate(S):
        if s < threshold:
            S_inv[i] = 0.0  # Truncate
        else:
            S_inv[i] = 1.0 / s
            
    # Solution x = V @ S_inv @ U^T @ b
    x_shifted = Vt.T @ (S_inv * (U.T @ b_shifted))
    
    # Shift back to original coordinate system
    vertex = x_shifted + centroid
    
    return vertex

def dual_contouring_qef_solver(grid_vertices, voxels, sdf, grid_normals=None, quad_split=False, pad=1):
    """
    Main solver for Dual Contouring QEF.
    
    Args:
        grid_vertices: Array of voxel corner positions.
        voxels: Sparse voxel grid structure.
        sdf: Implicit function values at grid vertices.
        grid_normals: Optional pre-computed normals.
        quad_split: Boolean to toggle triangle output.
        pad: Boundary padding for sparse grids.
        
    Returns:
        verts: Array of dual vertex positions.
        faces: Array of face indices.
    """
    verts = []
    faces = []
    
    # Iterate over all active voxels
    for cell in voxels.active_cells:
        # 1. Identify sign-crossing edges
        intersections = []
        normals = []
        
        for edge in cell.edges:
            v0, v1 = edge.vertices
            val0 = sdf[v0]
            val1 = sdf[v1]
            
            if val0 * val1 < 0:  # Sign crossing
                # Compute intersection point
                t = val0 / (val0 - val1)
                p = v0 + t * (v1 - v0)
                intersections.append(p)
                
                # Compute normal
                if grid_normals is not None:
                    n0, n1 = grid_normals[v0], grid_normals[v1]
                    n = n0 + t * (n1 - n0)
                    n = n / np.linalg.norm(n)
                else:
                    # Compute gradient of SDF
                    grad = compute_sdf_gradient(sdf, v0, v1)
                    n = grad / np.linalg.norm(grad)
                    
                normals.append(n)
                
        # 2. Solve QEF
        vertex = solve_qef(intersections, normals)
        
        if vertex is not None:
            verts.append(vertex)
            
            # 3. Generate Topology using Deterministic Cartesian Quadrant Slotting
            # Map edges to quadrilateral slots using lookup table
            quad_indices = get_quadrant_slots(cell, intersections)
            
            if quad_split:
                # 4. Split quads into triangles using Delaunay max-min angle splitting
                tri_faces = split_quads_to_tris(quad_indices)
                faces.extend(tri_faces)
            else:
                faces.extend(quad_indices)
                
    return np.array(verts), np.array(faces)
```

### Parameters
*   `quad_split`: Boolean flag. `True` for Delaunay triangulated triangle output; `False` for pure quadrilateral mesh output [S1, unofficial].
*   `pad`: Integer. Boundary padding for sparse voxel grids. Set to 1 to ensure all 4 incident voxels surrounding every sign-crossing edge are present in GPU memory [S1, unofficial].

### Edge Cases and Pitfalls
*   **Single Dual Vertex per Cell Limitation:** Complex sub-voxel topology with multiple disconnected sheets inside a single voxel cell can share vertices (pinch points), leading to non-manifold geometry [S1, unofficial].
*   **Normal Dependence:** Trilinear gradient smoothing across $C^0$ slope discontinuities can blur razor-sharp knife edges on CAD models. Exact Hermite/triangle normals must be passed via `grid_normals` to avoid this [S1, unofficial].
*   **Numerical Instability:** When normals are nearly coplanar, the QEF system becomes ill-conditioned. Regularized SVD truncation and mass-centroid regularization are required to prevent dual vertices from drifting outside the voxel boundary [S1, unofficial].
*   **Self-Intersecting Quads:** Using floating-point polar angle sorting (`atan2f`) can cause self-intersecting "bowtie" quads. Deterministic Cartesian Quadrant Slotting with a static lookup table `dc_edge_quadrant[12]` eliminates this [S1, unofficial].

## Subtopics

### [Quadric Error Function (QEF) for Sharp Features](1604-quadric-error-function-qef-for-sharp-features.md)
Quadric Error Function (QEF) for Sharp Features is a vertex relocation technique used in Dual Contouring to position a single vertex within an octree cell such that it accurately represents sharp features (edges and corners) of the isosurface. It solves the problem of standard Dual Contouring producing smoothed or inaccurate vertices at high-curvature regions by minimizing the sum of squared distances from the vertex to the tangent planes defined by isosurface intersection points. It sets itself apart from its siblings, such as [Brute-Force QEF Solver](4403-brute-force-qef-solver.md) and [QEF Solver Initialization](4722-qef-solver-initialization.md), by explicitly constructing "supporting planes" in addition to tangent planes to stabilize the minimization process and prevent numerical instabilities inherent in standard Quadric Error Metric (QEM) approaches.

### [QEF Data Accumulation](3712-qef-data-accumulation.md)
QEF Data Accumulation is a data structure for accumulating the components of a Quadratic Error Function (QEF) from multiple plane constraints without storing individual points and normals. It enables the downstream [Dual Contouring QEF Solver](1424-dual-contouring-qef-solver.md) to reconstruct the optimal vertex position from the accumulated normal equations. The method maintains only the pre-computed sums required for the least-squares solution and supports merging accumulators from different sources via element-wise addition.

### [Occupancy-Based Dual Contouring](3799-occupancy-based-dual-contouring.md)
Occupancy-Based Dual Contouring (ODC) is a dual contouring method designed to extract meshes from neural implicit functions defined in continuous 3D space, such as Occupancy Networks, IM-NET, or DeepSDF. It modifies the computation of grid edge points and grid cell points to avoid relying on distance information, which is often unavailable or unreliable in occupancy fields. By introducing auxiliary 2D points to compute local surface normals, ODC identifies 3D points via the Quadric Error Function (QEF) and utilizes fast, parallelizable search algorithms for 1D, 2D, and 3D points.

### [Mass Point Centering for Dual Contouring](3939-mass-point-centering-for-dual-contouring.md)
Mass Point Centering improves the numerical stability of the Quadratic Error Function (QEF) solver in Dual Contouring by shifting the solution space to the centroid of the sample points. It addresses the problem of ill-conditioned linear systems that arise when intersection points are far from the origin, ensuring robust convergence for the dual vertex position. It differs from [Brute-Force QEF Solver](4403-brute-force-qef-solver.md) by optimizing the conditioning of the matrix system rather than just accumulating errors, and from [QEF Solver Initialization](4722-qef-solver-initialization.md) by actively modifying the right-hand side vector to reflect the shifted coordinate system.

### [Brute-Force QEF Solver](4403-brute-force-qef-solver.md)
A method for solving the Quadratic Error Function (QEF) by performing a brute-force grid search over the unit cube $[0,1]^3$ to find the point with the minimum squared distance to a set of planes, rather than using a closed-form linear algebra solution. It serves as a robust, non-linear solver alternative for Dual Contouring when analytical solutions are unstable or when the QEF data is sparse. It differs from [QEF Solver Initialization](4722-qef-solver-initialization.md) by providing the full optimization routine, and from [Quadric Error Function (QEF) for Sharp Features](1604-quadric-error-function-qef-for-sharp-features.md) by ignoring feature preservation in favor of exhaustive spatial sampling.

### [QEF Solver Initialization](4722-qef-solver-initialization.md)
QEF Solver Initialization prepares the Quadratic Error Function (QEF) system for numerical solution by computing the centroid of the accumulated input data and assembling the symmetric linear system matrices $A^T A$ and $A^T b$. It transitions from raw data collection to the construction of the solvable linear algebraic structure required by the [Dual Contouring QEF Solver](1424-dual-contouring-qef-solver.md).

### [TRELLIS.2 QEF for Active Cube Vertex Positioning](9346-trellis-2-qef-for-active-cube-vertex-positioning.md)
TRELLIS.2 QEF is a vertex positioning method used in the preprocessing stage of the TRELLIS.2 framework. It solves the problem of determining the optimal position for dual vertices within "active cubes" identified by intersecting an input mesh, enabling the subsequent extraction of high-quality meshes via Dual Contouring. It distinguishes itself from standard Dual Contouring QEF solvers by explicitly incorporating boundary edge constraints and regularization terms to handle sharp features and intricate internal structures.

### Newton Surface Projection
Newton Surface Projection is a post-processing step that projects all extracted vertices exactly onto the isosurface using Newton's method, achieving machine precision accuracy [S3, unofficial]. After Dual Contouring vertex placement, it applies Newton iteration to each vertex position to minimize the distance to the isosurface defined by the implicit function [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** The solver operates in $O(N)$ time per voxel, where $N$ is the number of intersecting edges (typically 12). The SVD solution is constant time for a $3 \times 3$ system.
*   **Memory Complexity:** $O(1)$ per voxel for storing the QEF matrices and solution.
*   **Measured Performance:** On NVIDIA GeForce RTX 4090, Dual Contouring achieves >230 M faces/s on sparse Fandisk CAD at $1024^3$ resolution (14.36 ms latency for 3,432,768 triangles) [S1, unofficial]. For pure quad output, it achieves 134 M quads/s [S1, unofficial].
*   **Scaling:** The method is highly parallelizable and suitable for GPU implementation. Sparse voxel grids require boundary padding (`pad=1`) to ensure all incident voxels are present in memory [S1, unofficial].
*   **Strengths:** Preserves sharp features (creases, apexes) better than Dual Marching Cubes because it places vertices directly on sharp creases via QEF SVD [S1, unofficial]. Produces 100% watertight manifold topology with exact Cartesian quadrant indexing [S1, unofficial]. Fully differentiable, allowing backpropagation of analytical gradients directly into SDF values and colors features [S1, unofficial].
*   **Limitations:** Single dual vertex per cell limitation can lead to non-manifold geometry in complex sub-voxel topologies [S1, unofficial]. Dependent on accurate normals; trilinear gradient smoothing can blur sharp edges if exact normals are not provided [S1, unofficial].
*   **When to Use:** When sharp feature preservation is critical, and watertight manifold topology is required. Suitable for CAD models and mechanical parts.
*   **When Not to Use:** When the scene contains complex sub-voxel topology with multiple disconnected sheets within a single voxel, as this may lead to pinch points. Also not suitable if exact normals are unavailable and gradient smoothing is unacceptable.

## Background
Dual Contouring was introduced by Ju et al. in 2002 as a method for extracting polygonal meshes from implicit surfaces [S1, unofficial]. The QEF solver is a key component of this method, allowing for the placement of vertices that satisfy multiple plane constraints. Recent implementations, such as those by KhoiDOO, have optimized the solver for GPU performance and differentiability, achieving high throughput on modern hardware [S1, unofficial]. The method is used in various graphics applications requiring high-fidelity surface extraction, particularly in CAD and medical imaging.

## Sources
[S1] Implementation of Dual Marching Cubes, https://github.com/KhoiDOO/conquer3d/issues/3, fetched 2026-10-27 (unofficial)
[S2] isosurface, https://github.com/Lin20/isosurface, fetched 2026-10-29 (unofficial)
[S3] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
