# Mesh Post-Processing (Cluster Filtering)
Also called: Cluster-based Mesh Simplification

## Overview
This method filters out floating artifacts and disconnected components from a 3D mesh by clustering triangles and retaining only the largest connected clusters [S2, unofficial]. It serves as a cleanup step following [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md), specifically targeting small, noisy geometry that survives projection-based culling. Unlike the parent method which removes occluded or distant geometry, this method removes topologically isolated fragments based on triangle count [S1, S2, unofficial].

## How it works
**Inputs:** A triangle mesh (vertices and faces).
**Outputs:** A cleaned triangle mesh with small disconnected components removed.

**Data Structures:**
*   `mesh`: A mesh object containing vertices and faces.
*   `cluster_labels`: An array mapping each triangle index to a cluster ID.
*   `cluster_n_triangles`: A list or array of the number of triangles in each cluster.

**Steps:**
1.  **Cluster Identification:** Identify connected triangle clusters. This is typically done using a connected components algorithm on the dual graph of the mesh (where nodes are triangles and edges connect adjacent triangles) [S1, unofficial]. In practice, this is often implemented via library functions such as Open3D's `cluster_connected_triangles()` [S1, unofficial].
2.  **Threshold Determination:**
    *   Sort the cluster sizes (`cluster_n_triangles`) in descending order [S1, unofficial].
    *   Determine the size threshold based on two criteria:
        1.  The size of the $N$-th largest cluster, where $N$ is `cluster_to_keep` [S1, unofficial].
        2.  A minimum absolute size `min_cluster_size` [S1, unofficial].
    *   The effective threshold $T$ is the maximum of these two values:
        $$ T = \max(\text{size of } N\text{-th largest cluster}, \text{min\_cluster\_size}) $$
3.  **Mask Creation:** Create a boolean mask for triangles. A triangle is marked for removal if its cluster ID corresponds to a cluster size strictly less than $T$ [S1, unofficial].
4.  **Removal:**
    *   Remove triangles belonging to clusters smaller than the threshold using a function like `remove_triangles_by_mask` [S1, unofficial].
    *   Remove unreferenced vertices (vertices not part of any remaining triangle) [S1, unofficial].
    *   Remove degenerate triangles (triangles with zero area or invalid geometry) resulting from the removal process [S1, unofficial].

**Parameters:**
*   `cluster_to_keep`: Integer specifying the maximum number of largest connected clusters to retain. Default value is 1000 [S1, S2, unofficial].
*   `min_cluster_size`: Integer specifying the minimum number of triangles a cluster must have to be kept. Default value is 50 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def post_process_mesh(mesh, cluster_to_keep=1000, min_cluster_size=50):
    """
    Filters out floating artifacts and disconnected components from a 3D mesh.
    
    Args:
        mesh: A mesh object with vertices and faces.
        cluster_to_keep: Max number of largest clusters to retain.
        min_cluster_size: Minimum triangles per cluster to keep.
        
    Returns:
        Cleaned mesh.
    """
    # Step 1: Identify connected triangle clusters
    # Using Open3D's cluster_connected_triangles as per source [S1, unofficial]
    # Returns a list of cluster labels for each triangle
    cluster_labels = cluster_connected_triangles(mesh)
    
    # Step 2: Determine threshold
    # Count triangles per cluster
    cluster_counts = {}
    for label in cluster_labels:
        cluster_counts[label] = cluster_counts.get(label, 0) + 1
        
    # Get sizes and sort descending
    cluster_sizes = sorted(cluster_counts.values(), reverse=True)
    
    # Determine the size of the N-th largest cluster
    # If there are fewer clusters than cluster_to_keep, use the smallest cluster size
    if len(cluster_sizes) > cluster_to_keep:
        nth_largest_size = cluster_sizes[cluster_to_keep - 1]
    else:
        # If we keep all clusters, the threshold is effectively 0 or min_cluster_size
        nth_largest_size = 0
        
    # Threshold is the max of the N-th largest size and the minimum size
    threshold = max(nth_largest_size, min_cluster_size)
    
    # Step 3: Create mask for triangles to remove
    # A triangle is removed if its cluster size < threshold
    triangles_to_remove = []
    for i, label in enumerate(cluster_labels):
        if cluster_counts[label] < threshold:
            triangles_to_remove.append(i)
            
    # Step 4: Remove triangles and clean up
    # Remove triangles by mask
    mesh.remove_triangles_by_mask(triangles_to_remove)
    
    # Remove unreferenced vertices
    mesh.remove_unreferenced_vertices()
    
    # Remove degenerate triangles
    mesh.remove_degenerate_triangles()
    
    return mesh
```

**Edge Cases and Pitfalls:**
*   **Precision:** Ensure that the connected components algorithm correctly identifies adjacency. Floating point precision issues in vertex positions might cause adjacent triangles to be considered disconnected if they do not share vertices exactly. [G] Standard practice is to use a tolerance for vertex matching if the library does not handle it automatically.
*   **Boundaries:** If `cluster_to_keep` is larger than the total number of clusters, all clusters are considered for the size threshold, but the `min_cluster_size` still applies.
*   **Degenerate Input:** If the input mesh has no triangles, the function should return an empty mesh.

## Operating characteristics
*   **Time Complexity:** Dominated by the connected components step, which is linear in the number of triangles $O(T)$ for typical implementations using Union-Find or BFS/DFS on the dual graph [S1, unofficial]. Sorting cluster sizes is $O(C \log C)$ where $C$ is the number of clusters.
*   **Memory Complexity:** $O(T)$ to store cluster labels and counts.
*   **Strengths:** Effectively removes small, noisy artifacts that are topologically disconnected from the main geometry. Simple to implement using existing library functions.
*   **Limitations:** May remove valid small objects if they are below the `min_cluster_size` or if the number of clusters exceeds `cluster_to_keep`. It does not consider geometric significance, only topological connectivity and size.
*   **When to use:** After surface extraction or meshing from point clouds/voxels where noise and floating artifacts are common.
*   **When not to use:** When small objects are semantically important and should be preserved regardless of size.

## Background
This method is part of the post-processing pipeline for 3D Gaussian Splatting and related surface extraction techniques. It is specifically mentioned in the context of cleaning up meshes generated from 3D Gaussian splats [S1, S2, unofficial]. The approach leverages standard mesh processing libraries like Open3D [S1, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] Zhaoyibinn/VGVGGT, https://github.com/Zhaoyibinn/VGVGGT, fetched 2026-10-27 (unofficial)
