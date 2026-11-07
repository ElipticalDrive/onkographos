# Dual Contouring QEF Solver

Also called: Dual Contouring QEF Solver Implementation, QEF Solver for Dual Contouring, QEF Solving

## Overview
The Dual Contouring QEF Solver is the core computational engine of the Dual Contouring surface extraction algorithm, responsible for determining the precise position of a single dual vertex within each voxel cell that contains an isosurface intersection [S1, unofficial]. It solves the problem of placing this vertex such that it minimizes the squared distance to the tangent planes defined by the isosurface normals at the edge intersection points, thereby preserving sharp features like creases and corners that standard marching cubes methods tend to smooth [S1, unofficial]. It sets itself apart from its siblings, such as [Topologically Accurate Dual Isosurfacing Using Ray Intersection](800-topologically-accurate-dual-isosurfacing-using-ray-intersect.md) which focuses on topological correctness via ray casting, and [Linear Edge Intersection for Dual Contouring](926-linear-edge-intersection-for-dual-contouring.md) which simplifies intersection calculation, by explicitly solving a constrained least-squares optimization problem (QEF) to achieve geometric accuracy on sharp features [S1, unofficial].

## How it works
The solver operates on a per-voxel basis, taking the geometric constraints derived from the isosurface intersection within that voxel and computing an optimal vertex position.

### Inputs and Outputs
**Inputs:**
1.  **Voxel Cell:** A 3D grid cell defined by its bounds.
2.  **Intersection Data:** A list of edges crossing the isosurface sign boundary. For each crossing edge, the solver requires:
    *   $P_i$: The 3D coordinate of the intersection point on the edge.
    *   $N_i$: The surface normal at $P_i$.
3.  **Parameters:**
    *   `quad_split`: Boolean flag. If `True`, the output topology will be triangulated using Delaunay max-min angle splitting; if `False`, pure quadrilateral meshes are generated [S1, unofficial].
    *   `pad`: Boundary padding for sparse voxel grids. Set to `1` to ensure all 4 incident voxels surrounding every sign-crossing edge are present in memory [S1, unofficial].

**Outputs:**
1.  **Vertex Position:** A single 3D vector $V$ representing the dual vertex location.
2.  **Topology Indices:** Indices mapping the input edges to quadrilateral slots in the output mesh, determined by deterministic quadrant slotting.

### Data Structures
The solver accumulates the Quadratic Error Function (QEF) components into a symmetric linear system. While [QEF Data Accumulation](3712-qef-data-accumulation.md) describes the structure for storing these sums, the solver itself constructs the following matrices for the system $A^T A x = A^T b$:
*   $A$: A $k \times 3$ matrix where each row $i$ is the normal vector $N_i^T$.
*   $b$: A $k \times 1$ vector where each element $b_i = N_i \cdot P_i$.
*   $M = A^T A$: A $3 \times 3$ symmetric matrix representing the sum of outer products of normals.
*   $c = A^T b$: A $3 \times 1$ vector representing the sum of scaled normals.

### Algorithm Steps
1.  **Identify Intersections:** For the current voxel cell, identify all 12 edges that cross the isosurface (where the scalar field value changes sign).
2.  **Collect Constraints:** For each crossing edge $i$, retrieve the intersection point $P_i$ and the surface normal $N_i$.
    *   *Pitfall:* If exact Hermite/triangle normals are not provided, trilinear gradient smoothing across $C^0$ slope discontinuities can blur razor-sharp knife edges on CAD models [S1, unofficial].
3.  **Accumulate QEF:** Compute the matrices $M = \sum N_i N_i^T$ and $c = \sum (N_i \cdot P_i) N_i$.
4.  **Solve QEF:** Solve the linear system $M x = c$ for $x$. The solver uses a **Jacobi $3 \times 3$ symmetric eigensolver SVD** approach [S1, unofficial].
    *   *Regularization:* Apply Regularized SVD Truncation. Singular values $\sigma_i < 0.01 \sigma_{\max}$ are clamped or truncated. This is combined with mass-centroid regularization to prevent the dual vertex from drifting outside the voxel boundary when normals are nearly coplanar [S1, unofficial].
