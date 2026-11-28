# Constrained Laplacian Fairing

Also called: Constrained fairing, Dual-grid constrained smoothing

## Overview
Constrained Laplacian Fairing is a mesh smoothing technique for isosurfaces extracted from segmented volumes, specifically designed to reduce geometric noise while strictly bounding positional error relative to the source grid. It differs from standard Laplacian smoothing by constraining vertex movement to the dual-grid cell of origin, preventing the mesh from drifting arbitrarily far from the extracted topology. It serves as the primary smoothing step for the SurfaceNet extraction method, addressing the jaggedness inherent in non-manifold mesh extraction from multi-material volumes.

## How it works
The method operates on a mesh composed of quadrilateral faces, where most vertices have valences between three and six. The algorithm proceeds in two main phases: an unconstrained centroid-based averaging pass, followed by a constrained relaxation pass.

### Inputs and Outputs
- **Input**: A mesh $M$ extracted from a volumetric grid. Each vertex $v$ is associated with an initial dual-grid cell $C_v$ (the voxel in which the vertex was created). The mesh topology is defined by face connectivity.
- **Output**: A smoothed mesh $M'$ where vertex positions are updated.

### Data Structures
- **Mesh**: Defined by a list of vertices and a list of faces. Each face is defined by an ordered list of vertex indices.
- **Dual-Grid Mapping**: A mapping from each vertex index to its initial grid cell coordinates $(x, y, z)$. This is required to enforce the constraint.

### Algorithm Steps

1.  **Compute Face Centroids**:
    For every face $f$ in the mesh, compute its centroid $c_f$. If a face has vertices $v_1, v_2, \dots, v_k$, the centroid is:
    $$ c_f = \frac{1}{k} \sum_{i=1}^{k} v_i $$
    For quadrilateral faces ($k=4$), this is the arithmetic mean of the four vertex positions.

2.  **Compute Unconstrained Vertex Updates**:
    For every vertex $v$, compute a new position $v_{new}$ as the centroid of the centroids of all incident faces. To avoid storing explicit vertex-to-face adjacency lists, traverse the faces:
    - Initialize a sum vector $S = \mathbf{0}$ and a counter $N = 0$.
    - For each face $f$ in the mesh:
        - If $v$ is a vertex of $f$:
            - $S \leftarrow S + c_f$
            - $N \leftarrow N + 1$
    - If $N > 0$:
        - $v_{new} = \frac{1}{N} S$
    - Else:
        - $v_{new} = v$ (no change if vertex is isolated)

3.  **Apply Constraint (Constrained Fairing)**:
    For each vertex $v$, clamp its new position $v_{new}$ to remain within its initial dual-grid cell $C_v$.
    - Let $C_v$ be defined by the grid bounds $[x_{min}, x_{max}] \times [y_{min}, y_{max}] \times [z_{min}, z_{max}]$.
    - The maximum movement is bounded by half the width of a grid cell. Specifically, the vertex must remain in the cell where it was created.
    - For each coordinate axis $d \in \{x, y, z\}$:
        - $v_{new, d} = \text{clamp}(v_{new, d}, \text{bound}_{min, d}, \text{bound}_{max, d})$
    - Update $v \leftarrow v_{new}$.

4.  **Iterate**:
    Repeat steps 1–3 for a fixed number of iterations, typically five [S1].

