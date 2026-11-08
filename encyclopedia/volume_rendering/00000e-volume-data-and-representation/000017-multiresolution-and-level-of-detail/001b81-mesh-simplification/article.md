# Mesh Simplification

Also called: Edge Collapse Simplification, Mesh Simplification via Edge Collapse, Mesh Simplifier, Quadric Mesh Simplification, ngMeshSimplifier

## Overview
Mesh Simplification is a method that reduces the triangle count of a surface mesh to a target size while minimizing geometric error, primarily through iterative edge collapses [S2, S3, unofficial]. It solves the problem of excessive mesh complexity in volume rendering isosurfaces, enabling efficient rendering of large datasets [S2, S3, unofficial]. Unlike [Wavelet-based Volume Rendering](1052-wavelet-based-volume-rendering.md) which operates on frequency domains, or [Multi-Tetra Framework](1183-multi-tetra-framework.md) which uses tetrahedral primitives, this method operates directly on the triangular mesh topology [S3, unofficial]. It is distinct from [Grid-based LOD Simplification](4423-grid-based-lod-simplification.md) as it modifies topology via edge collapses rather than quantizing coordinates, and from [Linear Spline Simplification](3483-linear-spline-simplification.md) which merges spline segments [S2, S3, unofficial].

## How it works
**Inputs:**
- `vertices`: Array of vertex positions (and optionally normals/colors).
- `faces`: Array of triangle indices (triplets of vertex indices).
- `target_count`: Desired number of triangles after simplification [S3, unofficial].
- `options`: Configuration parameters including `targetPercentage` (fraction of initial triangles), `maxIterations`, `edgeFraction`, `minAngleCosine`, `maxEdgeSize`, `maxError`, and `agressiveness` [S2, S3, unofficial].

**Outputs:**
- Simplified `vertices` and `faces` arrays [S2, unofficial].

**Data Structures:**
- `MeshBuffer`: Contains `numVertices`, `numTriangles`, `vertices` (array of `MeshVertex`), and `triangles` (array of `MeshTriangle`) [S2, unofficial].
- `MeshVertex`: Contains `xyz` (vec3), `normal` (vec3), and `colour` (vec3) [S2, unofficial].
- `MeshTriangle`: Contains `indices_` (array of 3 ints) [S2, unofficial].
- `Edge`: Union containing `idx_` (uint64_t) and a struct with `min_` and `max_` (uint32_t) for sorting and storage [S2, unofficial].
- `LinearBuffer<T>`: Template class with `base_`, `end_`, `size_` fields, providing `push_back`, `resize`, `clear`, `begin`, `end`, `operator[]` [S2, unofficial].
- `collapseValid`: Boolean array indicating valid collapses for each edge [S2, unofficial].
- `collapseEdgeID`: Array storing the best edge index for each vertex [S2, unofficial].
- `collapseTarget`: Array mapping collapsed vertices to their target vertex [S2, unofficial].
- `collapsePosition`: Array storing the computed position for each collapse [S2, unofficial].
- `collapseNormal`: Array storing the average normal for each collapse [S2, unofficial].
- `vertexTriangleCounts`: Array counting triangles per vertex [S2, unofficial].

**Parameters:**
- `targetPercentage`: Float specifying target triangle count as a fraction of initial count [S2, unofficial].
- `maxIterations`: Integer limiting simplification iterations [S2, unofficial].
- `edgeFraction`: Fraction of total edges to sample for potential collapses (performance optimization) [S2, unofficial].
- `minAngleCosine`: Minimum cosine of the angle between vertex normals to allow a collapse (prevents collapsing across sharp creases) [S2, unofficial].
- `maxEdgeSize`: Maximum squared length of an edge allowed for collapse [S2, unofficial].
- `maxError`: Maximum acceptable Quadratic Error Function (QEF) error for a collapse [S2, unofficial].
- `agressiveness`: Parameter controlling simplification behavior (default 7.0) [S3, unofficial].
- `quality`: If > 0, performs lossless simplification after lossy (default 1) [S3, unofficial].
- `COLLAPSE_MAX_DEGREE`: Maximum allowed sum of triangle counts for the two vertices involved in a collapse (set to 16) [S2, unofficial].

