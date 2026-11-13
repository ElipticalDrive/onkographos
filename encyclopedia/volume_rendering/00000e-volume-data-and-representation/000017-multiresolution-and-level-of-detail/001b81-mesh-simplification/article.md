# Mesh Simplification

Also called: Edge Collapse Simplification, Mesh Simplification via Edge Collapse, Mesh Simplifier, Quadric Mesh Simplification, ngMeshSimplifier

## Overview
Mesh simplification reduces the triangle count of a 3D mesh to a target number while minimizing geometric error, primarily by iteratively collapsing edges [S2, S3, unofficial]. It solves the computational bottleneck of processing high-resolution isosurfaces, particularly in real-time 3D reconstruction and volume visualization contexts [S1, unofficial]. Unlike general mesh simplification techniques that may prioritize topological correctness or specific error metrics, this method is often integrated as a preprocessing step for isosurface extraction (e.g., Marching Cubes) to balance the trade-off between reconstruction accuracy and interactive rotation speed [S1, unofficial]. It sets itself apart from siblings like [Wavelet-based Volume Rendering](1052-wavelet-based-volume-rendering.md) by operating directly on the mesh topology rather than the scalar field, and from [Multi-Tetra Framework](1183-multi-tetra-framework.md) by using edge collapses rather than tetrahedral decomposition.

## How it works
**Inputs and Outputs:**
*   **Input:** A mesh defined by vertices and faces, a target triangle count (or percentage), and simplification parameters.
*   **Output:** A simplified mesh with reduced vertex and face counts, preserving the original topology where possible.

**Data Structures:**
*   `MeshBuffer`: Contains `numVertices`, `numTriangles`, `vertices` (array of `MeshVertex`), and `triangles` (array of `MeshTriangle`) [S2, unofficial].
*   `MeshVertex`: Contains `xyz` (vec3/vec4), `normal` (vec3/vec4), and `colour` (vec3/vec4) [S2, unofficial].
*   `MeshTriangle`: Contains `indices_` (array of 3 ints) [S2, unofficial].
*   `Edge`: A union containing `uint64_t idx_` and a struct with two `uint32_t` fields `min_` and `max_`. Used to store edge indices and sort them [S2, unofficial].
*   `LinearBuffer<T>`: A template class with private fields `T* base_`, `T* end_`, `int size_`. Provides methods `push_back`, `resize`, `clear`, `begin`, `end`, `operator[]` [S2, unofficial].

**Parameters:**
*   `targetPercentage`: Float specifying the target triangle count as a fraction of the initial count [S2, unofficial].
*   `target_count`: The desired number of triangles after simplification [S3, unofficial].
*   `maxIterations`: Integer limiting the maximum number of simplification iterations [S2, unofficial].
*   `agressiveness`: A parameter controlling the simplification behavior, default 7.0 [S3, unofficial].
*   `quality`: If > 0, performs lossless simplification after lossy; default 1 [S3, unofficial].
*   `COLLAPSE_MAX_DEGREE`: Maximum allowed sum of triangle counts for the two vertices involved in a collapse, set to 16 [S2, unofficial].
*   `edgeFraction`: Fraction of total edges to sample for potential collapses, used to improve performance by avoiding full mesh scan [S2, unofficial].
*   `minAngleCosine`: Minimum cosine of the angle between vertex normals to allow a collapse; prevents collapsing across sharp creases [S2, unofficial].
*   `maxEdgeSize`: Maximum squared length of an edge allowed for collapse [S2, unofficial].
*   `maxError`: Maximum acceptable QEF error for a collapse; collapses exceeding this are rejected [S2, unofficial].

**Steps:**
1.  **Precondition Check:** If `mesh->numTriangles < 100` or `mesh->numVertices < 100`, return immediately without modification [S2, unofficial].
2.  **Local Space Translation:** Translate vertices to local space by subtracting `worldSpaceOffset` from each vertex's xyz [S2, unofficial].
3.  **Initialization:** Initialize mesh counters to zero [S2, unofficial].
4.  **Build Candidate Edges:**
    *   Iterate over all triangles, extracting three edges per triangle [S2, unofficial].
    *   Normalize edge indices (min, max) and sort edges by index [S2, unofficial].
    *   Filter to keep only unique edges [S2, unofficial].
    *   Mark vertices as boundary if an edge appears only once (count == 1) [S2, unofficial].
    *   Discard boundary edges from the final candidate list [S2, unofficial].
