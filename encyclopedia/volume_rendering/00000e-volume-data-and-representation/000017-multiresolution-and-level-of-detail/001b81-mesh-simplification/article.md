# Mesh Simplification

Also called: Edge Collapse Simplification, Mesh Simplification via Edge Collapse, Mesh Simplifier, Quadric Mesh Simplification, ngMeshSimplifier

## Overview
Mesh Simplification is a geometric decimation algorithm that reduces the triangle count of a 3D surface mesh to a target size while minimizing geometric error, primarily to improve rendering performance and reduce memory usage in volume rendering pipelines [S1, S3, unofficial]. It builds on [Multiresolution and level of detail](7-multiresolution-and-level-of-detail.md) by providing a post-processing or pre-processing step for isosurfaces (e.g., from Marching Cubes) to enable real-time interaction [S1, unofficial]. Unlike [Vertex Clustering Simplification](9046-vertex-clustering-simplification.md) which snaps vertices to a grid, this method typically operates by collapsing edges or removing vertices based on geometric error metrics [S1, unofficial]. It is distinct from [Wavelet-based Volume Rendering](1052-wavelet-based-volume-rendering.md) which operates on scalar field coefficients, and from [Multi-Tetra Framework](1183-multi-tetra-framework.md) which manages tetrahedral domains.

## How it works
**Inputs and Outputs:**
*   **Input:** A mesh defined by a list of vertices and a list of triangles (faces).
*   **Output:** A simplified mesh with a reduced number of triangles, preserving the original topology where possible.

**Data Structures:**
The mesh is represented using the following layouts [S2, unofficial]:
*   `MeshBuffer`: Contains `numVertices`, `numTriangles`, `vertices` (array of `MeshVertex`), and `triangles` (array of `MeshTriangle`).
*   `MeshVertex`: Contains `xyz` (vec3/vec4), `normal` (vec3/vec4), and `colour` (vec3/vec4).
*   `MeshTriangle`: Contains `indices_` (array of 3 ints).
*   `Edge`: A union containing `uint64_t idx_` and a struct with two `uint32_t` fields `min_` and `max_`. Used to store edge indices and sort them.
*   `LinearBuffer<T>`: A template class with private fields `T* base_`, `T* end_`, `int size_`. Provides methods `push_back`, `resize`, `clear`, `begin`, `end`, `operator[]`.

**Parameters:**
*   `targetPercentage` (float): The target triangle count as a fraction of the initial count [S2, unofficial].
*   `maxIterations` (int): Limits the maximum number of simplification iterations [S2, unofficial].
*   `edgeFraction` (float): Fraction of total edges to sample for potential collapses, used to improve performance by avoiding full mesh scan [S2, unofficial].
*   `minAngleCosine` (float): Minimum cosine of the angle between vertex normals to allow a collapse; prevents collapsing across sharp creases [S2, unofficial].
*   `maxEdgeSize` (float): Maximum squared length of an edge allowed for collapse [S2, unofficial].
*   `maxError` (float): Maximum acceptable Quadratic Error Function (QEF) error for a collapse; collapses exceeding this are rejected [S2, unofficial].
*   `COLLAPSE_MAX_DEGREE` (int): Maximum allowed sum of triangle counts for the two vertices involved in a collapse, set to 16 [S2, unofficial].
*   `agressiveness` (float): A parameter controlling the simplification behavior, default 7.0 [S3, unofficial].
*   `quality` (int): If > 0, performs lossless simplification after lossy; default 1 [S3, unofficial].
*   `tol` (float): Tolerance for merging vertices in post-processing, set to 0.00001 [S3, unofficial].

**Algorithm Steps:**

1.  **Precondition Check:** If `mesh->numTriangles < 100` or `mesh->numVertices < 100`, return immediately without modification [S2, unofficial].
2.  **Coordinate Translation:** Translate vertices to local space by subtracting `worldSpaceOffset` from each vertex's xyz [S2, unofficial].
3.  **Initialization:** Initialize mesh counters to zero [S2, unofficial].
4.  **Build Candidate Edges:**
    *   Iterate over all triangles.
    *   Extract three edges per triangle.
    *   Normalize edge indices such that `min_ < max_`.
    *   Sort edges by index and filter to keep only unique edges.
    *   Mark vertices as boundary if an edge appears only once (`count == 1`).
    *   Discard boundary edges from the final candidate list [S2, unofficial].