**Steps:**
1. **Precondition Check:** If `mesh->numTriangles < 100` or `mesh->numVertices < 100`, return immediately without modification [S2, unofficial].
2. **Local Space Translation:** Translate vertices to local space by subtracting `worldSpaceOffset` from each vertex's xyz [S2, unofficial].
3. **Initialization:** Initialize mesh counters to zero [S2, unofficial].
4. **Build Candidate Edges:**
   - Iterate over all triangles, extracting three edges per triangle [S2, unofficial].
   - Normalize edge indices (min, max) and sort edges by index [S2, unofficial].
   - Filter to keep only unique edges [S2, unofficial].
   - Mark vertices as boundary if an edge appears only once (count == 1) [S2, unofficial].
   - Discard boundary edges from the final candidate list [S2, unofficial].
5. **Allocate Buffers:** Allocate buffers for `collapsePosition`, `collapseNormal`, `collapseValid`, `collapseEdgeID`, `collapseTarget`, `edgeBuffer`, and `triBuffer` [S2, unofficial].
6. **Compute Vertex Triangle Counts:** Iterate all triangles and increment count for each vertex index in `vertexTriangleCounts` [S2, unofficial].
7. **Simplification Loop:** Loop while `triangle count > target` and `iterations < maxIterations`:
   - Reset `collapseEdgeID` and `collapseTarget` to -1, clear `collapseValid` [S2, unofficial].
   - **Find Valid Collapses:**
     - Seed PRNG with 42 [S2, unofficial].
     - Calculate `numRandomEdges = edges.size() * options.edgeFraction` [S2, unofficial].
     - Generate random indices into the edges array [S2, unofficial].
     - Sort random indices for locality [S2, unofficial].
     - For each sampled edge:
       1. Check normal dot product (`cosAngle`) against `options.minAngleCosine`; skip if lower [S2, unofficial].
       2. Calculate squared edge length; skip if `> options.maxEdgeSize^2` [S2, unofficial].
       3. Calculate degree (sum of triangle counts for both vertices); skip if `> COLLAPSE_MAX_DEGREE` [S2, unofficial].
       4. Solve QEF for the two vertices using `qef_solve_from_points_4d_interleaved` [S2, unofficial].
       5. If error > 0, invert it (`error = 1.f / error`) [S2, unofficial].
       6. Apply penalty: `penalty = max(0, degree - 10)`; `error += penalty * (options.maxError * 0.1f)` [S2, unofficial].
       7. Skip if `error > options.maxError` [S2, unofficial].
       8. Store valid edge index in `collapseValid` [S2, unofficial].
       9. Compute collapse normal as average of vertex normals [S2, unofficial].
       10. Compute collapse position from QEF solution [S2, unofficial].
       11. Update `minEdgeCost` for both vertices if current error is lower, storing the edge index in `collapseEdgeID` [S2, unofficial].
   - **Collapse Edges:** Perform the collapses, updating vertices and setting `collapseTarget` mappings [S2, unofficial].
   - **Remove Triangles:**
     - Iterate over all triangles [S2, unofficial].
     - For each vertex index in the triangle, if `collapseTarget[index] != -1`, replace index with target [S2, unofficial].
     - If the resulting triangle has duplicate indices (degenerate), increment `removedCount` and skip [S2, unofficial].
     - Otherwise, add to `triBuffer` and increment `vertexTriangleCounts` for the new indices [S2, unofficial].
     - Swap tris with `triBuffer` [S2, unofficial].
   - **Remove Edges:** Update edge endpoints based on `collapseTarget`, removing self-loops (where `min_ == max_`) [S2, unofficial].
   - **Pitfall Check:** If no valid collapses are found (`countValidCollapse == 0`), break the loop to prevent infinite iteration [S2, unofficial].
