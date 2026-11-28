# Quadric Error Metric Tetrahedral Simplification

Also called: Garland and Zhou simplification, QEM Tetrahedral Decimation, Quadric error metric simplification, Quadric-Based Simplification, Tetrahedral QEM Simplification

## Overview
Quadric Error Metric Tetrahedral Simplification is a mesh decimation algorithm that reduces the vertex count of a tetrahedral mesh by iteratively collapsing edges, minimizing geometric and field approximation errors using quadric metrics [S2]. It distinguishes itself from sibling methods by supporting volumetric (tetrahedral) meshes rather than just surface triangles, and by incorporating a positional correction term to ensure numerical stability during collapse operations [S1, S2]. It builds on the edge collapse paradigm [3505-gpu-based-mesh-decimation-via-edge-collapse.md] but extends the error metric to handle 3D volume elements and scalar field data [G].

## How it works
**Inputs:** A tetrahedral mesh consisting of vertices, edges, and tetrahedra, optionally accompanied by a scalar field function $f$ defined over the mesh domain [S2].
**Outputs:** A simplified tetrahedral mesh with a reduced vertex count [G].

**Data Structures:**
*   **Quadric Matrix $A$:** A symmetric $n \times n$ matrix, where $n$ is the dimension of the embedding space (typically 3 for geometry, or 4 when including field data) [S2].
*   **Vector $b$:** An $n$-dimensional vector [S2].
*   **Scalar $c$:** A scalar value [S2].
*   **Vertex Data:** Each vertex stores its accumulated quadric $(A, b, c)$ and its current position [S2].

**Formulas:**
The quadric error metric measures the squared geometric and field distances from a point $x$ to hyperplanes spanned by tetrahedra [S2]. For a single hyperplane defined by normal $n$ and distance $d$ (equation $n^T x + d = 0$), the squared distance is $(n^T x + d)^2$ [G]. This expands to the quadratic form:
$$ Q(x) = x^T A x + 2b^T x + c $$
where:
*   $A = n n^T$
*   $b = n d$
*   $c = d^2$

For a vertex incident to multiple hyperplanes (tetrahedra faces), the total quadric is the component-wise sum of individual quadrics:
$$ A_{total} = \sum A_i, \quad b_{total} = \sum b_i, \quad c_{total} = \sum c_i $$

If field data is included, vertices are treated as points in $\mathbb{R}^4$: $p = [x, y, z, f]^T$ [S2]. The hyperplanes are extended to 4D to represent the field function approximation [S2].