5.  **Final Pass (Spike Removal)**:
    In the last iteration, drop the constraint. Allow vertices to move outside their originating dual-grid cells. This step avoids the formation of spikes that can occur if vertices are strictly confined to small cells during the final relaxation. The error remains bounded because all incident faces are located in the 27-neighborhood of the dual-grid voxel [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def constrained_laplacian_fairing(mesh, num_iterations=5):
    """
    mesh: object with attributes .vertices (list of Vec3), .faces (list of list of int)
    Returns: updated mesh.vertices
    """
    n_verts = len(mesh.vertices)
    n_faces = len(mesh.faces)
    
    # Precompute initial dual-grid cell bounds for each vertex
    # This requires external knowledge of the extraction grid.
    # Assume get_initial_cell_bounds(v_idx) returns (min_x, max_x, min_y, max_y, min_z, max_z)
    cell_bounds = [get_initial_cell_bounds(i) for i in range(n_verts)]

    for iteration in range(num_iterations):
        is_last_pass = (iteration == num_iterations - 1)
        
        # Step 1: Compute face centroids
        face_centroids = []
        for face in mesh.faces:
            centroid = Vec3(0, 0, 0)
            for v_idx in face:
                centroid += mesh.vertices[v_idx]
            centroid /= len(face)
            face_centroids.append(centroid)
            
        # Step 2: Compute new vertex positions as centroid of incident face centroids
        new_vertices = [Vec3(0, 0, 0) for _ in range(n_verts)]
        counts = [0] * n_verts
        
        # Traverse faces to accumulate contributions
        for face_idx, face in enumerate(mesh.faces):
            fc = face_centroids[face_idx]
            for v_idx in face:
                new_vertices[v_idx] += fc
                counts[v_idx] += 1
                
        # Average
        for i in range(n_verts):
            if counts[i] > 0:
                new_vertices[i] /= counts[i]
            else:
                new_vertices[i] = mesh.vertices[i]
                
        # Step 3 & 4: Apply constraints and update
        for i in range(n_verts):
            if not is_last_pass:
                # Constrain to initial dual-grid cell
                bounds = cell_bounds[i]
                new_vertices[i].x = clamp(new_vertices[i].x, bounds[0], bounds[1])
                new_vertices[i].y = clamp(new_vertices[i].y, bounds[2], bounds[3])
                new_vertices[i].z = clamp(new_vertices[i].z, bounds[4], bounds[5])
            # If is_last_pass, no constraint is applied
            
        mesh.vertices = new_vertices
        
    return mesh
```

### Parameters and Edge Cases
- **Iterations**: Typically 5 [S1].
- **Constraint Range**: The vertex movement is bounded by half the width of a grid cell [S1]. The range can be augmented to allow relaxation across multiple voxels if needed [S1].
- **Feature Lines**: Feature lines along common boundaries of three or more materials may remain visible after fairing [S1].
- **Volume Conservation**: The method does not conserve material volume. If volume conservation is required, correction operators must be applied separately [S1].

## Subtopics

### Surface Fairing with Sharp Boundaries
This variant handles multi-material SurfaceNets by treating the mesh as a union of two-sided surface patches separating pairs of materials, preserving topology by preventing touching surfaces from intersecting [S2, S2, unofficial]. It distinguishes between two vertex types:
- **SurfaceVertices**: Vertices in cells where each of the 6 faces separates at most two materials. These are faired using the standard binary data surface fairing method [S2, S2, unofficial].
- **EdgeVertices**: Vertices in cells containing at least one **JunctionFace**. A JunctionFace is a face intersected by more than two materials, or a face intersected by two materials where vertices of the same material are on diagonally opposite corners [S2, S2, unofficial].
For EdgeVertices, the method computes the `LocalAverage` using only vertices from JunctionFace neighbors, ignoring SurfaceFace neighbors. This encourages movement along the edge and discourages movement perpendicular to it, preserving sharp boundaries [S2, S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(F \cdot I)$, where $F$ is the number of faces and $I$ is the number of iterations. Computing face centroids is $O(F)$, and updating vertices by traversing faces is $O(F)$ per iteration.
- **Memory Complexity**: $O(V + F)$ for storing the mesh and face centroids. No explicit vertex-to-face adjacency list is required, saving memory [S1].
- **Strengths**: Fast, simple, and guarantees that the smoothed mesh stays close to the original extraction grid, preventing large topological errors. It is particularly effective for meshes with regular valences (3-6) [S1].
- **Limitations**: Does not conserve volume. Feature lines at multi-material boundaries may persist unless additional priority-based smoothing is applied [S1]. The constraint can lead to "staircasing" if the grid resolution is coarse relative to the feature size.

## Background
Constrained Laplacian Fairing was introduced by Bertram et al. in 2005 as part of the "Non-manifold Mesh Extraction" method for modeling a human heart from segmented volumes [S1]. It was designed to regularize the meshes produced by their contouring algorithm, which generated topologically simple but geometrically noisy quadrilateral meshes. The method was later adapted for multi-label segmentations in the SurfaceNet framework to preserve sharp boundaries between materials [S2, S2, unofficial].

## Sources
[S1] Non-manifold Mesh Extraction from Time-varying Segmented Volumes used for Modeling a Human Heart, https://doi.org/10.2312/vissym/eurovis05/199-206, fetched 2026-10-26
[S2] SurfaceNets for Multi-Label Segmentations with Preservation of Sharp Boundaries., https://europepmc.org/article/MED/36325473, fetched 2026-10-26 (unofficial)
