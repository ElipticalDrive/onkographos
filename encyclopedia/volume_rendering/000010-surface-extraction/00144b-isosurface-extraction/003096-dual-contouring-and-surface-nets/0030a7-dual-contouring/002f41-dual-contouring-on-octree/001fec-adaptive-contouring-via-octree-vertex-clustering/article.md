# Adaptive Contouring via Octree Vertex Clustering

Also called: Adaptive Simplification, Octree-based Vertex Clustering

## Overview
Adaptive Contouring via Octree Vertex Clustering is a surface extraction method that constructs multi-resolution iso-surfaces by clustering vertices of child cells in an octree into a single vertex in a topologically safe manner [S1, unofficial]. It allows an arbitrary number of vertices per cell, significantly reducing polygon count in flat regions compared to methods restricted to one vertex per cell [S1, unofficial]. It sets itself apart from [Dual Contouring with Octree](1271-dual-contouring-with-octree.md) by permitting multiple vertices per cell through a clustering phase, and from [Manifold Dual Contouring](2138-manifold-dual-contouring.md) by its specific vertex-tree-based clustering mechanism which requires additional constraints to guarantee manifold topology [S1, unofficial].

## How it works
**Inputs:** An octree representation of the volume data and an error threshold $\epsilon$ [S1, unofficial].
**Outputs:** A polygonal mesh representing the iso-surface [S1, unofficial].

**Data Structures:**
*   **Vertex:** Contains a parent pointer `parent`, a Quadratic Error Function (QEF) defined by matrix $A$ and vector $\mathbf{b}$, and the scalar error value $E = \|A\mathbf{v} - \mathbf{b}\|^2$ evaluated at the vertex position $\mathbf{v}$ [S1, unofficial].
*   **Vertex Tree:** A hierarchical structure where leaves are finest-level octree vertices and internal nodes represent clustered vertices. Each vertex maintains a pointer to its parent in this tree [S1, unofficial].
*   **Cell Status:** Each octree cell has a boolean flag `collapsed` [S1, unofficial].