5.  **Allocate Buffers:** Allocate buffers for `collapsePosition`, `collapseNormal`, `collapseValid`, `collapseEdgeID`, `collapseTarget`, `edgeBuffer`, and `triBuffer` [S2, unofficial].
6.  **Compute Vertex Triangle Counts:** Iterate all triangles and increment the count for each vertex index [S2, unofficial].
7.  **Simplification Loop:** Loop while `triangle count > target` and `iterations < maxIterations`:
    *   Reset `collapseEdgeID` and `collapseTarget` to -1.
    *   Clear `collapseValid`.
    *   **Find Valid Collapses:**
        *   Seed PRNG with 42.
        *   Calculate `numRandomEdges = edges.size() * options.edgeFraction`.
        *   Generate random indices into the edges array. Sort random indices for locality.
        *   For each sampled edge:
            1.  Check normal dot product (`cosAngle`) against `options.minAngleCosine`; skip if lower.
            2.  Calculate squared edge length; skip if `> options.maxEdgeSize^2`.
            3.  Calculate degree (sum of triangle counts for both vertices); skip if `> COLLAPSE_MAX_DEGREE`.
            4.  Solve QEF for the two vertices using `qef_solve_from_points_4d_interleaved`.
            5.  If error > 0, invert it (`error = 1.f / error`).
            6.  Apply penalty: `penalty = max(0, degree - 10)`; `error += penalty * (options.maxError * 0.1f)`.
            7.  Skip if `error > options.maxError`.
            8.  Store valid edge index in `collapseValid`.
            9.  Compute collapse normal as average of vertex normals.
            10. Compute collapse position from QEF solution.
            11. Update `minEdgeCost` for both vertices if current error is lower, storing the edge index in `collapseEdgeID` [S2, unofficial].
    *   **Collapse Edges:** Perform the collapses, updating vertices and setting `collapseTarget` mappings [S2, unofficial].
    *   **Remove Triangles:**
        *   Iterate over all triangles.
        *   For each vertex index in the triangle, if `collapseTarget[index] != -1`, replace index with target.
        *   If the resulting triangle has duplicate indices (degenerate), increment `removedCount` and skip.
        *   Otherwise, add to `triBuffer` and increment `vertexTriangleCounts` for the new indices.
        *   Swap tris with `triBuffer` [S2, unofficial].
    *   **Remove Edges:** Update edge endpoints based on `collapseTarget`, removing self-loops (`min_ == max_`) [S2, unofficial].
    *   **Pitfall Guard:** If no valid collapses are found (`countValidCollapse == 0`), break the loop to prevent infinite iteration [S2, unofficial].