5.  **Allocate Buffers:** Allocate buffers for `collapsePosition`, `collapseNormal`, `collapseValid`, `collapseEdgeID`, `collapseTarget`, `edgeBuffer`, and `triBuffer` [S2, unofficial].
6.  **Compute Vertex Triangle Counts:** Iterate all triangles and increment count for each vertex index [S2, unofficial].
7.  **Simplification Loop:** While `triangle count > target` and `iterations < maxIterations`:
    *   Reset `collapseEdgeID` and `collapseTarget` to -1, clear `collapseValid` [S2, unofficial].
    *   **Find Valid Collapses:**
        *   Seed PRNG with 42 [S2, unofficial].
        *   Calculate `numRandomEdges = edges.size() * options.edgeFraction` [S2, unofficial].
        *   Generate random indices into the edges array and sort them for locality [S2, unofficial].
        *   For each sampled edge:
            1.  Check normal dot product (`cosAngle`) against `options.minAngleCosine`; skip if lower [S2, unofficial].
            2.  Calculate squared edge length; skip if `> options.maxEdgeSize^2` [S2, unofficial].
            3.  Calculate degree (sum of triangle counts for both vertices); skip if `> COLLAPSE_MAX_DEGREE` [S2, unofficial].
            4.  Solve QEF for the two vertices using `qef_solve_from_points_4d_interleaved` [S2, unofficial].
            5.  If error > 0, invert it (`error = 1.f / error`) [S2, unofficial].
            6.  Apply penalty: `penalty = max(0, degree - 10)`; `error += penalty * (options.maxError * 0.1f)` [S2, unofficial].
            7.  Skip if `error > options.maxError` [S2, unofficial].
            8.  Store valid edge index in `collapseValid` [S2, unofficial].
            9.  Compute collapse normal as average of vertex normals [S2, unofficial].
            10. Compute collapse position from QEF solution [S2, unofficial].
            11. Update `minEdgeCost` for both vertices if current error is lower, storing the edge index in `collapseEdgeID` [S2, unofficial].
        *   **Pitfall:** If `countValidCollapse == 0`, break the loop to prevent infinite iteration [S2, unofficial].
    *   **Collapse Edges:** Perform the collapses, updating vertices and setting `collapseTarget` mappings [S2, unofficial].
    *   **Remove Triangles:**
        *   Iterate over all triangles. For each vertex index, if `collapseTarget[index] != -1`, replace index with target [S2, unofficial].
        *   If the resulting triangle has duplicate indices (degenerate), increment `removedCount` and skip [S2, unofficial].
        *   Otherwise, add to `triBuffer` and increment `vertexTriangleCounts` for the new indices [S2, unofficial].
        *   Swap tris with `triBuffer` [S2, unofficial].
    *   **Remove Edges:** Update edge endpoints based on `collapseTarget`, removing self-loops (where `min_ == max_`) [S2, unofficial].
