# Connected Component Filtering

Also called: Largest Connected Component Extraction, Mesh Post-Processing via Connected Component Filtering

## Overview
Connected Component Filtering segments a triangle mesh to retain only the largest connected component of faces, discarding isolated geometry artifacts and floaters [S1, S2, unofficial]. It solves the problem of disconnected fragments introduced during volume-to-mesh conversion, ensuring the output represents a single coherent object. Unlike [Geometry Cleaning](12700-geometry-cleaning.md), which focuses on topological validity and manifold constraints, this method isolates the primary object based on face adjacency connectivity [S1, S2, unofficial].

## How it works
**Inputs:** A triangle mesh object containing vertices, faces, and adjacency data.
**Outputs:** A filtered mesh containing only the vertices and faces belonging to the largest connected component.

**Data Structures:**
*   `mesh.faces`: Array of shape $(N_f, 3)$, indices of vertices forming each triangle.
*   `mesh.face_adjacency`: Graph structure or boolean matrix representing connectivity between faces sharing an edge.
*   `mesh.vertices`: Array of shape $(N_v, 3)$, vertex coordinates.

**Steps:**

1.  **Compute Connectivity Labels:**
    Calculate connected component labels for each face based on face adjacency.
    $$ \text{labels} = \text{connected\_component\_labels}(\text{mesh.face\_adjacency}) $$
    Where `labels` is an array of shape $(N_f)$, with integer values identifying the component ID for each face [S1, unofficial].

2.  **Identify Largest Component:**
    Count the occurrences of each label to determine component sizes.
    $$ \text{cc}, \text{cc\_cnt} = \text{unique}(\text{labels}, \text{return\_counts}=\text{True}) $$
    Identify the label corresponding to the maximum count:
    $$ \text{cc\_maxid} = \text{cc}[\text{argmax}(\text{cc\_cnt})] $$
    If component segmentation fails, the operation is skipped [S1, unofficial].

3.  **Filter Faces:**
    Create a boolean mask for faces belonging to the largest component:
    $$ \text{face\_mask} = (\text{labels} == \text{cc\_maxid}) $$
    Update the mesh to retain only these faces [S1, unofficial].

4.  **Filter Vertices:**
    Create a boolean vertex mask initialized to false for all vertices. Mark vertices as true if they are referenced by any face in the filtered set:
    $$ \text{vmask} = \text{zeros}([\text{len}(\text{mesh.vertices})], \text{dtype}=\text{bool}) $$
    $$ \text{vmask}[\text{mesh.faces}[\text{face\_mask}]] = 1 $$
    Update the mesh to retain only vertices present in the filtered faces [S1, unofficial].

**Alternative Implementation (Threshold-Based):**
Some implementations use a cluster-based approach with a minimum triangle threshold [S2, unofficial]:
1.  Compute triangle clusters and their sizes:
    $$ (\text{triangle\_clusters}, \text{cluster\_n\_triangles}, \text{cluster\_area}) = \text{cluster\_connected\_triangles}(\text{mesh}) $$
2.  Determine a threshold $n_{\text{cluster}}$ by sorting `cluster_n_triangles` and selecting the $k$-th largest value, or enforcing a minimum (e.g., 50 triangles) [S2, unofficial].
3.  Identify triangles to remove where their cluster size is less than $n_{\text{cluster}}$:
    $$ \text{triangles\_to\_remove} = \text{cluster\_n\_triangles}[\text{triangle\_clusters}] < n_{\text{cluster}} $$
4.  Remove identified triangles and clean up unreferenced vertices and degenerate triangles [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def connected_component_filtering(mesh):
    """
    Filters mesh to keep only the largest connected component of faces.
    Based on S1.
    """
    # Step 1: Compute connected component labels for faces
    # Assumes mesh.face_adjacency provides the graph structure
    labels = connected_component_labels(mesh.face_adjacency)
    
    # Step 2: Identify the largest component
    unique_labels, counts = np.unique(labels, return_counts=True)
    if len(unique_labels) == 0:
        print("Failed to segment largest cc")
        return mesh
    
    max_count_idx = np.argmax(counts)
    cc_maxid = unique_labels[max_count_idx]
    
    # Step 3: Filter faces
    face_mask = (labels == cc_maxid)
    
    # Get indices of faces to keep
    keep_face_indices = np.where(face_mask)[0]
    
    # Step 4: Filter vertices
    # Create vertex mask
    num_vertices = len(mesh.vertices)
    vmask = np.zeros(num_vertices, dtype=bool)
    
    # Mark vertices used by kept faces
    # mesh.faces is (N, 3) array of vertex indices
    kept_faces = mesh.faces[face_mask]
    vmask[kept_faces] = True
    
    # Update mesh
    # Note: Implementation details for mesh.update_faces and mesh.update_vertices
    # depend on the specific mesh library. Here we assume standard operations.
    mesh.update_faces(face_mask)
    mesh.update_vertices(vmask)
    
    return mesh
```

## Subtopics

### [Geometry Cleaning](12700-geometry-cleaning.md)
Focuses on topological validity, removing non-manifold geometry and degenerate cells, rather than isolating the largest object.

## Operating characteristics
*   **Time Complexity:** Dominated by the connected component analysis on the face adjacency graph, typically $O(N_f + E_f)$ where $N_f$ is the number of faces and $E_f$ is the number of face adjacencies [S1, unofficial].
*   **Memory Complexity:** Requires storage for labels ($N_f$ integers) and vertex masks ($N_v$ booleans) [S1, unofficial].
*   **Strengths:** Effectively removes noise and isolated fragments common in voxel-based extractions [S1, S2, unofficial].
*   **Limitations:** May discard valid small objects if they are disconnected from the main body [S2, unofficial]. Relies on accurate face adjacency computation [S1, unofficial].

## Background
Connected Component Filtering is a standard post-processing step in 3D reconstruction pipelines, particularly those involving voxel-based representations or implicit surfaces converted to triangle meshes. It is commonly used in conjunction with [Preprocessed Marching Cubes](3765-preprocessed-marching-cubes.md) to ensure the resulting mesh represents a single coherent object [S1, S2, unofficial].

## Sources
[S1] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S2] Zhaoyibinn/VGVGGT, https://github.com/Zhaoyibinn/VGVGGT, fetched 2026-10-27 (unofficial)