8. **Copy Triangles:** Copy remaining triangles back to `mesh->triangles` [S2, unofficial].
9. **Compact Vertices:** Remove unused vertices and remap triangle indices [S2, unofficial].
10. **World Space Translation:** Translate vertices back to world space by adding `worldSpaceOffset`, then copy to `mesh->vertices` with position, normal, and colour [S2, unofficial].
11. **Reconstruction:** Reconstruct mesh object from simplified vertices and faces using `trimesh.Trimesh` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mesh_simplify(vertices, faces, target_count, options):
    # Precondition check
    if len(faces) < 100 or len(vertices) < 100:
        return vertices, faces

    # Local space translation
    vertices = vertices - options.worldSpaceOffset

    # Build candidate edges
    edges = []
    vertex_triangle_counts = [0] * len(vertices)
    for tri in faces:
        for i in range(3):
            v1 = tri[i]
            v2 = tri[(i + 1) % 3]
            edge = (min(v1, v2), max(v1, v2))
            edges.append(edge)
            vertex_triangle_counts[v1] += 1
            vertex_triangle_counts[v2] += 1

    # Sort and unique edges
    edges = sorted(list(set(edges)))
    
    # Identify boundary edges (edges appearing only once in original face list)
    # Note: The source implies filtering boundary edges from candidates.
    # We need to count occurrences in the original face list to identify boundaries.
    edge_counts = {}
    for tri in faces:
        for i in range(3):
            v1 = tri[i]
            v2 = tri[(i + 1) % 3]
            edge = (min(v1, v2), max(v1, v2))
            edge_counts[edge] = edge_counts.get(edge, 0) + 1
            
    candidate_edges = [e for e in edges if edge_counts[e] > 1]

    # Initialize buffers
    collapse_valid = [False] * len(candidate_edges)
    collapse_edge_id = [-1] * len(vertices)
    collapse_target = [-1] * len(vertices)
    collapse_position = [None] * len(vertices)
    collapse_normal = [None] * len(vertices)
    tri_buffer = []

    max_iterations = options.maxIterations
    target_triangles = target_count
    
    for iteration in range(max_iterations):
        # Reset buffers
        collapse_edge_id = [-1] * len(vertices)
        collapse_target = [-1] * len(vertices)
        collapse_valid = [False] * len(candidate_edges)
        
        # Find valid collapses
        num_random_edges = int(len(candidate_edges) * options.edgeFraction)
        random_indices = sorted(random.sample(range(len(candidate_edges)), num_random_edges))
        
        min_edge_cost = float('inf')
        best_edge_idx = -1
        
        for idx in random_indices:
            edge = candidate_edges[idx]
            v1, v2 = edge
            
            # Check normal dot product
            cos_angle = dot(vertices[v1].normal, vertices[v2].normal)
            if cos_angle < options.minAngleCosine:
                continue
                
            # Check edge length
            edge_len_sq = squared_length(vertices[v1].xyz - vertices[v2].xyz)
            if edge_len_sq > options.maxEdgeSize ** 2:
                continue
                
            # Check degree
            degree = vertex_triangle_counts[v1] + vertex_triangle_counts[v2]
            if degree > COLLAPSE_MAX_DEGREE:
                continue
                
            # Solve QEF
            # qef_solve_from_points_4d_interleaved returns position and error
            pos, error = qef_solve_from_points_4d_interleaved([vertices[v1].xyz, vertices[v2].xyz], 
                                                              [vertices[v1].normal, vertices[v2].normal])
            
            if error > 0:
                error = 1.0 / error
                
            # Apply penalty
            penalty = max(0, degree - 10)
            error += penalty * (options.maxError * 0.1)
            
            if error > options.maxError:
                continue
                
            collapse_valid[idx] = True
            collapse_position[v1] = pos
            collapse_normal[v1] = average_normal([vertices[v1].normal, vertices[v2].normal])
            
            if error < min_edge_cost:
                min_edge_cost = error
                best_edge_idx = idx
                # Store best edge for both vertices
                collapse_edge_id[v1] = idx
                collapse_edge_id[v2] = idx

        # Check if any valid collapses were found
        if not any(collapse_valid):
            break
            
        # Perform collapses
        # Note: The source describes updating vertices and setting collapseTarget mappings.
        # We need to apply the best collapses.
        # The source implies that collapseEdgeID stores the edge index for each vertex.
        # We need to resolve conflicts if multiple edges are marked for a vertex.
        # For simplicity, we assume the first valid edge found for a vertex is used, 
        # or the one with the lowest cost. The source says "Update minEdgeCost for both vertices 
        # if current error is lower, storing the edge index in collapseEdgeID".
        
        # Apply collapses
        for i, edge in enumerate(candidate_edges):
            if collapse_valid[i]:
                v1, v2 = edge
                # Collapse v2 into v1
                collapse_target[v2] = v1
                # Update vertex position to the QEF solution
                vertices[v1].xyz = collapse_position[v1]
                vertices[v1].normal = collapse_normal[v1]
                
        # Remove triangles
        new_faces = []
        for tri in faces:
            new_indices = []
            for v_idx in tri:
                if collapse_target[v_idx] != -1:
                    new_indices.append(collapse_target[v_idx])
                else:
                    new_indices.append(v_idx)
            
            # Check for degenerate triangles
            if len(set(new_indices)) == 3:
                new_faces.append(new_indices)
                # Update vertex triangle counts for new indices
                for v_idx in new_indices:
                    vertex_triangle_counts[v_idx] += 1
            else:
                # Degenerate triangle removed
                pass
                
        faces = new_faces
        
        # Remove edges
        # Update edge endpoints based on collapseTarget
        new_edges = []
        for edge in candidate_edges:
            v1, v2 = edge
            if collapse_target[v1] != -1:
                v1 = collapse_target[v1]
            if collapse_target[v2] != -1:
                v2 = collapse_target[v2]
            if v1 != v2:
                new_edges.append((min(v1, v2), max(v1, v2)))
        candidate_edges = sorted(list(set(new_edges)))
        
        # Compact vertices
        # This step is handled by the final copy and remapping
        pass

    # Copy remaining triangles back
    # (Already done in the loop)
    
    # Compact vertices
    # Remove unused vertices and remap triangle indices
    # This is a standard vertex compaction step
    used_vertices = set()
    for tri in faces:
        for v_idx in tri:
            used_vertices.add(v_idx)
            
    # Create mapping from old index to new index
    old_to_new = {}
    new_vertices = []
    for i, v in enumerate(vertices):
        if i in used_vertices:
            old_to_new[i] = len(new_vertices)
            new_vertices.append(v)
            
    # Remap faces
    new_faces = []
    for tri in faces:
        new_tri = [old_to_new[v_idx] for v_idx in tri]
        new_faces.append(new_tri)
        
    # Translate back to world space
    new_vertices = [v + options.worldSpaceOffset for v in new_vertices]
    
    return new_vertices, new_faces
