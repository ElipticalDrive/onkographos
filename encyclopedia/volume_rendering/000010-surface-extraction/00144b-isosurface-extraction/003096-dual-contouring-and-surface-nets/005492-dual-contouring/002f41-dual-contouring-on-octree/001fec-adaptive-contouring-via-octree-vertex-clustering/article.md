# Adaptive Contouring via Octree Vertex Clustering

Also called: Adaptive Simplification, Octree-based Vertex Clustering

## Overview
Adaptive Contouring via Octree Vertex Clustering is a surface extraction method that constructs multi-resolution iso-surfaces by clustering vertices of child cells in an octree into a single vertex in a topologically safe manner [S1][S2, unofficial]. It allows an arbitrary number of vertices per cell to significantly reduce polygon count in flat regions, addressing the restrictive single-vertex-per-cell limitation of standard Dual Contouring [S1][S2, unofficial]. Unlike its siblings, it explicitly builds a "vertex tree" to manage hierarchical simplification and uses QEF minimization to determine the position of clustered vertices [S1][S2, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   An octree representation of the scalar field [S1][S2, unofficial].
*   An error threshold $\epsilon$ [S1][S2, unofficial].
*   Hermite data (surface normals and intersection points) associated with octree edges [S1][S2, unofficial].

**Outputs:**
*   A triangle mesh representing the iso-surface [S1][S2, unofficial].
*   Vertices are those marked "collapsible" that do not have any collapsible ancestors in the vertex tree [S1][S2, unofficial].

### Data Structures

**Vertex Structure:**
Each vertex in the vertex tree contains:
1.  **Position:** The 3D coordinates of the vertex [S1][S2, unofficial].
2.  **Parent Pointer:** A reference to the parent vertex in the vertex tree (initially `NULL`) [S1][S2, unofficial].
3.  **QEF:** The Quadratic Error Function matrix and vector associated with the vertex [S1][S2, unofficial].
4.  **Error:** The value of the QEF evaluated at the vertex's current position [S1][S2, unofficial].
5.  **Collapsible Flag:** A boolean indicating if the vertex's error is less than the threshold $\epsilon$ [S1][S2, unofficial].

**Octree Cell Structure:**
*   **Collapsed Flag:** A boolean indicating if the cell is "collapsed" (all clustered vertices satisfy the collapsible criterion and all children are leaves or marked collapsed) [S1][S2, unofficial].

### Algorithm Steps

1.  **Initialization:**
    *   Create a vertex tree starting with the vertices at the finest level of the octree [S1][S2, unofficial].
    *   For each vertex at the finest level:
        *   Initialize its parent pointer to `NULL` [S1][S2, unofficial].
        *   Compute its QEF from the Hermite data on the incident edges [S1][S2, unofficial].
        *   Mark the vertex as **collapsible** (initially true for all, or determined by initial error check) [S1][S2, unofficial].

2.  **Bottom-Up Traversal and Clustering:**
    *   Traverse the octree cells from bottom to top (finest to coarsest) [S1][S2, unofficial].
    *   For each non-leaf cell, consider its eight children [S1][S2, unofficial].
    *   Identify the 12 internal faces shared between these children (4 for each Euclidean axis: X, Y, Z) [S1][S2, unofficial].
    *   Find all vertices at the root of the vertex tree that are topologically connected by edges dual to these 12 internal faces [S1].
    *   Group these vertices into clusters based on their topological connectivity [S1][S2, unofficial].
    *   For each cluster group:
        *   Combine the QEFs of all vertices in the group: $Q_{combined} = \sum Q_i$ [S1][S2, unofficial].
        *   Minimize the combined QEF to find the new vertex location $v_{new}$ and its associated error $E_{new}$ [S1][S2, unofficial].
        *   Create a new parent vertex for this cluster with position $v_{new}$, QEF $Q_{combined}$, and error $E_{new}$ [S1][S2, unofficial].
        *   Set the parent pointers of the child vertices in the cluster to this new parent vertex [S1][S2, unofficial].
        *   If $E_{new} < \epsilon$, mark the new parent vertex as **collapsible** [S1][S2, unofficial].
        *   Otherwise, mark it as not collapsible [S1][S2, unofficial].

3.  **Collapsed Cell Marking (Optimization):**
    *   During the clustering process, mark a cell as **collapsed** if:
        *   All clustered vertices created in that cell satisfy the collapsible criterion (error $< \epsilon$) [S1][S2, unofficial].
        *   All children of that cell are either leaf cells or marked collapsed [S1][S2, unofficial].
    *   This flag is used to truncate traversal in subsequent steps [S1][S2, unofficial].

4.  **Polygon Generation:**
    *   Traverse the octree to identify edges that exhibit a sign change (iso-surface intersection) [S1][S2, unofficial].
    *   For each such edge, identify the vertices associated with it [S1][S2, unofficial].
    *   For each vertex, follow the parent pointers up the vertex tree to find the **last vertex marked as collapsible** [S1][S2, unofficial].
    *   Construct a polygon connecting these resolved vertices [S1][S2, unofficial].
    *   **Optimization:**
        *   Truncate `cellProc` traversal if the current cell is marked as **collapsed** [S1][S2, unofficial].
        *   Truncate `faceProc` traversal on a face if both cells sharing the face are marked as **collapsed** [S1][S2, unofficial].
    *   If the resulting polygon collapses to an edge or a vertex, discard it [S1][S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_contouring_octree_vertex_clustering(octree, error_threshold):
    # Step 1: Initialization
    finest_vertices = get_finest_level_vertices(octree)
    vertex_tree = {} 

    for v in finest_vertices:
        qef_A, qef_b = compute_qef_from_hermite_data(v)
        error = evaluate_qef(qef_A, qef_b, v.position)
        vertex = {
            'position': v.position,
            'qef_A': qef_A,
            'qef_b': qef_b,
            'error': error,
            'parent': None,
            'collapsible': error < error_threshold
        }
        vertex_tree[v.id] = vertex

    # Step 2: Bottom-Up Traversal and Clustering
    non_leaf_cells = get_non_leaf_cells_in_bottom_up_order(octree)
    
    for cell in non_leaf_cells:
        children = cell.children
        if not children:
            continue
            
        internal_faces = get_internal_faces(children)
        clusters = find_topologically_connected_vertices(children, internal_faces, vertex_tree)
        
        for cluster in clusters:
            combined_A = sum(v['qef_A'] for v in cluster)
            combined_b = sum(v['qef_b'] for v in cluster)
            
            new_vertex_pos = minimize_qef(combined_A, combined_b)
            new_error = evaluate_qef(combined_A, combined_b, new_vertex_pos)
            
            new_vertex = {
                'position': new_vertex_pos,
                'qef_A': combined_A,
                'qef_b': combined_b,
                'error': new_error,
                'parent': None,
                'collapsible': new_error < error_threshold
            }
            
            for v in cluster:
                v['parent'] = new_vertex
            
            vertex_tree[new_vertex['id']] = new_vertex 
            
            if all(v['collapsible'] for v in cluster) and all_is_leaf_or_collapsed(cell):
                cell.collapsed = True
            else:
                cell.collapsed = False

    # Step 3: Polygon Generation
    triangles = []
    
    def cell_proc(current_cell):
        if current_cell.collapsed:
            return 
        if current_cell.is_leaf():
            process_leaf_edges(current_cell, triangles, vertex_tree)
        else:
            for child in current_cell.children:
                cell_proc(child)
                
    def face_proc(current_face):
        cell1, cell2 = current_face.adjacent_cells
        if cell1.collapsed and cell2.collapsed:
            return 
        process_face_edges(current_face, triangles, vertex_tree)

    def edge_proc(current_edge):
        vertices = get_vertices_for_edge(current_edge, vertex_tree)
        resolved_vertices = []
        for v in vertices:
            current = v
            while current is not None:
                if current['collapsible']:
                    resolved_vertices.append(current)
                current = current['parent']
        
        if len(resolved_vertices) >= 3:
            triangles.append(create_triangle(resolved_vertices))
        elif len(resolved_vertices) <= 2:
            pass 

    cell_proc(octree)
    return triangles

def get_finest_level_vertices(octree):
    # Traverse to leaves and collect edge intersection vertices
    pass

def compute_qef_from_hermite_data(vertex):
    # QEF for a plane with normal n and point p: A = n*n^T, b = n*(n*p)
    # Sum over all incident edges
    A = [[0.0, 0.0, 0.0], [0.0, 0.0, 0.0], [0.0, 0.0, 0.0]]
    b = [0.0, 0.0, 0.0]
    for edge in vertex.incident_edges:
        n = edge.normal
        p = edge.intersection_point
        # A += n * n^T
        for i in range(3):
            for j in range(3):
                A[i][j] += n[i] * n[j]
        # b += n * (n . p)
        dot_np = sum(n[k] * p[k] for k in range(3))
        for i in range(3):
            b[i] += n[i] * dot_np
    return A, b

def evaluate_qef(A, b, x):
    # Computes x^T A x - 2 b^T x
    Ax = [[sum(A[i][j] * x[j] for j in range(3)) for i in range(3)]]
    term1 = sum(x[i] * Ax[0][i] for i in range(3))
    term2 = 2 * sum(b[i] * x[i] for i in range(3))
    return term1 - term2

def minimize_qef(A, b):
    # Solve 3x3 linear system Ax = b
    pass

def get_internal_faces(children):
    # Returns the 12 internal faces shared among 8 children of an octree cell
    pass

def find_topologically_connected_vertices(children, internal_faces, vertex_tree):
    # Uses recursive octree traversal algorithm from Ju et al. [3]
    pass

def all_is_leaf_or_collapsed(cell):
    for child in cell.children:
        if not child.is_leaf() and not child.collapsed:
            return False
    return True

def process_leaf_edges(cell, triangles, vertex_tree):
    for edge in cell.edges:
        if edge.has_sign_change():
            edge_proc(edge)

def process_face_edges(face, triangles, vertex_tree):
    for edge in face.edges:
        if edge.has_sign_change():
            edge_proc(edge)

def create_triangle(vertices):
    return (vertices[0], vertices[1], vertices[2])
```

### Parameters
*   **error_threshold ($\epsilon$):** The maximum allowed QEF error for a vertex to be considered collapsible. If the error associated with a vertex is less than this threshold, it is marked as collapsible [S1][S2, unofficial].

### Pitfalls and Edge Cases
*   **Non-Manifold Topology:** Clustering topologically connected vertices without additional constraints may yield non-manifold topology (e.g., threads collapsing to single polygons or non-manifold edges) [S1][S2, unofficial]. This is addressed in the sibling article [Manifold Dual Contouring](2138-manifold-dual-contouring.md) [S1][S2, unofficial].
*   **Degenerate Polygons:** If the resulting polygon collapses to an edge or a vertex, it must be discarded [S1][S2, unofficial].
*   **Precision:** QEF minimization requires solving a linear system. Numerical stability should be considered, especially for flat regions where the QEF matrix may be ill-conditioned [G].
*   **Arbitrary Intersections:** The method can handle an arbitrary number of intersections per edge, which is an advantage over some other methods [S1][S2, unofficial].

## Operating characteristics

*   **Time Complexity:** The clustering phase involves a bottom-up traversal of the octree [S1][S2, unofficial]. For each non-leaf cell, vertices are clustered by combining QEFs and minimizing them [S2, unofficial]. The polygon generation phase also traverses the octree, but can be optimized by truncating collapsed cells [S1][S2, unofficial].
*   **Memory Complexity:** Requires storing the vertex tree, QEFs for each vertex, and octree structure [S1][S2, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales with the resolution of the octree and the complexity of the surface. Flat regions are simplified more aggressively, reducing polygon count [S1][S2, unofficial].
*   **Strengths:**
    *   Reduces polygon count in flat regions [S1][S2, unofficial].
    *   Allows arbitrary number of vertices per cell, unlike standard Dual Contouring [S1][S2, unofficial].
    *   Topologically safe clustering (though may produce non-manifold results without additional constraints) [S1][S2, unofficial].
*   **Limitations:**
    *   May produce non-manifold topology [S1][S2, unofficial].
    *   Requires Hermite data (normals and intersections) [S1][S2, unofficial].
    *   Optimization via collapsed cells requires careful implementation to ensure correctness [S1][S2, unofficial].
*   **When to Use:** When a multi-resolution iso-surface is desired, and the surface has large flat regions that can be simplified. Suitable when Hermite data is available [S1][S2, unofficial].
*   **When Not to Use:** When manifold topology is strictly required (use [Manifold Dual Contouring](2138-manifold-dual-contouring.md) instead). When Hermite data is not available [S1][S2, unofficial].

## Background

*   **Origin:** The method was introduced in the paper "Manifold Dual Contouring" by Yu et al. [S1][S2, unofficial]. It builds upon the Dual Contouring (DC) method, which was restrictive in allowing only one vertex per cell [S1][S2, unofficial].
*   **History:** The method was developed to address the large number of polygons in flat regions produced by uniform contouring algorithms [S1][S2, unofficial]. It uses the recursive octree traversal algorithm from Ju et al. [3] to efficiently find topologically connected vertices [S1][S2, unofficial].
*   **Usage:** Used in surface extraction from volumetric data, particularly when adaptive resolution is beneficial [S1][S2, unofficial].

## Sources

[S1] Manifold Dual Contouring, https://doi.org/10.1109/tvcg.2007.1012, fetched 2026-10-26
[S2] PDF Manifold Dual Contouring, https://people.engr.tamu.edu/schaefer/research/dualsimp_tvcg.pdf, fetched 2026-10-27 (unofficial)