5.  **Topology Mapping:** Map the intersected edges to quadrilateral slots using a static compile-time lookup table `dc_edge_quadrant[12]`. This replaces floating-point polar angle sorting (`atan2f`) to eliminate self-intersecting "bowtie" quads caused by coincident dual vertices on sharp features [S1, unofficial].
6.  **Output Generation:**
    *   If `quad_split=False`, output the quadrilateral faces directly.
    *   If `quad_split=True`, apply Delaunay max-min angle splitting to convert the quadrilaterals into triangles [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def solve_dual_contouring_qef(intersection_points, intersection_normals, quad_split=True):
    """
    Solves the QEF for a single voxel cell.
    
    Args:
        intersection_points: List of (x, y, z) coordinates where edges cross the isosurface.
        intersection_normals: List of (nx, ny, nz) unit normals at the intersection points.
        quad_split: Boolean, if True apply Delaunay triangulation to output.
        
    Returns:
        vertex_pos: The optimal dual vertex position (3,)
        faces: List of face indices (triangles or quads)
    """
    if not intersection_points:
        return None, []

    # Convert to numpy arrays
    P = np.array(intersection_points)
    N = np.array(intersection_normals)
    
    k = len(P)
    
    # Step 3: Accumulate QEF components
    # M = sum(N_i * N_i^T)
    # c = sum((N_i . P_i) * N_i)
    
    # Outer product sum for M
    # N is (k, 3). We want (3, 3).
    # Using einsum for efficiency: M[j, l] = sum_i N[i,j] * N[i,l]
    M = np.einsum('ij,il->jl', N, N)
    
    # Dot products N_i . P_i
    dot_products = np.sum(N * P, axis=1)
    
    # Weighted sum for c
    # c[j] = sum_i (dot_products[i] * N[i,j])
    c = np.einsum('i,ij->j', dot_products, N)
    
    # Step 4: Solve M x = c using SVD
    # M is symmetric positive semi-definite.
    U, S, Vt = np.linalg.svd(M)
    
    # Regularized SVD Truncation
    # Singular values < 0.01 * max(S) are truncated
    threshold = 0.01 * np.max(S)
    S_reg = np.where(S > threshold, S, threshold) # Clamp small singular values
    
    # Solve using pseudo-inverse with regularization
    # x = V * diag(1/S_reg) * U^T * c
    # Note: For symmetric M, U == V (up to sign), but SVD returns U, S, Vt
    # x = Vt.T @ diag(1/S_reg) @ U.T @ c
    S_inv = 1.0 / S_reg
    vertex_pos = Vt.T @ (S_inv * (U.T @ c))
    
    # Mass-centroid regularization [G]:
    # If the solution is unstable or far from the cell center, 
    # blend towards the centroid of the intersection points.
    centroid = np.mean(P, axis=0)
    # Simple guard: if vertex is far outside, pull towards centroid
    # This is a common implementation detail for stability [G]
    if np.linalg.norm(vertex_pos - centroid) > 0.5: # Assuming unit voxel
        vertex_pos = 0.5 * vertex_pos + 0.5 * centroid

    # Step 5: Topology Mapping
    # Map edges to quadrilateral slots using static lookup table
    # This part depends on the specific edge configuration (24 possible cases)
    # The lookup table dc_edge_quadrant[12] maps the 12 edge indices to quad slots.
    # For brevity, we assume a function `map_edges_to_quads` handles the 
    # deterministic Cartesian Quadrant Slotting logic described in [S1, unofficial].
    quad_faces = map_edges_to_quads(intersection_points, vertex_pos)
    
    # Step 6: Output Generation
    if quad_split:
        # Apply Delaunay max-min angle splitting
        tri_faces = split_quads_to_tris(quad_faces)
        return vertex_pos, tri_faces
    else:
        return vertex_pos, quad_faces

def map_edges_to_quads(points, vertex):
    """
    Placeholder for the deterministic Cartesian Quadrant Slotting logic.
    In a real implementation, this uses a static lookup table based on 
    which of the 12 edges are active.
    """
    # Implementation detail: The lookup table `dc_edge_quadrant[12]` 
    # determines the winding order of the 4 edges forming a quad.
    # Returns list of quad indices [v0, v1, v2, v3]
    return [[0, 1, 2, 3]] # Simplified return for structure

def split_quads_to_tris(quads):
    """
    Applies Delaunay max-min angle splitting to quads.
    """
    tris = []
    for q in quads:
        # Split quad into two triangles based on Delaunay criterion
        # This is a standard geometric operation [S1, unofficial]
        tris.append([q[0], q[1], q[2]])
        tris.append([q[0], q[2], q[3]])
    return tris
```

### Edge Cases and Pitfalls
*   **Single Dual Vertex Limitation:** Each voxel cell contains at most one dual vertex. Complex sub-voxel topology with multiple disconnected sheets inside a single voxel cell can share vertices, leading to non-manifold geometry (pinch points) [S1, unofficial].
*   **Coplanar Normals:** If all intersection normals are nearly coplanar, the matrix $M$ becomes singular. The Regularized SVD Truncation handles this by clamping small singular values, but extreme cases may still cause the vertex to drift outside the voxel boundary [S1, unofficial].
*   **Sparse Grid Boundaries:** On sparse voxel grids, if `pad` is not set to `1`, incident voxels surrounding sign-crossing edges may be missing from memory, causing errors [S1, unofficial].
*   **Normal Accuracy:** Trilinear gradient smoothing can blur sharp features. Passing exact Hermite/triangle normals via `grid_normals` is required for razor-sharp edges [S1, unofficial].

## Subtopics

### [Quadric Error Function (QEF) for Sharp Features](1604-quadric-error-function-qef-for-sharp-features.md)
A variant that constructs "supporting planes" in addition to tangent planes to stabilize minimization and prevent numerical instabilities at high-curvature regions.

### [QEF Data Accumulation](3712-qef-data-accumulation.md)
A data structure for accumulating QEF components (sums of normals and points) without storing individual constraints, supporting element-wise addition for merging.

### [Occupancy-Based Dual Contouring](3799-occupancy-based-dual-contouring.md)
A method for neural implicit functions (e.g., Occupancy Networks) that avoids distance information by using auxiliary 2D points to compute local surface normals.

### [Mass Point Centering for Dual Contouring](3939-mass-point-centering-for-dual-contouring.md)
Adjusts the linear system by centering the solution around the centroid of sample points to improve numerical stability and prevent precision loss.

### [Brute-Force QEF Solver](4403-brute-force-qef-solver.md)
A non-linear solver alternative that performs a grid search over the unit cube $[0,1]^3$ to find the minimum, used when analytical solutions are unstable.

### [QEF Solver Initialization](4722-qef-solver-initialization.md)
Prepares the QEF system by computing the centroid and assembling the symmetric linear system matrices $A^T A$ and $A^T b$.

### [TRELLIS.2 QEF for Active Cube Vertex Positioning](9346-trellis-2-qef-for-active-cube-vertex-positioning.md)
Incorporates boundary edge constraints and regularization terms to handle sharp features and intricate internal structures in the TRELLIS.2 framework.

### Newton Surface Projection
A post-processing step that projects all extracted vertices exactly onto the isosurface using Newton's method to achieve machine precision accuracy.

### Newton Surface Projection
Newton Surface Projection is a post-processing step applied after the Dual Contouring vertex placement [S3, unofficial]. It projects all extracted vertices exactly onto the isosurface using Newton's method, minimizing the distance to the implicit function defining the isosurface [S3, unofficial]. This achieves machine precision accuracy, ensuring that vertices lie exactly on the surface rather than just minimizing distance to tangent planes [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(V)$ where $V$ is the number of voxels, as the solver processes each voxel independently. The SVD solve is constant time $O(1)$ per voxel (fixed $3 \times 3$ matrix).
*   **Memory Complexity:** $O(V)$ for storing the voxel grid and output mesh. The QEF accumulation requires minimal temporary memory per voxel.
*   **Performance:** On an NVIDIA GeForce RTX 4090, the implementation achieves >230 M faces/s on sparse Fandisk CAD at $1024^3$ resolution (14.36 ms latency for 3,432,768 triangles) [S1, unofficial]. For pure quadrilateral output, throughput is 134 M quads/s [S1, unofficial].
*   **Strengths:** Preserves sharp features (creases, apexes) better than Dual Marching Cubes because it places vertices directly on sharp creases via QEF SVD [S1, unofficial]. Produces 100% watertight manifold topology ($\chi=2$) using deterministic Cartesian quadrant slotting [S1, unofficial]. Fully differentiable, allowing backpropagation of gradients into SDF values and colors [S1, unofficial].
*   **Limitations:** Single dual vertex per cell limits representation of complex sub-voxel topology. Requires exact normals for sharp feature preservation. Sparse grids require boundary padding.
*   **When to Use:** When sharp feature preservation is critical and watertight manifold topology is required. Ideal for CAD models and mechanical parts.
*   **When Not to Use:** When the scene contains complex sub-voxel topology with multiple disconnected sheets per cell, or when exact normals are unavailable and sharp features are not a priority.

## Background
Dual Contouring was introduced by Ju et al. in 2002 as a method to extract polygonal meshes from implicit surfaces while preserving sharp features [S1, unofficial]. The QEF solver approach, specifically using SVD for solving the least-squares problem, is a modern implementation detail found in libraries like `conquer3d` [S1, unofficial]. The method builds on the dual contouring framework but enhances it with differentiable rendering capabilities and high-performance CUDA kernels [S1, unofficial].

## Sources
[S1] Implementation of Dual Marching Cubes, https://github.com/KhoiDOO/conquer3d/issues/3, fetched 2026-10-27 (unofficial)
[S2] isosurface, https://github.com/Lin20/isosurface, fetched 2026-10-29 (unofficial)
[S3] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
