# Manifold Dual Contouring - Cell Clustering

Also called: ClusterCell, MDC, Manifold Dual Contouring (Java Implementation)

## Overview
Manifold Dual Contouring - Cell Clustering is a vertex aggregation step within the Manifold Dual Contouring pipeline that consolidates vertices from an octree cell's children into a single manifold-compliant vertex per topological feature [S2, unofficial]. It solves the problem of non-manifold geometry and redundant vertices that arise when multiple child nodes intersect the same surface edge or face, ensuring the output mesh maintains a valid 2-manifold structure [S2, unofficial]. It differs from standard Dual Contouring by explicitly tracking surface indices and collapsing vertices across child boundaries using parent pointers and Euler characteristic constraints, building on [Manifold Dual Contouring](2138-manifold-dual-contouring.md) [S2, unofficial].

## How it works
**Inputs:** An octree node (`MdcOctreeNode`) representing a cell, which may contain child nodes (`MdcOctreeNode`) and existing vertices (`MdcVertex`) [S2, unofficial]. The implicit field values at the cell corners are stored in `node.corners` (bitmask) [S2, unofficial].
**Outputs:** The input node’s `vertices` array is replaced with a new array of aggregated `MdcVertex` objects [S2, unofficial]. Each new vertex represents a collapsed cluster of child vertices [S2, unofficial].