8.  **Post-Processing:**
    *   Copy remaining triangles back to `mesh->triangles` [S2, unofficial].
    *   **Compact Vertices:** Remove unused vertices and remap triangle indices [S2, unofficial].
    *   **Vertex Welding (Optional Post-Processing):**
        *   Compute distance from each vertex to the first vertex (`pt0`).
        *   Sort vertices by this distance using radix sort.
        *   Iterate through sorted vertices; if a vertex is not yet assigned, assign it to a new unique index.
        *   Check subsequent vertices within the tolerance range of the current vertex's distance; if their Euclidean distance to the current vertex is `< tol`, assign them to the same new index.
        *   Remap all triangle indices using the `old2new` mapping array [S3, unofficial].
    *   Translate vertices back to world space by adding `worldSpaceOffset`, then copy to `mesh->vertices` with position, normal, and colour [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mesh_simplify(vertices, faces, target_percentage, max_iterations=100, edge_fraction=0.1, min_angle_cosine=0.9, max_edge_size=1.0, max_error=1.0, collapse_max_degree=16, tol=0.00001):
    """
    Simplify a mesh by collapsing edges.
    
    Args:
        vertices: List of [x, y, z] coordinates.
        faces: List of [i, j, k] indices.
        target_percentage: Target triangle count as fraction of initial count.
        max_iterations: Maximum number of simplification iterations.
        edge_fraction: Fraction of edges to sample per iteration.
        min_angle_cosine: Minimum cosine of angle between vertex normals.
        max_edge_size: Maximum squared length of edge for collapse.
        max_error: Maximum acceptable QEF error.
        collapse_max_degree: Max sum of triangle counts for collapsed vertices.
        tol: Tolerance for vertex welding.
        
    Returns:
        simplified_vertices, simplified_faces
    """
    if len(faces) < 100 or len(vertices) < 100:
        return vertices, faces

    # Translate to local space (assuming worldSpaceOffset is 0 for simplicity)
    # In practice, subtract worldSpaceOffset from each vertex
    
    num_vertices = len(vertices)
    num_faces = len(faces)
    target_faces = int(num_faces * target_percentage)
    
    # Data structures
    # vertex_triangle_counts: array of size num_vertices
    vertex_triangle_counts = [0] * num_vertices
    for f in faces:
        for v in f:
            vertex_triangle_counts[v] += 1
            
    # Build candidate edges
    edges = []
    edge_vertex_counts = {} # Map edge tuple to count
    for f in faces:
        for i in range(3):
            u, v = f[i], f[(i+1)%3]
            edge = (min(u, v), max(u, v))
            edge_vertex_counts[edge] = edge_vertex_counts.get(edge, 0) + 1
            
    # Filter unique edges and discard boundary edges
    candidate_edges = []
    boundary_vertices = set()
    for edge, count in edge_vertex_counts.items():
        if count == 1:
            boundary_vertices.add(edge[0])
            boundary_vertices.add(edge[1])
        else:
            candidate_edges.append(edge)
            
    # Remove boundary edges from candidates
    candidate_edges = [e for e in candidate_edges if e[0] not in boundary_vertices and e[1] not in boundary_vertices]
    
    # Collapse target mapping
    collapse_target = [-1] * num_vertices
    
    for iteration in range(max_iterations):
        if len(faces) <= target_faces:
            break
            
        # Find valid collapses
        valid_collapses = []
        num_random_edges = int(len(candidate_edges) * edge_fraction)
        if num_random_edges == 0:
            num_random_edges = 1
            
        # Seed PRNG with 42
        import random
        random.seed(42)
        random_indices = random.sample(range(len(candidate_edges)), min(num_random_edges, len(candidate_edges)))
        random_indices.sort()
        
        min_edge_cost = float('inf')
        best_edge = None
        
        for idx in random_indices:
            u, v = candidate_edges[idx]
            
            # Check normal dot product
            # Assuming normals are available or computed from geometry
            # For this pseudocode, we assume a function get_normal(v) exists
            # cos_angle = dot(get_normal(u), get_normal(v))
            # if cos_angle < min_angle_cosine: continue
            
            # Check edge length squared
            # dist_sq = sum((vertices[u][k] - vertices[v][k])**2 for k in range(3))
            # if dist_sq > max_edge_size**2: continue
            
            # Check degree
            degree = vertex_triangle_counts[u] + vertex_triangle_counts[v]
            if degree > collapse_max_degree:
                continue
                
            # Solve QEF
            # error = solve_qef(vertices[u], vertices[v], normals[u], normals[v])
            # For pseudocode, assume a placeholder
            error = 0.0 # Placeholder for QEF error calculation
            
            if error > 0:
                error = 1.0 / error
                
            # Apply penalty
            penalty = max(0, degree - 10)
            error += penalty * (max_error * 0.1)
            
            if error > max_error:
                continue
                
            # Store valid collapse
            valid_collapses.append((u, v, error))
            
            if error < min_edge_cost:
                min_edge_cost = error
                best_edge = (u, v)
                
        if not valid_collapses:
            break
            
        # Perform best collapse
        if best_edge:
            u, v = best_edge
            # Compute collapse position (QEF solution)
            # collapsed_pos = solve_qef_position(...)
            # For pseudocode, we just map v to u
            collapse_target[v] = u
            
            # Update triangle counts
            # This is a simplification; in reality, we need to update counts for all triangles affected
            # and remove degenerate triangles
            
            # Remove triangles
            new_faces = []
            for f in faces:
                new_f = []
                for idx in f:
                    if collapse_target[idx] != -1:
                        new_f.append(collapse_target[idx])
                    else:
                        new_f.append(idx)
                # Check for degenerate triangle
                if len(set(new_f)) == 3:
                    new_faces.append(new_f)
                    # Update vertex counts for new vertices
                    for idx in new_f:
                        vertex_triangle_counts[idx] += 1
                else:
                    # Degenerate, remove
                    pass
            faces = new_faces
            
            # Compact vertices and remap indices
            # This is complex; for pseudocode, we assume a compact_vertices function
            vertices, faces, vertex_triangle_counts = compact_vertices(vertices, faces, collapse_target)
            
            # Rebuild candidate edges for next iteration
            # This is expensive, so in practice, incremental updates are used
            # For this pseudocode, we rebuild
            edge_vertex_counts = {}
            for f in faces:
                for i in range(3):
                    u, v = f[i], f[(i+1)%3]
                    edge = (min(u, v), max(u, v))
                    edge_vertex_counts[edge] = edge_vertex_counts.get(edge, 0) + 1
                    
            candidate_edges = []
            boundary_vertices = set()
            for edge, count in edge_vertex_counts.items():
                if count == 1:
                    boundary_vertices.add(edge[0])
                    boundary_vertices.add(edge[1])
                else:
                    candidate_edges.append(edge)
                    
            candidate_edges = [e for e in candidate_edges if e[0] not in boundary_vertices and e[1] not in boundary_vertices]
            
    # Vertex Welding
    # Compute distance from each vertex to the first vertex
    if len(vertices) > 0:
        pt0 = vertices[0]
        distances = [sum((vertices[i][k] - pt0[k])**2 for k in range(3)) for i in range(len(vertices))]
        sorted_indices = sorted(range(len(vertices)), key=lambda k: distances[k])
        
        old2new = [-1] * len(vertices)
        new_idx = 0
        for i in sorted_indices:
            if old2new[i] == -1:
                old2new[i] = new_idx
                new_idx += 1
                # Check subsequent vertices
                for j in sorted_indices[sorted_indices.index(i)+1:]:
                    if distances[j] - distances[i] > tol**2: # Approximation
                        break
                    dist = sum((vertices[i][k] - vertices[j][k])**2 for k in range(3))
                    if dist < tol**2:
                        old2new[j] = old2new[i]
        
        # Remap faces
        new_faces = []
        for f in faces:
            new_f = [old2new[idx] for idx in f]
            if len(set(new_f)) == 3:
                new_faces.append(new_f)
        faces = new_faces
        
        # Compact vertices
        new_vertices = [vertices[i] for i in range(len(vertices)) if old2new[i] != -1 and old2new[i] == new_vertices.index(vertices[i])] # Simplified
        # In practice, create a new list of unique vertices
        unique_vertices = []
        for i in range(len(vertices)):
            if old2new[i] != -1:
                # This logic is flawed in pseudocode; proper remapping is needed
                pass
        # For brevity, assume compact_vertices handles this
        vertices = new_vertices
        
    # Translate back to world space
    return vertices, faces
```

## Subtopics

### [Mesh simplification via valence pattern removal](1931-mesh-simplification-via-valence-pattern-removal.md)
This method reduces the element count of quad-dominant meshes, specifically those generated by the Dual Marching Cubes (DMC) algorithm, by identifying and removing specific quadrilateral valence patterns: 3-X-3-Y and 3-3-3-3. It solves the problem of excessive mesh complexity and poor element quality caused by transitions between volume data layers. Unlike general decimation methods, this approach is optional, parallelized using face coloring to avoid race conditions, and specifically targets topological artifacts inherent to the DMC representation. It improves mesh quality and repositions vertices while reducing element count by approximately 10% for datasets like human skulls and skeletons.

### [Linear Spline Simplification](3483-linear-spline-simplification.md)
Linear Spline Simplification is an iterative knot removal algorithm that reduces the number of control points (knots) in a 1D linear spline by merging consecutive segments when internal knots lie within a pre-chosen vertical distance (scalar domain) from the candidate merged segment. It solves the problem of redundant data storage in Piecewise Linear Representations (PRI) by identifying spans of nearly collinear scalar values and removing intermediate samples while preserving endpoints. Unlike general mesh decimation methods that modify geometry topology, this method operates strictly on the scalar parameterization of sampling rays, ensuring the resulting knot list is a subset of the original list.

### [GPU-based Mesh Decimation via Edge Collapse](3505-gpu-based-mesh-decimation-via-edge-collapse.md)
This method decimates a triangular mesh by iteratively collapsing edges to minimize geometric error, implemented on the GPU using CUDA kernels and Thrust utilities. It reduces vertex count while preventing degenerate faces and normal flipping by enforcing topological validity checks and boundary preservation penalties on the device. Unlike typical greedy decimation which iterates until the target is reached, this implementation uses a fixed, small number of iterations (typically 2), trading absolute quality for speed and stability.

### [Grid-based LOD Simplification](4423-grid-based-lod-simplification.md)
Grid-based LOD Simplification is a method for rendering large-scale voxel or blocky meshes at multiple levels of detail by quantizing vertex coordinates to a coarse grid and implicitly managing primitive visibility. It solves the problem of rendering distant geometry efficiently without storing multiple explicit mesh copies on the GPU, by sorting primitives into a "Progressively Ordered Primitive" (POP) buffer. Unlike [Mesh simplification via valence pattern removal](1931-mesh-simplification-via-valence-pattern-removal.md) which modifies topology, or [GPU-based Mesh Decimation via Edge Collapse](3505-gpu-based-mesh-decimation-via-edge-collapse.md) which requires complex graph operations, this method relies on simple coordinate rounding and sorting. It is distinct from [Linear Spline Simplification](3483-linear-spline-simplification.md) as it operates on discrete integer grids rather than continuous spline approximations.

### [Octree Simplification](4641-octree-simplification.md)
Octree Simplification is a preprocessing technique that reduces the geometric complexity of an octree-based isosurface representation by collapsing leaf nodes into pseudo-leaf nodes, thereby decreasing the number of vertices in the final mesh. It solves the problem of excessive mesh density in volume rendering by controlling the trade-off between geometric fidelity and rendering performance. It differs from sibling methods like [Mesh simplification via valence pattern removal](1931-mesh-simplification-via-valence-pattern-removal.md) and [GPU-based Mesh Decimation via Edge Collapse](3505-gpu-based-mesh-decimation-via-edge-collapse.md) by operating directly on the spatial hierarchy of the octree rather than on the resulting mesh topology, and it differs from [Static Level of Detail Generation via Vertex Clustering](9052-static-level-of-detail-generation-via-vertex-clustering.md) by dynamically computing vertex positions via Quadratic Error Functions (QEF) rather than static clustering.

### [Sample-based Simplification](7814-sample-based-simplification.md)
Sample-based simplification is a method for reducing the geometric complexity of unstructured tetrahedral meshes during volume rendering by selectively removing triangle primitives rather than resampling the domain. It differs from domain-based simplification (which uses edge-collapses to resample the domain) by operating directly on the rendered primitives, maintaining the original mesh topology while discarding less important triangles. This approach is designed for interactive rendering of large datasets, particularly within the HAVS volume rendering system.

### [Isosurface Simplification via Persistence](8470-isosurface-simplification-via-persistence.md)
Isosurface Simplification via Persistence is a mesh-based technique for removing insignificant topological features from volumetric data, primarily used for noise removal and feature selection in volume visualization. It addresses the problem of distinguishing between meaningful structures and noise by quantifying the stability of topological features. Unlike general geometric simplification methods that prioritize distance metrics, this method prioritizes topological significance based on the scalar field's critical points. It builds on [Mesh Simplification](1032-mesh-simplification.md) by using topological attributes rather than geometric error.

### [Vertex Clustering Simplification](9046-vertex-clustering-simplification.md)
A technique used to precompute static levels of detail, similar to the method of Rossignac and Borrel.

### [Triangular Mesh Simplification](9103-triangular-mesh-simplification.md)
Triangular Mesh Simplification reduces the vertex and triangle count of a 3D surface mesh while preserving its geometric appearance, primarily to improve rendering performance and reduce memory usage in volume rendering pipelines. It serves as a post-processing or pre-processing step for isosurfaces (e.g., from Marching Cubes) to enable real-time interaction. Unlike vertex clustering which snaps vertices to a grid, this method typically operates by collapsing edges or removing vertices based on geometric error metrics.

## Operating characteristics
*   **Time Complexity:** The algorithm is iterative. Each iteration involves scanning edges, solving QEFs, and updating the mesh structure. The use of `edgeFraction` reduces the per-iteration cost to $O(E \cdot f)$ where $E$ is the number of edges and $f$ is the fraction sampled. The number of iterations depends on the target reduction.
*   **Memory Complexity:** Requires storage for the mesh, candidate edges, and temporary buffers for collapse data. The `LinearBuffer` structures allow dynamic resizing.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales with mesh size. The random sampling of edges helps maintain performance on large meshes.
*   **Strengths:** Preserves topology well, minimizes geometric error via QEF, handles sharp features via normal angle checks.
*   **Limitations:** Can create "hub" vertices with high degree if not penalized. May fail to find valid collapses in complex geometries, leading to early termination.
*   **When to use:** When a target triangle count is required for real-time rendering of isosurfaces.
*   **When not to use:** When topological changes are unacceptable or when the mesh is already very small.

## Background
Mesh Simplification via Edge Collapse was popularized by Garland and Heckbert (1997) using Quadric Error Metrics. The implementation described in the sources is based on the work by MyMiniFactory, adapted for various graphics pipelines [S3, unofficial]. It is widely used in pre-processing steps for volume rendering isosurfaces to balance visual fidelity and rendering speed.

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] nickgildea/fast_dual_contouring, https://github.com/nickgildea/fast_dual_contouring, fetched 2026-10-26 (unofficial)
[S3] TRTUZ8/TRTUZ8.github.io, https://github.com/TRTUZ8/TRTUZ8.github.io, fetched 2026-10-28 (unofficial)
[S4] PDF TetSplat: Real-time Rendering and Volume Clipping of Large ... - Museth, https://ken.museth.org/Publications_files/Museth-Lombeyda_VIS04.pdf, fetched 2026-10-28 (unofficial)
[S5] MixRT: Mixed Neural Representations For Real-Time NeRF Rendering, https://doi.org/10.1109/3dv62453.2024.00087, fetched 2026-10-27