**Parameters:**
*   `error_threshold` ($\epsilon$): The maximum allowed QEF error for a vertex to be considered collapsible. If $E < \epsilon$, the vertex is marked collapsible [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Create a vertex tree starting with all vertices at the finest level of the octree [S1, unofficial]. For each vertex, initialize its parent pointer to `NULL` [S1, unofficial]. Flag all vertices as `collapsible` [S1, unofficial].
2.  **Bottom-Up Traversal and Clustering:** Traverse octree cells from bottom to top [S1, unofficial]. For each non-leaf cell, consider its eight children [S1, unofficial]. Identify the twelve internal faces shared by these children (4 for each Euclidean axis) [S1, unofficial]. Find edges dual to these twelve internal faces using a recursive octree traversal algorithm [S1, unofficial]. Group vertices at the root of the vertex tree that are topologically connected by these dual edges [S1, unofficial]. For each group: combine the QEFs of all vertices in the group ($A_{\text{new}} = \sum A_i$, $\mathbf{b}_{\text{new}} = \sum \mathbf{b}_i$) [S1, unofficial]. Minimize the combined error function to find the new vertex location $\mathbf{v}_{\text{new}}$ and its error $E_{\text{new}}$ [S1, unofficial]. If $E_{\text{new}} < \epsilon$, mark the new vertex as `collapsible` [S1, unofficial]. Set the parent pointers of the grouped vertices to point to this new vertex [S1, unofficial].
3.  **Cell Collapse Optimization:** Mark a cell as `collapsed` if all clustered vertices created within it satisfy the collapsible criterion ($E < \epsilon$) AND all children of that cell are either leaf cells or marked as `collapsed` [S1, unofficial].
4.  **Polygon Generation:** Traverse the octree to generate polygons [S1, unofficial]. For each edge exhibiting a sign change in the volume data, create a polygon connecting the associated vertices [S1, unofficial]. For each such vertex, follow its parent pointers up the vertex tree to find the last vertex marked as `collapsible` [S1, unofficial]. Include a vertex in the output mesh if it is marked `collapsible` and has no `collapsible` ancestors in the vertex tree [S1, unofficial]. **Optimization:** If a cell is `collapsed`, truncate the `cellProc` traversal; no children polygons are generated [S1, unofficial]. If both cells sharing a face are `collapsed`, truncate the `faceProc` traversal for that face [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_contouring_octree_vertex_clustering(octree, error_threshold):
    # Data structures
    class QEF:
        def __init__(self, normals, distances):
            self.normals = normals
            self.distances = distances
        def combine(self, other):
            return QEF(self.normals + other.normals, self.distances + other.distances)
        def minimize(self):
            # Solve (A^T A) v = A^T b for v
            import numpy as np
            A = np.array(self.normals)
            b = np.array(self.distances)
            v = np.linalg.lstsq(A, b, rcond=None)[0]
            error = np.sum((A @ v - b) ** 2)
            return v, error

    class Vertex:
        def __init__(self, pos, qef):
            self.pos = pos
            self.qef = qef
            self.error = qef.minimize()[1]
            self.parent = None
            self.is_collapsible = True

    # 1. Initialization
    vertex_tree = {}
    for v_data in octree.get_finest_level_vertices():
        qef = compute_qef_from_surface_intersection(v_data)  # [G] Standard dual contouring step
        vertex_tree[v_data.id] = Vertex(v_data.pos, qef)

    # 2. Bottom-Up Clustering
    for cell in octree.traverse_bottom_up():
        if cell.is_leaf:
            continue
        children = cell.get_children()
        internal_edges = find_dual_edges_for_children(children)  # [G] Lookup 12 internal faces/edges

        active_vertices = []
        for child in children:
            active_vertices.extend(child.get_active_vertices())

        groups = group_connected_vertices(active_vertices, internal_edges)  # [G] Union-Find on dual edges

        for group in groups:
            combined_qef = group[0].qef
            for v in group[1:]:
                combined_qef = combined_qef.combine(v.qef)
            new_pos, new_error = combined_qef.minimize()
            new_vertex = Vertex(new_pos, combined_qef)
            new_vertex.is_collapsible = (new_error < error_threshold)
            for v in group:
                v.parent = new_vertex
                vertex_tree[v.id] = new_vertex
            for child in children:
                child.add_active_vertex(new_vertex)

    # 3. Mark Collapsed Cells
    for cell in octree.traverse_bottom_up():
        if cell.is_leaf:
            continue
        clustered_vertices = cell.get_active_vertices()
        if not clustered_vertices:
            continue
        all_collapsible = all(v.is_collapsible for v in clustered_vertices)
        children_ok = all(c.is_leaf or c.is_collapsed for c in cell.get_children())
        if all_collapsible and children_ok:
            cell.is_collapsed = True

    # 4. Polygon Generation
    mesh = []
    for edge in octree.get_sign_changing_edges():
        verts = edge.get_associated_vertices()
        final_verts = []
        for v in verts:
            curr = v
            while curr.parent is not None and curr.parent.is_collapsible:
                curr = curr.parent
            final_verts.append(curr)
        if len(final_verts) >= 3:
            mesh.append(final_verts)
        elif len(final_verts) == 2:
            mesh.append(final_verts)
    return mesh
```

**Pitfalls:**
*   **Non-manifold topology:** Clustering topologically connected vertices without additional constraints may yield non-manifold topology (e.g., threads collapsing to single polygons or non-manifold edges) [S1, unofficial].

**Subtopics:**
*   [Manifold Dual Contouring](2138-manifold-dual-contouring.md): Adds topology constraints to prevent non-manifold collapses (e.g., threads collapsing to single polygons) which this method may produce. Choose this when manifold output is strictly required.
*   [Octree-Based Dual Contouring (3D)](1417-octree-based-dual-contouring-3d.md): Uses a different approach to octree-based dual contouring. This method specifically uses vertex clustering to allow arbitrary vertices per cell.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Reduces polygon count in flat regions by allowing arbitrary vertices per cell. Optimizations (collapsed cells/faces) prune the traversal [S1, unofficial].
*   **Limitations:** May produce non-manifold topology (e.g., threads collapsing to single polygons) if no additional constraints are applied [S1, unofficial]. Requires careful handling of topological connectivity [S1, unofficial].
*   **When to use:** When a simplified, multi-resolution mesh is desired and non-manifold artifacts are acceptable or can be filtered [S1, unofficial].
*   **When not to use:** When strict manifold topology is required without post-processing [S1, unofficial].

## Background
Originated in the context of dual contouring simplification, specifically addressing the restriction of one vertex per cell in standard DC [S1, unofficial]. It builds on vertex clustering techniques similar to Zhang et al. [7] but uses a vertex tree for topological connectivity [S1, unofficial]. The method was developed to allow arbitrary numbers of vertices per cell, enabling more aggressive simplification [S1, unofficial].

## Sources
[S1] Manifold Dual Contouring, https://people.engr.tamu.edu/schaefer/research/dualsimp_tvcg.pdf, fetched 2026-10-27 (unofficial)
