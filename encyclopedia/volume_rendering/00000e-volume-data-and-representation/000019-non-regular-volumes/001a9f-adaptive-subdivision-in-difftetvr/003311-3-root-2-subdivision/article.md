# 3-root-2 Subdivision

Also called: 3-subdivision, 3√2 subdivision

## Overview
3-root-2 subdivision is a hierarchical volume representation scheme that refines a mesh by doubling the total number of vertices per step, theoretically increasing vertex density by a factor of $\sqrt{2}$ in each spatial dimension [S1, unofficial]. It solves the problem of coarse granularity in standard octree refinement, which multiplies vertex counts by 8 per step, by providing finer resolution control for multiresolution rendering [S1, unofficial]. It is distinguished from its siblings by its specific geometric insertion sequence (centroid, then face centers, then edge midpoints) and its reliance on B-spline wavelets for data value updates [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A volumetric mesh (typically structured-rectilinear grids or tetrahedral grids) and a scalar field defined on vertices.
*   **Output:** A refined mesh with higher vertex density and updated scalar values at all vertices.

**Data Structures**
The method operates on a hierarchical structure where each subdivision step generates new vertices.
*   **Vertices:** Stored with 3D coordinates $(x, y, z)$ and scalar values.
*   **Connectivity:** Adjacency lists defining which new vertices are "adjacent" to an old vertex for the averaging step. In structured grids, eight cuboids share a vertex.

**Numbered Steps**
The subdivision of a single cuboid element requires three sequential steps to achieve the equivalent refinement of one octree split:

1.  **Step 1 (Centroid):** Insert the centroid of the original cuboid. Connect this new vertex to all 8 original vertices of the cuboid.
2.  **Step 2 (Face Centers):** Insert the centers of the 6 faces of the original cuboid. These form the centroids of the octahedra created by the previous step. Connect each face center to the vertices of that face. Delete old edges that were not inserted in the previous step (specifically, edges from the original cuboid that are not part of the new connectivity).
3.  **Step 3 (Edge Midpoints):** Insert the midpoints of the 12 edges of the original cuboid. Connect each edge midpoint to the vertices of that edge. Delete old edges that were not inserted in the last two steps (specifically, edges inserted in Step 1 and Step 2 that are not part of the final edge-midpoint connectivity).

**Formulas**
The data value update (averaging step) for an old vertex $v$ is defined as:
$$ v_{new} = \alpha v_{old} + (1 - \alpha) w $$
Where:
*   $v_{old}$ is the scalar value at the original vertex.
*   $w$ is the centroid (average) of the scalar values of the adjacent new vertices created in the subdivision steps.
*   $\alpha$ is a weighting parameter.

**Parameters**
*   **Alpha ($\alpha$):** The source does not provide a specific value or solution for $\alpha$ in dimensions larger than two ($n=3$) [S1, unofficial]. It states: "we cannot provide a solution for the choice of $\alpha$ in the update rule" for volumetric data [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Structured-Rectilinear Grids:** If the input is a structured rectilinear grid where eight cuboids share a vertex and all cuboids are of equal size, the averaging step does not change the vertex positions regardless of the specific $\alpha$ value [S1, unofficial].
*   **Wavelet Artifacts:** Wavelets based on 3-root-2 subdivision create over- and undershoots. This can cause isosurfaces to vanish or create spurious components when increasing resolution [S1, unofficial].
*   **Tetrahedral Grids:** For tetrahedral grids, the splitting step is based on bisection of the longest edge, analogous to the splitting step in the cuboid scheme, but a full subdivision scheme for tetrahedra is not fully detailed in the source [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_3root2_subdivision(mesh, alpha):
    """
    Applies 3-root-2 subdivision to a mesh.
    Note: Alpha value for 3D is not defined in sources [S1, unofficial].
    """
    # Step 1: Insert Cuboid Centroids
    new_vertices_step1 = []
    for cuboid in mesh.cuboids:
        centroid = compute_centroid(cuboid.vertices)
        new_vertices_step1.append(centroid)
        # Connect centroid to all 8 old vertices of the cuboid
        update_connectivity(cuboid, centroid)
    
    # Step 2: Insert Face Centers
    new_vertices_step2 = []
    for cuboid in mesh.cuboids:
        for face in cuboid.faces:
            face_center = compute_centroid(face.vertices)
            new_vertices_step2.append(face_center)
            # Connect face center to face vertices
            update_connectivity(face, face_center)
            # Delete old edges not part of new connectivity
            # (Specifically, original cuboid edges not used in Step 1 or 2)
            cleanup_edges(cuboid)

    # Step 3: Insert Edge Midpoints
    new_vertices_step3 = []
    for cuboid in mesh.cuboids:
        for edge in cuboid.edges:
            midpoint = compute_midpoint(edge.vertices)
            new_vertices_step3.append(midpoint)
            # Connect midpoint to edge vertices
            update_connectivity(edge, midpoint)
            # Delete old edges not part of final connectivity
            # (Edges from Step 1 and 2 not used in Step 3)
            cleanup_edges(cuboid)

    # Averaging Step: Update scalar values
    for vertex in mesh.old_vertices:
        # w is the centroid of adjacent new vertices
        adjacent_new = get_adjacent_new_vertices(vertex)
        if adjacent_new:
            w = average_values(adjacent_new)
            # Update rule: v = alpha * v + (1 - alpha) * w
            # Note: Alpha value for 3D is not provided in sources [S1, unofficial]
            vertex.value = alpha * vertex.value + (1 - alpha) * w
        else:
            # Fallback if no adjacent new vertices (edge case)
            pass # Implementation detail not specified in sources

    return mesh
```

## Subtopics

### Differentiable Mesh Refinement
A training strategy that refines the tetrahedral mesh by subdividing vertices with high color gradients. It differs by focusing on gradient-based adaptive refinement for neural rendering rather than fixed hierarchical subdivision steps [984-differentiable-mesh-refinement.md].

### Recursive Subdivision
A general approach to mesh refinement. It differs by potentially using arbitrary subdivision rules, whereas 3-root-2 uses a specific 3-step geometric insertion sequence [1304-recursive-subdivision.md].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Each 3-root-2 subdivision step doubles the total number of vertices. Theoretically, this increases vertex count by a factor of $\sqrt{2}$ in each dimension [S1, unofficial]. Three subdivision steps are equivalent to one octree refinement step (which multiplies vertices by 8) [S1, unofficial].
*   **Strengths:** Provides finer granularity than octree refinement, allowing the renderer to get closer to a required mesh-element size and potentially render less data for desired quality [S1, unofficial].
*   **Limitations:** Wavelets based on this scheme create over- and undershoots, which can cause isosurfaces to vanish when increasing resolution [S1, unofficial]. The choice of the averaging parameter $\alpha$ for 3D is not defined [S1, unofficial].
*   **When to use:** When finer granularity is needed than octree refinement provides, and the data is structured-rectilinear or tetrahedral.
*   **When not to use:** When isosurface stability is critical and wavelet artifacts (vanishing isosurfaces) are unacceptable, or when a defined $\alpha$ parameter is required for 3D averaging.

## Background
The method was introduced in the context of hierarchical volume representation using n-variate B-spline wavelets [S1, unofficial]. It generalizes the 4-subdivision scheme (for $n=2$) to 3-root-2 subdivision for $n=3$ [S1, unofficial]. The name derives from the theoretical factor of $\sqrt{2}$ increase per dimension [S1, unofficial]. The work references prior research in 3D subdivision by MacCracken and Joy (1996) and lifting schemes by Bertram et al. (2001) and Kobbelt (2000) [S1, unofficial].

## Sources
[S1] Hierarchical Volume Representation with 3{radical}2 Subdivision and Trivariate B-Spline Wavelets, https://digital.library.unt.edu/ark:/67531/metadc1411654/?q=multiresolution volume rendering, fetched 2026-10-28 (unofficial)