**Steps:**
1.  **Initialization:** For each vertex, compute the initial quadric $(A, b, c)$ by summing the contributions from all incident tetrahedra (and boundary faces if applicable), applying appropriate weighting to preserve volume boundaries [S2].
2.  **Edge Selection & Cost Calculation:** For each edge, combine the quadrics of the two connected vertices by component-wise addition: $A_{new} = A_1 + A_2$, $b_{new} = b_1 + b_2$, $c_{new} = c_1 + c_2$ [S2]. Solve the linear system $A_{new} x = b_{new}$ to find the optimal position $x$ that minimizes the quadric error [S2]. Calculate the error value $E = x^T A_{new} x + 2b_{new}^T x + c_{new}$ [G].
3.  **Stability Check:** Apply a positional correction term to the computed position $x$ to maintain numerical stability during the collapse [S1].
4.  **Validity Check:** Test if collapsing the edge to position $x$ causes any adjacent tetrahedra to flip (i.e., change the sign of their volume). If a flip occurs, disallow the edge collapse [S2].
5.  **Collapse:** Select the edge with the minimum valid error cost. Collapse the edge by merging the two vertices into one at position $x$, updating the connectivity, and recomputing quadrics for affected vertices [S2].
6.  **Repeat:** Continue until the target vertex count or error threshold is reached [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_quadric_components(normal, distance):
    n = len(normal)
    A = [[normal[i] * normal[j] for j in range(n)] for i in range(n)]
    b = [normal[i] * distance for i in range(n)]
    c = distance * distance
    return A, b, c

def add_quadrics(q1, q2):
    A1, b1, c1 = q1
    A2, b2, c2 = q2
    n = len(b1)
    A_new = [[A1[i][j] + A2[i][j] for j in range(n)] for i in range(n)]
    b_new = [b1[i] + b2[i] for i in range(n)]
    c_new = c1 + c2
    return A_new, b_new, c_new

def solve_linear_system(A, b):
    # [G] Uses a numerically stable solver (e.g., least squares) since A may be singular [S2].
    # Standard implementation omitted; returns x minimizing ||Ax - b||.
    return [0.0] * len(b)

def apply_positional_correction(position):
    # [G] Applies a stabilization term to the computed position [S1].
    return position

def compute_signed_volume(v0, v1, v2, v3):
    return (v0[0]*(v1[1]*v2[2] - v1[2]*v2[1]) -
            v0[1]*(v1[0]*v2[2] - v1[2]*v2[0]) +
            v0[2]*(v1[0]*v2[1] - v1[1]*v2[0])) / 6.0

def check_tetrahedron_flips(tetrahedra, new_position, v1_idx, v2_idx):
    for tet in tetrahedra:
        verts = [tet.v0, tet.v1, tet.v2, tet.v3]
        for i in range(4):
            if i == v1_idx or i == v2_idx:
                verts[i] = new_position
        if compute_signed_volume(*verts) < 0:
            return True
    return False

def simplify_tetrahedral_mesh(mesh, target_vertex_count):
    for vertex in mesh.vertices:
        n = 4 if vertex.has_field_data else 3
        A = [[0.0]*n for _ in range(n)]
        b = [0.0]*n
        c = 0.0
        for tet in vertex.incident_tetrahedra:
            for face in tet.faces:
                normal, distance = face.get_hyperplane_params()
                A_f, b_f, c_f = compute_quadric_components(normal, distance)
                for i in range(n):
                    for j in range(n):
                        A[i][j] += A_f[i][j]
                    b[i] += b_f[i]
                c += c_f
        vertex.quadric = (A, b, c)

    edge_queue = []
    for edge in mesh.edges:
        A_new, b_new, c_new = add_quadrics(edge.v1.quadric, edge.v2.quadric)
        optimal_pos = solve_linear_system(A_new, b_new)
        if optimal_pos is None:
            continue
        cost = sum(optimal_pos[i] * sum(A_new[i][j] * optimal_pos[j] for j in range(len(optimal_pos))) for i in range(len(optimal_pos))) + 2 * sum(b_new[i] * optimal_pos[i] for i in range(len(b_new))) + c_new
        edge_queue.append((cost, edge))
    edge_queue.sort(key=lambda x: x[0])

    while mesh.vertex_count > target_vertex_count and edge_queue:
        _, edge = edge_queue.pop(0)
        if not edge.is_valid():
            continue

        A_new, b_new, c_new = add_quadrics(edge.v1.quadric, edge.v2.quadric)
        optimal_pos = solve_linear_system(A_new, b_new)
        if optimal_pos is None:
            continue

        optimal_pos = apply_positional_correction(optimal_pos)

        if check_tetrahedron_flips(edge.affected_tetrahedra, optimal_pos, edge.v1.idx, edge.v2.idx):
            continue

        edge.v1.position = optimal_pos
        edge.v2.mark_invalid()
        mesh.remove_edge(edge)
        mesh.update_connectivity(edge)

        for v in mesh.get_affected_vertices(edge):
            n = 4 if v.has_field_data else 3
            A = [[0.0]*n for _ in range(n)]
            b = [0.0]*n
            c = 0.0
            for tet in v.incident_tetrahedra:
                for face in tet.faces:
                    normal, distance = face.get_hyperplane_params()
                    A_f, b_f, c_f = compute_quadric_components(normal, distance)
                    for i in range(n):
                        for j in range(n):
                            A[i][j] += A_f[i][j]
                        b[i] += b_f[i]
                    c += c_f
            v.quadric = (A, b, c)
        edge_queue.sort(key=lambda x: x[0])
```

**Parameters:**
*   **Target Vertex Count:** The desired number of vertices in the simplified mesh. Not given in the sources.
*   **Error Threshold:** Maximum allowable quadric error per collapse. Not given in the sources.
*   **Boundary Weighting:** Weight applied to boundary face errors to preserve volume boundaries. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Non-invertible Matrix:** The matrix $A$ is not necessarily invertible. A numerically stable linear solver must be chosen [S2].
*   **Tetrahedron Flips:** Collapsing to the optimal point may cause tetrahedra to flip (change volume sign). This must be tested and disallowed if it occurs [S2].
*   **Numerical Stability:** A positional correction term is incorporated during vertex/edge collapse to maintain stability [S1].

## Subtopics

### Streaming Simplification
An out-of-core technique for simplifying large tetrahedral meshes by combining streaming meshes with quadric-based simplification, allowing processing of meshes larger than memory [S2].

### Field-Aware Simplification
Extends the quadric metric to 4D to optimize field approximation along with geometric position, treating vertices as points on a 3D manifold embedded in $\mathbb{R}^4$ [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:**
    *   Dataset F16: 1,124,648 vertices, 6,345,709 tetrahedra. Simplified to 76,823 vertices. Construction time: 0:40:31 [S1].
    *   Dataset Earthquake: 2,461,694 vertices, 13,980,162 tetrahedra. Simplified to 122,312 vertices. Construction time: 1:12:45 [S1].
*   **Strengths:** High-quality approximations, preserves volume boundaries, handles field data, numerically stable with correction terms [S1, S2].
*   **When to use:** When simplifying tetrahedral meshes for volume rendering or simulation where field data preservation is important [S2].
*   **When not to use:** For surface-only meshes where triangular simplification is sufficient [G].

## Background
The quadric error metric was proposed by Garland and Zhou [14] (Garland and Zhou, 2005) [S2]. The specific implementation for tetrahedral meshes with streaming capabilities was described in "Streaming Simplification of Tetrahedral Meshes" (2007) [S2]. It is used in segment-based tetrahedral meshing [S1].

## Sources
[S1] Segment-based tetrahedral meshing and rendering, https://doi.org/10.1145/1174429.1174507, fetched 2026-11-06
[S2] Streaming Simplification of Tetrahedral Meshes, https://doi.org/10.1109/tvcg.2007.21, fetched 2026-11-04
