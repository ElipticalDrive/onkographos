# Quadric Error Metrics Simplification

Also called: Garland and Heckbert Simplification, QEM Simplification

## Overview
Quadric Error Metrics (QEM) Simplification is a mesh simplification algorithm that iteratively collapses edges to reduce polygon count while minimizing geometric error [S1, unofficial]. It differs from topology-preserving methods like Hoppe's Progressive Meshes by allowing non-manifold operations and prioritizing visual fidelity over strict connectivity, making it suitable for multi-resolution representations where appearance matters more than mesh validity [S1, unofficial].

## How it works
**Inputs:** A triangular mesh $M$ (vertices $V$, faces $F$).
**Outputs:** A simplified mesh $M'$ with fewer triangles.

**Data Structures:**
*   **Quadric $Q_i$**: A $4 \times 4$ symmetric matrix for each vertex $i$, representing the accumulated squared distance to all incident planes.
*   **Error Map**: Stores the minimum error for each possible edge collapse.

**Steps:**
1.  **Initialization**: For each vertex $v_i$ with position $\mathbf{p}_i = (x, y, z, 1)^T$, initialize its quadric $Q_i$ as the sum of quadrics of all incident faces. For a face with plane equation $ax+by+cz+d=0$, the plane quadric is $Q_{plane} = \mathbf{n}\mathbf{n}^T$ where $\mathbf{n} = [a, b, c, d]^T$.
2.  **Edge Collapse Evaluation**: For each edge $(v_i, v_j)$, calculate the error of collapsing $v_j$ into $v_i$. The error is $E = \mathbf{p}_i^T (Q_i + Q_j) \mathbf{p}_i$.
3.  **Selection**: Select the edge with the minimum error.
4.  **Collapse**: Remove $v_j$, merge its quadric into $v_i$ ($Q_i \leftarrow Q_i + Q_j$), and update the error map for adjacent edges.
5.  **Iteration**: Repeat until the target triangle count is reached.

**Parameters:**
*   `triangle_count`: Target number of triangles. Typical value: `mesh.faces.shape[0] // 10` [S1, unofficial].
*   `max_error`: Maximum allowable error per collapse. Example: 40 [S1, unofficial].
*   `reduction_factor`: Percentage of triangles to remove. Example: 100 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simplify_quadric(mesh, target_triangles):
    # Initialize quadrics for all vertices
    quadrics = [zero_4x4_matrix() for _ in mesh.vertices]
    for face in mesh.faces:
        plane = get_plane_equation(face)
        plane_quadric = outer_product(plane, plane)
        for v_idx in face.vertices:
            quadrics[v_idx] += plane_quadric

    # Priority queue of edge collapses sorted by error
    pq = PriorityQueue()
    for edge in mesh.edges:
        error = compute_edge_error(edge, quadrics)
        pq.push(edge, error)

    while mesh.triangle_count > target_triangles and not pq.empty():
        edge = pq.pop_min()
        if edge.is_valid(mesh):
            collapse_edge(edge, mesh, quadrics)
            update_neighbors(edge, mesh, quadrics, pq)

    return mesh

def compute_edge_error(edge, quadrics):
    v1, v2 = edge.vertices
    # Error of collapsing v2 into v1
    p1 = v1.position_homogeneous()
    Q = quadrics[v1] + quadrics[v2]
    return p1.T @ Q @ p1
```

## Subtopics

### FQMR Simplification
A non-topology preserving variant derived from Fast Quadric Mesh Reduction (FQMR), useful for multi-resolution meshes where visual appearance is prioritized over connectivity [S1, unofficial]. It adapts the original QEM algorithm to allow non-manifold operations, making it faster and more robust for large-scale simplification [S1, unofficial]. It is implemented in libraries like `zmesh` via `simplify_fqmr` [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \log N)$ or $O(N)$ depending on implementation, where $N$ is the number of triangles.
*   **Memory Complexity**: $O(N)$ for storing quadrics and edge errors.
*   **Performance**: In `zmesh`, a reduction factor of 100 with max error 40 processes ~14.4 labels per second on a specific hardware setup, compared to 433.5 for the original mesh [S1, unofficial].
*   **Strengths**: High visual fidelity, fast, handles large meshes well.
*   **Limitations**: Does not preserve topology, may produce non-manifold geometry.

## Background
Introduced by M. Garland and P. Heckbert in 1997 in "Surface simplification using quadric error metrics" [S1, unofficial]. Extended by H. Hoppe in 1999 with "New Quadric Metric for Simplifying Meshes with Appearance Attributes" [S1, unofficial]. The algorithm is widely used in computer graphics for level-of-detail generation and mesh compression.

## Sources
[S1] seung-lab/zmesh, https://github.com/seung-lab/zmesh, fetched 2026-10-26 (unofficial)
