# Dynamic Boundary Projection Algorithm (DBPA) Voxel Removal and Mesh Update

Also called: Ball-Pivoting Algorithm (BPA), DBPA Voxel Removal and Mesh Update, Dynamic Ball-Pivoting Algorithm (DBPA)

## Overview
The Dynamic Boundary Projection Algorithm (DBPA) Voxel Removal and Mesh Update is a method for incrementally updating a triangle mesh representation of a voxel set during volume carving [S1, book]. It solves the problem of maintaining a smooth, non-blocky visualization of a volume when voxels are removed, avoiding the computational cost of global remeshing [S1, book]. It sets itself apart from static surface extraction methods by preserving the "BPA front invariant property" through iterative edge loop management and a "glue" operation to remove undesirable adjacent edges [S1, book].

## How it works
**Inputs and Outputs**
*   **Input:** A voxel-set representing the volume, a triangle mesh representing the current surface, and a modified BPA front-end data structure [S1, book]. Each surface voxel in the voxel-set is linked to its corresponding vertex in the triangle mesh [S1, book].
*   **Output:** An updated triangle mesh reflecting the removal of specified voxels [S1, book].

**Data Structures**
The algorithm relies on three major data structures:
1.  **Voxel-set:** The 3D grid representation of the object [S1, book].
2.  **Triangle Mesh:** Defined such that vertices are shared between adjacent edges, and edges are shared between adjacent triangles [S1, book].
3.  **BPA Front-end:** A modified front-end used for updating the display mesh [S1, book].

**Edge Data Structure Layout**
Each edge in the front stores specific geometric data required for the pivoting ball calculation [S1, book]:
1.  Coordinates of the pivotal edge’s vertices.
2.  Ball-center coordinates: The coordinates of the center of the BPA $\rho$-ball when it previously struck the point that generated the triangle [S1, book].
3.  Opposite-point coordinates: The coordinates of the point (vertex) opposite to the front edge in its binding triangle [S1, book].

Newly created triangles in the mesh store their respective ball-centers [S1, book]. When an edge is added to the front, the front’s corresponding edge stores both the ball-center and opposite-point as additional data [S1, book].

**Algorithm Steps**
The initialization phase treats the centers of all voxels in the voxel-set as a point cloud input to the Ball-Pivoting Algorithm (BPA) to generate the initial mesh [S1, book].

For each voxel $v$ removed from the voxel-set, the update procedure is as follows:

1.  **Check Surface Status:** Determine if $v$ was a surface voxel.
    *   If $v$ was **not** a surface voxel, terminate the update procedure for this voxel [S1, book].
    *   If $v$ **was** a surface voxel, proceed to step 2 [S1, book].

2.  **Remove Vertex and Triangles:** Identify the corresponding mesh vertex linked to $v$. Remove this vertex from the triangle mesh and delete all triangles adjacent to that vertex [S1, book].

3.  **Create Boundary Loop:** Create a new loop in the front-end that bounds the region where the triangles were removed. Care must be taken when adding these loops to ensure that the BPA front invariant property is preserved iteratively [S1, book].

4.  **Glue Operation:** Check if adding the new loops has caused undesirable adjacent edges. If so, remove these edges using the BPA’s "glue" operation [S1, book].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_mesh_for_voxel_removal(voxel_set, mesh, bpa_front, voxel_to_vertex_map, removed_voxel):
    """
    Updates the triangle mesh and BPA front after removing a single voxel.
    
    Args:
        voxel_set: Set of active voxels.
        mesh: Current triangle mesh.
        bpa_front: The modified BPA front-end structure.
        voxel_to_vertex_map: Dictionary mapping surface voxel coordinates to mesh vertices.
        removed_voxel: The coordinates of the voxel being removed.
    """
    # Step 1: Check if the removed voxel was a surface voxel
    if removed_voxel not in voxel_to_vertex_map:
        # Not a surface voxel, no mesh update needed
        return

    # Step 2: Identify corresponding vertex and adjacent triangles
    vertex = voxel_to_vertex_map[removed_voxel]
    adjacent_triangles = mesh.get_triangles_adjacent_to_vertex(vertex)
    
    # Remove the vertex and all adjacent triangles from the mesh
    mesh.remove_vertex(vertex)
    mesh.remove_triangles(adjacent_triangles)
    
    # Step 3: Create a new boundary loop in the front-end
    # The loop bounds the region where triangles were removed.
    # The implementation must preserve the BPA front invariant property.
    new_loop = bpa_front.create_boundary_loop(adjacent_triangles)
    
    # Step 4: Glue operation to remove undesirable adjacent edges
    # If the new loop creation caused undesirable adjacent edges, remove them.
    bpa_front.glue(new_loop)
```

**Parameters**
*   **Ball-center:** Stored in newly created triangles and edges. Represents the center of the BPA $\rho$-ball when it previously struck the point that generated the triangle [S1, book].
*   **Opposite-point:** Stored in the front's corresponding edge. Represents the coordinates of the point (vertex) opposite to the front edge in its binding triangle [S1, book].
*   **$\rho$-ball radius:** The radius of the ball used in the BPA. Not given in the sources.

**Edge Cases and Pitfalls**
*   **Non-surface voxels:** If a removed voxel is not on the surface, no mesh update is required. Failing to check this leads to unnecessary computation or errors [S1, book].
*   **Invariant Property:** Care must be taken when adding loops to ensure the BPA front invariant property is preserved. Violating this may break the algorithm's ability to correctly project the next ball strike [S1, book].
*   **Undesirable Edges:** The "glue" operation is necessary to clean up topological artifacts (undesirable adjacent edges) created by the loop insertion [S1, book].

## Operating characteristics
*   **Time Complexity:** Incremental update per voxel removal. Complexity depends on the number of adjacent triangles and the cost of the glue operation. Not given in the sources.
*   **Memory Complexity:** Requires storing additional data (ball-center, opposite-point) in edges and triangles. Not given in the sources.
*   **Strengths:** Provides smooth real-time visualization of volume removal; avoids blocky appearance of raw voxels; avoids computational cost of global remeshing [S1, book].
*   **Limitations:** Requires maintaining complex data structures (BPA front-end with extra fields); relies on the correctness of the BPA initialization and invariant preservation [S1, book].
*   **When to use:** Real-time surgical training simulators or applications requiring interactive carving of voxel-based volumes [S1, book].
*   **When not to use:** Static volume visualization; applications where global remeshing is acceptable or preferred for topological correctness [S1, book].

## Background
The Dynamic Boundary Projection Algorithm (DBPA) is an extension of the Ball-Pivoting Algorithm (BPA) [S1, book]. The BPA was originally intended for 3D data-acquisition of real-world objects [S1, book]. Williams et al. found it to be equally well-suited for generating a triangle mesh from a voxel-set by using the centers of the voxels as the point set [S1, book]. The DBPA was developed to support haptic-based 3D carving simulators, specifically for surgical training, where users manipulate a virtual tool to remove voxels and require immediate, smooth visual feedback [S1, book].

## Sources
[S1] Haptic-Based 3D Carving Simulator, https://doi.org/10.5772/8682, fetched 2026-10-26 (book)