```

## Subtopics

### [Mesh simplification via valence pattern removal](1931-mesh-simplification-via-valence-pattern-removal.md)
This method reduces the element count of quad-dominant meshes, specifically those generated by the Dual Marching Cubes (DMC) algorithm, by identifying and removing specific quadrilateral valence patterns: **3-X-3-Y** and **3-3-3-3**. It solves the problem of excessive mesh complexity and poor element quality caused by transitions between volume data layers. Unlike general decimation methods, this approach is optional, parallelized using face coloring to avoid race conditions, and specifically targets topological artifacts inherent to the DMC representation. It improves mesh quality and repositions vertices while reducing element count by approximately 10% for datasets like human skulls and skeletons.

### [Linear Spline Simplification](3483-linear-spline-simplification.md)
Linear Spline Simplification is an iterative algorithm that reduces the number of knots in a piecewise linear spline by merging consecutive segments when internal knots lie within a specified vertical distance (scalar error) from the candidate merged segment. It is used in the context of Piecewise Rational Interpolation (PRI) for unstructured grid volume rendering to identify spans of nearly collinear scalar values, thereby removing redundant internal samples while preserving endpoints. Unlike general mesh decimation methods that may introduce new vertices, this method ensures the resulting knot list is a strict subset of the original list.

### [GPU-based Mesh Decimation via Edge Collapse](3505-gpu-based-mesh-decimation-via-edge-collapse.md)
This method decimates a triangular mesh by iteratively collapsing edges to minimize geometric error, implemented on the GPU using CUDA kernels and Thrust utilities. It reduces vertex count while preventing degenerate faces and normal flipping by enforcing topological validity checks and boundary preservation penalties on the device. Unlike typical greedy decimation which iterates until the target is reached, this implementation uses a fixed, small number of iterations (typically 2), trading absolute quality for speed and stability.

### [Grid-based LOD Simplification](4423-grid-based-lod-simplification.md)
Grid-based LOD Simplification is a method for rendering large-scale voxel or blocky meshes at multiple levels of detail by quantizing vertex coordinates to a coarse grid and implicitly managing primitive visibility. It solves the problem of rendering distant geometry efficiently without storing multiple explicit mesh copies on the GPU, by sorting primitives into a "Progressively Ordered Primitive" (POP) buffer. Unlike [Mesh simplification via valence pattern removal](1931-mesh-simplification-via-valence-pattern-removal.md) which modifies topology, or [GPU-based Mesh Decimation via Edge Collapse](3505-gpu-based-mesh-decimation-via-edge-collapse.md) which requires complex graph operations, this method relies on simple coordinate rounding and sorting. It is distinct from [Linear Spline Simplification](3483-linear-spline-simplification.md) as it operates on discrete integer grids rather than continuous spline approximations.

### [Octree Simplification](4641-octree-simplification.md)
Octree Simplification is a preprocessing technique that reduces the geometric complexity of an octree-based isosurface representation by collapsing leaf nodes into pseudo-leaf nodes, thereby decreasing the number of vertices in the final mesh. It solves the problem of excessive mesh density in volume rendering by controlling the trade-off between geometric fidelity and rendering performance. It differs from sibling methods like [Mesh simplification via valence pattern removal](1931-mesh-simplification-via-valence-pattern-removal.md) and [GPU-based Mesh Decimation via Edge Collapse](3505-gpu-based-mesh-decimation-via-edge-collapse.md) by operating directly on the spatial hierarchy of the octree rather than on the resulting mesh topology, and it differs from [Static Level of Detail Generation via Vertex Clustering](9052-static-level-of-detail-generation-via-vertex-clustering.md) by dynamically computing vertex positions via Quadratic Error Functions (QEF) rather than static clustering.

### [Sample-based Simplification](7814-sample-based-simplification.md)
Sample-based simplification is a method for reducing the geometric complexity of unstructured tetrahedral meshes during volume rendering by selectively removing triangle primitives rather than resampling the domain. It differs from domain-based simplification (which uses edge-collapses to resample the domain) by operating directly on the rendered primitives, maintaining the original mesh topology while discarding less important triangles. This approach is designed for interactive rendering of large datasets, particularly within the HAVS volume rendering system.

### [Isosurface Simplification via Persistence](8470-isosurface-simplification-via-persistence.md)
A simplification technique for topological objects modeled as meshes, used for noise removal and feature selection in volume visualization. It addresses the problem of identifying and removing insignificant topological features based on their persistence.

### [Triangular Mesh Simplification](9103-triangular-mesh-simplification.md)

## Operating characteristics
- **Time Complexity:** The algorithm iterates until the target triangle count is reached or max iterations are exceeded. Each iteration involves scanning a fraction of edges (`edgeFraction`), solving QEFs, and updating the mesh. The complexity is roughly $O(I \cdot E \cdot f)$, where $I$ is iterations, $E$ is edges, and $f$ is `edgeFraction` [S2, unofficial].
- **Memory Complexity:** Requires buffers for edges, collapse data, and triangle updates. The memory usage is proportional to the number of vertices and edges in the mesh [S2, unofficial].
- **Measured Performance:** The method is implemented in C++ and can be called via Cython or WASM [S1, S3, unofficial]. Performance is influenced by `edgeFraction` and `maxIterations` [S2, unofficial].
- **Scaling:** The method scales with mesh size, but the use of `edgeFraction` helps maintain performance on large meshes by sampling edges rather than scanning all [S2, unofficial].
- **Strengths:** Preserves topology, minimizes geometric error using QEF, and allows for control over simplification aggressiveness [S2, S3, unofficial].
- **Limitations:** Can create vertices with very high degree (hub formation), which is mitigated by penalties [S2, unofficial]. May not be suitable for meshes with sharp features if `minAngleCosine` is not set appropriately [S2, unofficial].
- **When to use:** When a triangular mesh representation of a volume is needed, and reducing polygon count is required for performance [S2, S3, unofficial].
- **When not to use:** When preserving exact topology or sharp features is critical without parameter tuning, or when the mesh is already very small [S2, unofficial].

## Background
Mesh Simplification via Edge Collapse was popularized by Garland and Heckbert in 1997 with their paper "Surface Simplification Using Quadric Error Metrics" [S2, S3, unofficial]. The method uses Quadratic Error Functions (QEF) to measure the geometric error of collapsing an edge. It has been widely adopted in computer graphics for level of detail generation and mesh compression. The specific implementation described here is based on the "Fast Quadric Mesh Simplification" algorithm, which has been ported to various platforms including WebAssembly [S3, unofficial]. It is used in volume rendering to simplify isosurfaces generated from volume data, such as those from medical imaging [S1, S3, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] nickgildea/fast_dual_contouring, https://github.com/nickgildea/fast_dual_contouring, fetched 2026-10-26 (unofficial)
[S3] TRTUZ8/TRTUZ8.github.io, https://github.com/TRTUZ8/TRTUZ8.github.io, fetched 2026-10-28 (unofficial)
[S4] PDF TetSplat: Real-time Rendering and Volume Clipping of Large ... - Museth, https://ken.museth.org/Publications_files/Museth-Lombeyda_VIS04.pdf, fetched 2026-10-28 (unofficial)
[S5] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