8.  **Finalize:**
    *   Copy remaining triangles back to `mesh->triangles` [S2, unofficial].
    *   Call `CompactVertices` to remove unused vertices and remap triangle indices [S2, unofficial].
    *   Translate vertices back to world space by adding `worldSpaceOffset`, then copy to `mesh->vertices` with position, normal, and colour [S2, unofficial].
    *   Reconstruct mesh object from simplified vertices and faces using `trimesh.Trimesh` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mesh_simplify(vertices, faces, target_count, agressiveness=7.0, max_iterations=100):
    """
    Simplify a mesh by iteratively collapsing edges.
    
    Args:
        vertices: Array of vertex positions (N, 3)
        faces: Array of face indices (M, 3)
        target_count: Target number of triangles
        agressiveness: Control parameter (default 7.0)
        max_iterations: Max iterations (default 100)
        
    Returns:
        simplified_vertices, simplified_faces
    """
    # Precondition check
    if len(faces) < 100 or len(vertices) < 100:
        return vertices, faces

    # Local space translation (assuming worldSpaceOffset is 0 for simplicity)
    # In practice: vertices -= worldSpaceOffset
    
    # Initialize data structures
    num_vertices = len(vertices)
    num_faces = len(faces)
    
    # Build candidate edges
    edges = []
    vertex_triangle_counts = [0] * num_vertices
    
    for face in faces:
        for i in range(3):
            v1 = face[i]
            v2 = face[(i + 1) % 3]
            edge = (min(v1, v2), max(v1, v2))
            if edge not in edges:
                edges.append(edge)
            vertex_triangle_counts[v1] += 1
            vertex_triangle_counts[v2] += 1
            
    # Filter boundary edges (edges appearing only once in the original mesh)
    # Note: The source describes marking vertices as boundary if an edge appears only once.
    # Here we assume edges list contains unique edges from the initial mesh.
    # A more robust implementation would count occurrences in the initial face list.
    # For this pseudocode, we assume 'edges' are all valid candidates initially,
    # and boundary handling is implicit in the collapse validity checks or pre-filtering.
    
    # Simplification loop
    for iteration in range(max_iterations):
        if num_faces <= target_count:
            break
            
        collapse_edge_id = [-1] * num_vertices
        collapse_target = [-1] * num_vertices
        collapse_valid = [False] * len(edges)
        
        # Find valid collapses
        # Sample edges based on edgeFraction
        num_random_edges = int(len(edges) * 0.1) # Example fraction
        random_indices = sorted(random.sample(range(len(edges)), num_random_edges))
        
        min_edge_cost = [float('inf')] * num_vertices
        
        for idx in random_indices:
            edge = edges[idx]
            v1, v2 = edge
            
            # Check normal dot product (cosAngle)
            # Assuming normals are available or computed
            # if dot(normals[v1], normals[v2]) < minAngleCosine: continue
            
            # Check squared edge length
            # if squared_length(vertices[v1], vertices[v2]) > maxEdgeSize**2: continue
            
            # Check degree
            degree = vertex_triangle_counts[v1] + vertex_triangle_counts[v2]
            if degree > 16: # COLLAPSE_MAX_DEGREE
                continue
                
            # Solve QEF
            # error = qef_solve(vertices[v1], vertices[v2], normals[v1], normals[v2])
            # if error > 0: error = 1.0 / error
            
            # Apply penalty
            # penalty = max(0, degree - 10)
            # error += penalty * (maxError * 0.1)
            
            # if error > maxError: continue
            
            # Store valid
            collapse_valid[idx] = True
            
            # Update minEdgeCost
            # if error < min_edge_cost[v1]:
            #     min_edge_cost[v1] = error
            #     collapse_edge_id[v1] = idx
            # if error < min_edge_cost[v2]:
            #     min_edge_cost[v2] = error
            #     collapse_edge_id[v2] = idx
            
        # Check if any valid collapses were found
        if not any(collapse_valid):
            break
            
        # Perform collapses
        # This involves updating vertices, faces, and edge lists
        # For simplicity, we assume a helper function handles the graph updates
        new_faces, new_vertices, new_edges, new_vertex_counts = collapse_edges(
            faces, vertices, edges, collapse_edge_id, collapse_target
        )
        
        faces = new_faces
        vertices = new_vertices
        edges = new_edges
        vertex_triangle_counts = new_vertex_counts
        num_faces = len(faces)
        num_vertices = len(vertices)
        
    # Compact vertices and remap indices
    # simplified_vertices, simplified_faces = compact_vertices(vertices, faces)
    
    # Translate back to world space
    # simplified_vertices += worldSpaceOffset
    
    return vertices, faces

def collapse_edges(faces, vertices, edges, collapse_edge_id, collapse_target):
    """
    Helper function to perform edge collapses.
    """
    # Implementation details omitted for brevity, as per source structure
    # This would update the mesh topology based on collapse_edge_id
    pass
