# Connected Triangle Cluster Filtering

Also called: Cluster-based Floater Removal, Mesh Post-Processing

## Overview
Connected Triangle Cluster Filtering removes floating artifacts and disconnected mesh components by clustering triangles based on connectivity and discarding small clusters [S1, unofficial]. It serves as a post-processing step for isosurfaces extracted via methods like Marching Cubes, specifically addressing the issue of isolated geometry artifacts that persist after initial surface extraction [S1, unofficial]. Unlike [Connected Component Filtering](10789-connected-component-filtering.md), which typically operates on vertex connectivity or large-scale components, this method explicitly clusters individual triangles and uses a size-based threshold (triangle count) to determine validity, offering finer control over noise removal [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A triangular mesh (vertices and faces).
*   **Output:** A cleaned triangular mesh with small disconnected clusters removed.

**Data Structures**
*   **Mesh:** Defined by a vertex array $V$ of size $N_v \times 3$ and a face array $F$ of size $N_f \times 3$, where each row contains indices into $V$.
*   **Cluster Data:** Arrays mapping each face index to a cluster ID, the count of faces per cluster, and the area of each cluster.

**Parameters**
*   `cluster_to_keep`: An index parameter used to determine the threshold. The threshold is selected from the sorted list of cluster sizes [S1, unofficial].
*   `min_len`: An absolute minimum triangle count for a cluster to survive. In the reference implementation, this is typically 50, though an alternative function uses 1000 [S1, unofficial].

**Algorithm Steps**
1.  **Deep Copy:** Create a deep copy of the input mesh to preserve the original geometry [S1, unofficial].
2.  **Clustering:** Identify connected components of triangles. Two triangles are connected if they share a vertex. Assign a unique cluster ID to each connected group of triangles. Calculate the number of triangles and the total surface area for each cluster [S1, unofficial].
3.  **Threshold Determination:** Sort the clusters by their triangle count in ascending order. Select a threshold value based on the index `cluster_to_keep` in this sorted list [S1, unofficial].
4.  **Minimum Size Enforcement:** Ensure the threshold is at least `min_len` triangles. If the calculated threshold is less than `min_len`, use `min_len` as the effective threshold [S1, unofficial].
5.  **Mask Creation:** Create a boolean mask for all triangles. Mark triangles belonging to clusters with a triangle count strictly less than the effective threshold as `True` (to be removed) [S1, unofficial].
6.  **Triangle Removal:** Remove all triangles marked in the mask from the mesh [S1, unofficial].
7.  **Vertex Cleanup:** Identify vertices that are no longer referenced by any remaining triangle. Remove these orphaned vertices and re-index the face array to maintain consistency [S1, unofficial].
8.  **Degenerate Removal:** Remove degenerate triangles, defined as those with zero area or duplicate vertex indices within a single face [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def connected_triangle_cluster_filtering(mesh, cluster_to_keep, min_len=50):
    """
    Removes small disconnected triangle clusters from a mesh.
    
    Args:
        mesh: Object containing vertices (V) and faces (F).
        cluster_to_keep: Index in the sorted list of cluster sizes to determine the threshold.
        min_len: Absolute minimum number of triangles for a cluster to survive.
        
    Returns:
        Cleaned mesh object.
    """
    # Step 1: Deep copy
    cleaned_mesh = deep_copy(mesh)
    V = cleaned_mesh.vertices
    F = cleaned_mesh.faces
    
    # Step 2: Cluster connected triangles
    # Build adjacency graph where nodes are faces and edges exist if faces share a vertex
    # Perform connected components on this graph
    cluster_ids, cluster_counts, cluster_areas = compute_triangle_clusters(F)
    
    # Step 3: Determine threshold
    # Sort cluster counts ascending
    sorted_counts = sorted(cluster_counts)
    
    # Select threshold based on index. Handle index out of bounds.
    if cluster_to_keep < len(sorted_counts):
        threshold = sorted_counts[cluster_to_keep]
    else:
        threshold = sorted_counts[-1] if len(sorted_counts) > 0 else 0
        
    # Step 4: Enforce minimum size
    if threshold < min_len:
        threshold = min_len
        
    # Step 5: Create mask
    # Mask is True for triangles belonging to clusters smaller than threshold
    mask = cluster_counts[F] < threshold
    
    # Step 6: Remove triangles
    # Keep only faces where mask is False
    valid_faces = F[~mask]
    cleaned_mesh.faces = valid_faces
    
    # Step 7: Remove unreferenced vertices
    # Find unique vertex indices present in valid_faces
    used_vertices = np.unique(valid_faces)
    # Create mapping from old index to new index
    vertex_map = np.full((V.shape[0],), -1, dtype=int)
    vertex_map[used_vertices] = np.arange(len(used_vertices))
    
    # Update face indices
    cleaned_mesh.faces = vertex_map[valid_faces]
    # Update vertices
    cleaned_mesh.vertices = V[used_vertices]
    
    # Step 8: Remove degenerate triangles
    # Remove faces with duplicate vertices or zero area
    cleaned_mesh = remove_degenerate_triangles(cleaned_mesh)
    
    return cleaned_mesh

def compute_triangle_clusters(F):
    """
    Computes connected components of triangles sharing vertices.
    Returns cluster_ids (N_f,), cluster_counts (N_clusters,), cluster_areas (N_clusters,).
    """
    N_f = F.shape[0]
    cluster_ids = np.zeros(N_f, dtype=int)
    visited = np.zeros(N_f, dtype=bool)
    current_cluster = 0
    
    # Build adjacency implicitly or explicitly
    # For simplicity, we iterate through unvisited faces and flood fill
    # Note: This is a conceptual implementation. Efficient implementation uses Union-Find or BFS on face-vertex graph.
    
    # Using Union-Find for efficiency
    parent = np.arange(N_f)
    
    def find(i):
        path = []
        while parent[i] != i:
            path.append(i)
            i = parent[i]
        for node in path:
            parent[node] = i
        return i
    
    def union(i, j):
        root_i = find(i)
        root_j = find(j)
        if root_i != root_j:
            parent[root_i] = root_j
            
    # Connect faces sharing vertices
    # Map vertex index to list of face indices
    vertex_to_faces = {}
    for face_idx in range(N_f):
        for v_idx in F[face_idx]:
            if v_idx not in vertex_to_faces:
                vertex_to_faces[v_idx] = []
            vertex_to_faces[v_idx].append(face_idx)
            
    for v_idx, face_indices in vertex_to_faces.items():
        if len(face_indices) > 1:
            first = face_indices[0]
            for other in face_indices[1:]:
                union(first, other)
                
    # Assign cluster IDs
    unique_roots = {}
    cluster_id_map = {}
    cluster_count = 0
    
    for i in range(N_f):
        root = find(i)
        if root not in unique_roots:
            unique_roots[root] = cluster_id_map
            cluster_id_map[root] = cluster_count
            cluster_count += 1
        cluster_ids[i] = cluster_id_map[root]
        
    # Count triangles per cluster
    unique, counts = np.unique(cluster_ids, return_counts=True)
    cluster_counts = counts
    
    # Calculate area per cluster (placeholder, requires vertex positions)
    # cluster_areas = np.zeros(cluster_count)
    # for i in range(cluster_count):
    #     cluster_faces = np.where(cluster_ids == i)[0]
    #     # Compute area...
    
    return cluster_ids, cluster_counts, np.zeros(cluster_count) # Areas omitted for brevity as not critical for filtering logic

def remove_degenerate_triangles(mesh):
    """
    Removes faces with duplicate vertices or zero area.
    """
    F = mesh.faces
    V = mesh.vertices
    
    # Check for duplicate vertices in a face
    v0, v1, v2 = F[:, 0], F[:, 1], F[:, 2]
    valid = (v0 != v1) & (v1 != v2) & (v0 != v2)
    
    # Check for zero area (collinear vertices)
    # Vector math: cross product of two edges
    p0 = V[v0]
    p1 = V[v1]
    p2 = V[v2]
    edge1 = p1 - p0
    edge2 = p2 - p0
    cross = np.cross(edge1, edge2)
    area = np.linalg.norm(cross, axis=1)
    valid = valid & (area > 1e-9) # Threshold for zero area
    
    mesh.faces = F[valid]
    return mesh
```

## Subtopics

### Distance-based Face Pruning
A method to handle textureless surface areas by pruning triangle faces that are far from the inference results of the PM-MVS method and removing isolated components with small diameters [S2]. It addresses the failure of MVS methods to recover correct surfaces in textureless areas, which often results in convex hulls, concave envelopes, or isolated components [S2]. It is chosen when textureless regions are present and standard connectivity filtering is insufficient to remove artifacts arising from lack of texture [S2].

## Operating characteristics
*   **Time Complexity:** Dominated by the clustering step. Building the face-vertex adjacency and performing connected components is approximately $O(N_f \alpha(N_f))$ using Union-Find, where $N_f$ is the number of faces and $\alpha$ is the inverse Ackermann function [S1, unofficial].
*   **Memory Complexity:** $O(N_f)$ to store cluster IDs and masks [S1, unofficial].
*   **Performance:** In the PGSR implementation, the filtering step is part of a larger pipeline. The total pipeline time on the Tanks and Temples dataset ranges from 45 minutes to 1.2 hours depending on the variant, running on an NVIDIA RTX 3090 GPU [S1, unofficial]. The specific time for the filtering step alone is not isolated in the sources, but it is described as a post-processing step [S1, unofficial].
*   **Strengths:** Effective at removing small floating artifacts and noise clusters that are disconnected from the main mesh body [S1, unofficial]. Simple to implement and parameterize [S1, unofficial].
*   **Limitations:** Relies on the assumption that valid geometry forms larger connected components than noise [S1, unofficial]. May remove valid small objects if the threshold is set too high or if `min_len` is inappropriate for the scene scale [S1, unofficial]. Does not handle textureless artifacts directly; that is the domain of [Distance-based Face Pruning](#distance-based-face-pruning) [S2].

## Background
Connected Triangle Cluster Filtering is used in the PGSR (Patch-based Global Surface Reconstruction) pipeline for 3D reconstruction [S1, unofficial]. It is cited as a method to clean up meshes generated by the reconstruction process, specifically to remove isolated geometry artifacts [S1, unofficial]. The method is implemented in the `PGSR` repository by zju3dv [S1, unofficial].

## Sources
[S1] zju3dv/PGSR, https://github.com/zju3dv/PGSR, fetched 2026-10-27 (unofficial)
[S2] HelixSurf: A Robust and Efficient Neural Implicit Surface Learning of Indoor Scenes with Iterative Intertwined Regularization, https://arxiv.org/abs/2302.14340v2, fetched 2026-10-26