**Data Structures:**
*   `MdcOctreeNode`:
    *   `Type`: Enum (`Node_Internal`, `Node_Leaf`).
    *   `min`: `Vec3i` (position).
    *   `size`: `int` (side length).
    *   `corners`: `byte` (8-bit mask, bit $i$ is 1 if corner $i$ is inside, 0 otherwise).
    *   `vertices`: `MdcVertex[]`.
    *   `children`: `MdcOctreeNode[8]`.
    *   `child_index`: `int` (index of this node among its parent's children).
*   `MdcVertex`:
    *   `parent`: `MdcVertex` (pointer to the vertex it collapsed into).
    *   `surface_index`: `int` (unique ID for the surface feature this vertex belongs to; -1 if unassigned).
    *   `qef`: `QEFData` (Quadratic Error Function data, solved via `LevenQefSolver`).
    *   `normal`: `Vec3f` (accumulated normal).
    *   `eis`: `int[12]` (Edge Intersection Signs, counts of intersections for the 12 cell edges).
    *   `euler`: `int` (Euler characteristic contribution).
    *   `in_cell`: `int` (index of the cell this vertex originated from).
    *   `face_prop2`: `boolean` (flag indicating specific face topology validity).
    *   `pos`: `Vec3f` (position, solved from QEF).
    *   `error`: `float`.
    *   `index`: `int`.
    *   `collapsible`: `boolean`.

**Algorithm Steps:**

1.  **Recursive Child Processing & Sign Propagation:**
    *   If `node.Type` is not `Node_Internal`, return [S2, unofficial].
    *   Initialize `signs` array of size 8 to -1, `mid_sign` to -1, and `is_collapsible` to `true` [S2, unofficial].
    *   Loop `i` from 0 to 7:
        *   If `node.children[i]` is `null`, continue [S2, unofficial].
        *   Recursively call `ClusterCell(node.children[i], error)` [S2, unofficial].
        *   If `node.children[i].Type == Node_Internal`, set `is_collapsible = false` [S2, unofficial].
        *   Else (leaf), extract the corner bit: `mid_sign = (node.children[i].corners >> (7 - i)) & 1` [S2, unofficial].
        *   Set `signs[i] = (node.children[i].corners >> i) & 1` [S2, unofficial].
    *   Reset `node.corners` to 0 [S2, unofficial].
    *   Loop `i` from 0 to 7:
        *   If `signs[i] == -1`, `node.corners |= (byte)(mid_sign << i)` [S2, unofficial].
        *   Else, `node.corners |= (byte)(signs[i] << i)` [S2, unofficial].

2.  **Face and Edge Clustering Initialization:**
    *   Initialize `surface_index` to 0, `collected_vertices` list, and `new_vertices` list [S2, unofficial].
    *   **Process Faces:** Loop `i` from 0 to 11:
        *   Get `c1`, `c2` from `Utilities.EdgePairs[i]` [S2, unofficial].
        *   Set `face_nodes[0] = node.children[c1]`, `face_nodes[1] = node.children[c2]` [S2, unofficial].
        *   Call `ClusterFace(face_nodes, Utilities.EdgePairs[i][2], surface_index, collected_vertices)` [S2, unofficial].
    *   **Process Internal Edges:** Loop `i` from 0 to 5:
        *   Get `edge_nodes` from `node.children` using `Utilities.TCellProcEdgeMask[i][0..3]` [S2, unofficial].
        *   Call `ClusterEdge(edge_nodes, Utilities.TCellProcEdgeMask[i][4], surface_index, collected_vertices)` [S2, unofficial].

3.  **Stray Vertex Assignment:**
    *   Set `highest_index = surface_index.value` [S2, unofficial].
    *   If `highest_index == -1`, set `highest_index = 0` [S2, unofficial].
    *   Loop over `n` in `node.children`:
        *   If `n` is `null`, continue [S2, unofficial].
        *   Loop over `v` in `n.vertices`:
            *   If `v` is `null`, continue [S2, unofficial].
            *   If `v.surface_index == -1`, set `v.surface_index = highest_index++` and add `v` to `collected_vertices` [S2, unofficial].

4.  **Vertex Aggregation and QEF Solving:**
    *   Initialize `clustered_count = 0` [S2, unofficial].
    *   If `collected_vertices.size() > 0`, loop `i` from 0 to `highest_index`:
        *   Create `QEFData qef` with `LevenQefSolver` [S2, unofficial].
        *   Initialize `normal = Vec3f(0)`, `count = 0`, `edges` array of size 12 to 0, `euler = 0`, `e = 0` [S2, unofficial].
        *   Loop over `v` in `collected_vertices`:
            *   If `v.surface_index == i`:
                *   For `k` from 0 to 2: `edge = Utilities.TExternalEdges[v.in_cell][k]`, `edges[edge] += v.eis[edge]` [S2, unofficial].
                *   For `k` from 0 to 8: `edge = Utilities.TInternalEdges[v.in_cell][k]`, `e += v.eis[edge]` [S2, unofficial].
                *   `euler += v.euler` [S2, unofficial].
                *   `qef.add(v.qef)` [S2, unofficial].
                *   `normal = normal.add(v.normal)` [S2, unofficial].
                *   `count++` [S2, unofficial].
        *   If `count == 0`, continue to next `i` [S2, unofficial].
        *   **Face Property Check:** Set `face_prop2 = true` [S2, unofficial]. Loop `f` from 0 to 5:
            *   If `face_prop2`, `intersections = 0` [S2, unofficial].
            *   Loop `ei` from 0 to 3: `intersections += edges[Utilities.TFaces[f][ei]]` [S2, unofficial].
            *   If `intersections != 0` and `intersections != 2`, set `face_prop2 = false` [S2, unofficial].
        *   **Create New Vertex:**
            *   `new_vertex = new MdcVertex()` [S2, unofficial].
            *   `normal = normal.div(count)` [S2, unofficial].
            *   `normal.normalize()` [S2, unofficial].
            *   `new_vertex.normal = normal` [S2, unofficial].
            *   `new_vertex.qef = qef` [S2, unofficial].
            *   `new_vertex.eis = edges` [S2, unofficial].
            *   `new_vertex.euler = euler - e / 4` [S2, unofficial].
            *   `new_vertex.in_cell = node.child_index` [S2, unofficial].
            *   `new_vertex.face_prop2 = face_prop2` [S2, unofficial].
            *   Add `new_vertex` to `new_vertices` [S2, unofficial].
        *   **Set Parent Pointers:**
            *   Loop over `v` in `collected_vertices`:
                *   If `v.surface_index == i`:
                    *   If `v != new_vertex`, `v.parent = new_vertex` [S2, unofficial].
                    *   Else, `v.parent = null` [S2, unofficial].

5.  **Finalization:**
    *   If `collected_vertices.size() == 0`, return [S2, unofficial].
    *   Loop over `v2` in `collected_vertices`, set `v2.surface_index = -1` [S2, unofficial].
    *   Set `node.vertices = new_vertices` array [S2, unofficial].

**Formulas:**
*   **Intersection Point (used in child leaf construction):**
    $$p = p_1 + \frac{-d_1}{d_2 - d_1} (p_2 - p_1)$$
    Where $p_1, p_2$ are endpoints of an edge, and $d_1, d_2$ are scalar field values at those endpoints [S2, unofficial].
*   **Surface Normal (used in child leaf construction):**
    $$n = \frac{1}{2H} [f(x+H,y,z)-f(x-H,y,z), f(x,y+H,z)-f(x,y-H,z), f(x,y,z+H)-f(x,y,z-H)]^T$$
    Normalized, where $H=1$ [S2, unofficial].

**Parameters:**
*   `EnforceManifold`: Boolean flag controlling manifold enforcement; default value is `true` [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Null Children:** The algorithm skips null children in loops. If all children are null, the function returns early without processing vertices [S2, unofficial].
*   **Face Property 2:** The check `intersections != 0 && intersections != 2` validates that the face has exactly two edge intersections, a requirement for manifold validity in this topology [S2, unofficial].

## Subtopics

### Manifold Dual Contouring - Face Clustering
This method processes the vertices of a single octree cell's faces to determine triangle indices and resolve vertex conflicts by traversing parent pointers [S1]. It differs from standard dual contouring by enforcing manifold constraints through parent traversal, building on [Manifold Dual Contouring - Cell Clustering](3677-manifold-dual-contouring-cell-clustering.md) [S1]. It is chosen when face-level topology consistency is required before index generation.

### [Connectivity Generation via Recursive Traversal](7454-connectivity-generation-via-recursive-traversal.md)
This algorithm extracts surface connectivity from an octree-based dual contouring representation by traversing the tree to enumerate minimal edges intersected by the surface. It establishes a one-to-one relationship between surface patches and these minimal edges, ensuring manifold output. It differs from sibling methods by focusing exclusively on topological traversal and patch generation via recursive cell, face, and edge processing, rather than index processing or triangle generation.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of vertices in the octree leaves within the cell. The recursive nature means it processes each node once. The aggregation step involves iterating over collected vertices, which is proportional to the number of intersecting edges/faces.
*   **Memory Complexity:** Proportional to the number of vertices in the cell. It creates a new list of vertices per cell.
*   **Strengths:** Ensures manifold output by design, handles adaptive octree structures efficiently, and simplifies the mesh by collapsing redundant vertices.
*   **Limitations:** Relies on specific topology masks (`Utilities` tables) which are complex and hard-coded. The manifold enforcement may fail or produce artifacts if the implicit field sampling is inconsistent or if the octree structure is invalid.
*   **When to use:** When a manifold mesh is required from an adaptive octree-based implicit field, and topological correctness is prioritized over raw speed.
*   **When not to use:** When non-manifold geometry is acceptable or desired, or when a simpler, faster method like Marching Cubes is sufficient.

## Background
Originated from the work of John Lin20, as implemented in the Java repository `proton2/Manifold-dual-contouring-Java` [S2, unofficial]. It is a variant of Dual Contouring that integrates topology constraints to maintain manifold surfaces, building on the general concept of isosurface extraction using octrees [S1]. It is used in 3D visualization for constructing manifold surfaces from implicit fields like SimplexNoise [S2, unofficial].

## Sources
[S1] Isosurface, https://en.wikipedia.org/wiki/Isosurface, fetched 2026-10-26
[S2] proton2/Manifold-dual-contouring-Java, https://github.com/proton2/Manifold-dual-contouring-Java, fetched 2026-10-26 (unofficial)