```

## Subtopics

### [Mesh simplification via valence pattern removal](1931-mesh-simplification-via-valence-pattern-removal.md)
This method reduces the element count of quad-dominant meshes, specifically those generated by the Dual Marching Cubes (DMC) algorithm, by identifying and removing specific quadrilateral valence patterns: 3-X-3-Y and 3-3-3-3. It solves the problem of excessive mesh complexity and poor element quality caused by transitions between volume data layers. Unlike general decimation methods, this approach is optional, parallelized using face coloring to avoid race conditions, and specifically targets topological artifacts inherent to the DMC representation. It improves mesh quality and repositions vertices while reducing element count by approximately 10% for datasets like human skulls and skeletons.

### [Linear Spline Simplification](3483-linear-spline-simplification.md)
Linear Spline Simplification is an iterative knot removal algorithm that reduces the number of control points (knots) in a 1D linear spline by merging consecutive segments when internal knots lie within a pre-chosen vertical distance (scalar domain) from the candidate merged segment. It solves the problem of redundant data storage in Piecewise Linear Representations (PRI) by identifying spans of nearly collinear scalar values and removing intermediate samples while preserving endpoints. Unlike general mesh decimation methods that modify geometry topology, this method operates strictly on the scalar parameterization of sampling rays, ensuring the resulting knot list is a subset of the original list.

### [GPU-based Mesh Decimation via Edge Collapse](3505-gpu-based-mesh-decimation-via-edge-collapse.md)
This method decimates a triangular mesh by iteratively collapsing edges to minimize geometric error, implemented on the GPU using CUDA kernels and Thrust utilities. It reduces vertex count while preventing degenerate faces and normal flipping by enforcing topological validity checks and boundary preservation penalties on the device. Unlike typical greedy decimation which iterates until the target is reached, this implementation uses a fixed, small number of iterations (typically 2), trading absolute quality for speed and stability.

### [Grid-based LOD Simplification](4423-grid-based-lod-simplification.md)
Grid-based LOD Simplification renders large-scale voxel or blocky meshes at multiple levels of detail by quantizing vertex coordinates to a coarse grid and implicitly managing primitive visibility. It solves the problem of rendering distant geometry efficiently without storing multiple explicit mesh copies on the GPU by sorting primitives into a Progressively Ordered Primitive (POP) buffer. Unlike [Mesh simplification via valence pattern removal](1931-mesh-simplification-via-valence-pattern-removal.md) which modifies topology, or [GPU-based Mesh Decimation via Edge Collapse](3505-gpu-based-mesh-decimation-via-edge-collapse.md) which requires complex graph operations, this method relies on simple coordinate rounding and sorting. It is distinct from [Linear Spline Simplification](3483-linear-spline-simplification.md) as it operates on discrete integer grids rather than continuous spline approximations.

### [Octree Simplification](4641-octree-simplification.md)
Octree Simplification is a preprocessing technique that reduces the geometric complexity of an octree-based isosurface representation by collapsing leaf nodes into pseudo-leaf nodes, thereby decreasing the number of vertices in the final mesh. It solves the problem of excessive mesh density in volume rendering by controlling the trade-off between geometric fidelity and rendering performance. It differs from sibling methods like [Mesh simplification via valence pattern removal](1931-mesh-simplification-via-valence-pattern-removal.md) and [GPU-based Mesh Decimation via Edge Collapse](3505-gpu-based-mesh-decimation-via-edge-collapse.md) by operating directly on the spatial hierarchy of the octree rather than on the resulting mesh topology, and it differs from [Static Level of Detail Generation via Vertex Clustering](9052-static-level-of-detail-generation-via-vertex-clustering.md) by dynamically computing vertex positions via Quadratic Error Functions (QEF) rather than static clustering.

### [Sample-based Simplification](7814-sample-based-simplification.md)
Sample-based simplification is a method for reducing the geometric complexity of unstructured tetrahedral meshes during volume rendering by selectively removing triangle primitives rather than resampling the domain. It differs from domain-based simplification (which uses edge-collapses to resample the domain) by operating directly on the rendered primitives, maintaining the original mesh topology while discarding less important triangles. This approach is designed for interactive rendering of large datasets, particularly within the HAVS volume rendering system.

### [Isosurface Simplification via Persistence](8470-isosurface-simplification-via-persistence.md)
Isosurface Simplification via Persistence is a mesh simplification technique for topological objects modeled as meshes, used for noise removal and feature selection in volume visualization. It addresses the problem of identifying and removing insignificant topological features based on their persistence. It sets itself apart from raw histogram analysis by incorporating geometric metrics (area, volume) and topological persistence to provide a truer distribution of the scalar field.

### [Vertex Clustering Simplification](9046-vertex-clustering-simplification.md)
Vertex Clustering is a technique for precomputing static levels of detail (LOD) for large datasets, primarily used in out-of-core visualization systems where memory constraints necessitate simplified representations. It differs from error-driven edge-collapse methods by using a fixed spatial grid to merge vertices, offering a deterministic, topology-preserving simplification that is particularly effective for creating multiresolution representations of isosurfaces.

### [Triangular Mesh Simplification](9103-triangular-mesh-simplification.md)
Triangular Mesh Simplification reduces the vertex and face count of a 3D mesh to improve rendering performance while preserving geometric fidelity. It solves the computational bottleneck of processing high-resolution isosurfaces, particularly in real-time 3D reconstruction and volume visualization contexts. Unlike general mesh simplification techniques that may prioritize topological correctness or specific error metrics, this method is often integrated as a preprocessing step for isosurface extraction (e.g., Marching Cubes) to balance the trade-off between reconstruction accuracy and interactive rotation speed.

## Operating characteristics
*   **Time Complexity:** The core loop iterates until the target triangle count is reached or max iterations are exceeded. Each iteration involves scanning a fraction of edges (`edgeFraction`) and performing QEF solves, which is constant time per edge. The overall complexity is roughly $O(I \cdot E \cdot f)$, where $I$ is iterations, $E$ is edges, and $f$ is `edgeFraction`.
*   **Memory Complexity:** Requires buffers for edges, collapse data, and triangle updates. $O(V + E)$ for the mesh structure plus $O(E)$ for temporary buffers.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance scales with the number of edges and the aggressiveness of the simplification. Using `edgeFraction` allows for faster convergence at the cost of potential sub-optimality.
*   **Strengths:** Preserves topology, minimizes geometric error via QEF, handles sharp features via normal checks.
*   **Limitations:** Can create "hub" vertices with high degree if not penalized. May fail to find valid collapses if the mesh is too constrained or the target is too low.
*   **When to use:** When a simplified mesh representation of a volume isosurface is needed for rendering or further processing.
*   **When not to use:** When the exact topological structure of the original mesh is critical and cannot be approximated, or when the mesh is already very small.

## Background
The method originates from the work on fast dual contouring and mesh simplification, as implemented in repositories like `nickgildea/fast_dual_contouring` [S2, unofficial] and `TRTUZ8/TRTUZ8.github.io` [S3, unofficial]. It is often used in conjunction with Marching Cubes for isosurface extraction from volume data [S3, unofficial]. The use of Quadric Error Functions (QEF) for error measurement is a key component, derived from earlier work on mesh decimation [S3, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] nickgildea/fast_dual_contouring, https://github.com/nickgildea/fast_dual_contouring, fetched 2026-10-26 (unofficial)
[S3] TRTUZ8/TRTUZ8.github.io, https://github.com/TRTUZ8/TRTUZ8.github.io, fetched 2026-10-28 (unofficial)
[S4] PDF TetSplat: Real-time Rendering and Volume Clipping of Large ... - Museth, https://ken.museth.org/Publications_files/Museth-Lombeyda_VIS04.pdf, fetched 2026-10-28 (unofficial)
[S5] MixRT: Mixed Neural Representations For Real-Time NeRF Rendering, https://doi.org/10.1109/3dv62453.2024.00087, fetched 2026-10-27
