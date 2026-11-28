# Geometric Mesh Refinement

Also called: Curvature-Based Refinement, Geometric Mesh Creation

## Overview
Geometric Mesh Refinement is a post-processing step that creates a high-detail geometric mesh from a coarse mechanical mesh by refining edges and triangles based on normal angle differences [S1][S2, unofficial]. It solves the problem of insufficient surface resolution in implicit surface rendering by adaptively subdividing triangles where the surface curvature is high [S1]. Unlike red-green schemes which may not be anisotropic, this method produces T-vertex-free, anisotropic geometry suited to curvature directions, although it introduces "ghost" points (T-vertices) in specific split cases [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A coarse mechanical mesh (initial geometric mesh), a scalar field function $F(\mathbf{x})$ defining the implicit surface, a user-defined normal angle threshold $\alpha_r$, and a maximum number of refinement passes $N_p$.
*   **Output:** A refined geometric mesh with higher triangle density in high-curvature regions.

**Data Structures:**
*   **Triangle $\tau$:** Defined by three vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$.
*   **Normals:** Stored for each vertex and triangle centroid.
*   **Edge Status:** Each edge is marked for refinement or not.

**Formulas:**
*   **Gradient/Normal:** The normal at a point $\mathbf{x}$ is computed as the gradient of the field function: $\mathbf{n}(\mathbf{x}) = \nabla F(\mathbf{x}) = \frac{\partial F}{\partial x}\mathbf{i} + \frac{\partial F}{\partial y}\mathbf{j} + \frac{\partial F}{\partial z}\mathbf{k}$ [S1].
*   **Angle between normals:** For two vectors $\mathbf{a}$ and $\mathbf{b}$, the angle $\theta$ is $\theta = \arccos\left(\frac{\mathbf{a} \cdot \mathbf{b}}{\|\mathbf{a}\| \|\mathbf{b}\|}\right)$ [S1][S2, unofficial].
*   **Newton-Raphson Projection:** To project a point $\mathbf{p}$ onto the surface $F(\mathbf{x})=0$, iterate: $\mathbf{p}_{new} = \mathbf{p}_{old} - \frac{F(\mathbf{p}_{old})}{\|\nabla F(\mathbf{p}_{old})\|^2} \nabla F(\mathbf{p}_{old})$ [S1][S2, unofficial].

**Parameters:**
*   $\alpha_r$: User-defined threshold for the angle between normals. If the angle exceeds $\alpha_r$, refinement is triggered. Typical values range from $18^\circ$ to $60^\circ$ [S1][S2, unofficial]. Minimal value $-\pi$ implies no refinement [S1][S2, unofficial].
*   $N_p$: Maximum number of refinement passes. Limits recursion to avoid infinite refinement in sharp areas [S1][S2, unofficial].

**Steps:**
1.  **Initialization:** Create an initial geometric mesh by copying the mechanical mesh [S1][S2, unofficial].
2.  **Pass Loop:** Repeat the following refinement pass up to $N_p$ times [S1]:
    3.  **Normal Computation:** Compute normals by measuring the gradient $\nabla F$ at all vertices and triangle centroids of the current mesh [S1].
    4.  **Edge Marking:** For each triangle $\tau$ with edges $e_0, e_1, e_2$:
        *   For each edge $e_i$ connecting vertices $\mathbf{u}$ and $\mathbf{v}$: If the angle between $\mathbf{n}(\mathbf{u})$ and $\mathbf{n}(\mathbf{v})$ is $> \alpha_r$, mark $e_i$ for refinement [S1][S2, unofficial].
        *   For each edge $e_i$ shared with a neighboring triangle $\tau'$: If the angle between the normal of $\tau$ and the normal of $\tau'$ is $> \alpha_r$, mark the *other* two edges of $\tau$ (not $e_i$) for refinement [S1].
    5.  **Refinement Decision:** For each triangle $\tau$:
        *   If no edges are marked, skip $\tau$ [S2, unofficial].
        *   If one edge is marked, split $\tau$ into two triangles by inserting a new vertex at the midpoint of the marked edge [S2, unofficial].
        *   If two edges are marked, split $\tau$ into four triangles. This involves refining the third (unmarked) edge as well to maintain proper triangulation. The new vertex for this third edge is a "ghost" point (not projected to the surface) to match the neighboring triangle [S2, unofficial].
        *   If three edges are marked, split $\tau$ into four triangles (standard barycentric split) [S1][S2, unofficial].
        *   *Note on Two-Edge Split:* If two edges need refinement, choose the split direction that best fits the local curvature direction [S1].
    6.  **Vertex Projection:** For every newly created vertex that is *not* a ghost point, project its position onto the implicit surface using Newton-Raphson iteration [S1][S2, unofficial].
    7.  **Recursion:** Call `REFINE` recursively on all newly created triangles [S2, unofficial].
        *   *Guard:* Limit recursive depth to avoid infinite recursion in degenerate areas [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_normal(field_func, point):
    """Compute normal as gradient of field function at point."""
    # Numerical gradient or analytical gradient depending on field_func
    eps = 1e-5
    grad = [0.0, 0.0, 0.0]
    for i in range(3):
        p_plus = point.copy()
        p_minus = point.copy()
        p_plus[i] += eps
        p_minus[i] -= eps
        grad[i] = (field_func(p_plus) - field_func(p_minus)) / (2 * eps)
    return normalize(grad)

def project_to_surface(field_func, point, max_iter=10, tol=1e-4):
    """Project point onto implicit surface F(x)=0 using Newton-Raphson."""
    current = point
    for _ in range(max_iter):
        f_val = field_func(current)
        if abs(f_val) < tol:
            break
        grad = compute_normal(field_func, current)
        grad_norm_sq = dot(grad, grad)
        if grad_norm_sq == 0:
            break
        displacement = (f_val / grad_norm_sq) * grad
        current = current - displacement
    return current

def angle_between(v1, v2):
    """Compute angle in radians between two vectors."""
    dot_prod = dot(v1, v2)
    norm_prod = norm(v1) * norm(v2)
    if norm_prod == 0:
        return 0.0
    cos_angle = dot_prod / norm_prod
    # Clamp to [-1, 1] to avoid domain errors
    cos_angle = max(-1.0, min(1.0, cos_angle))
    return acos(cos_angle)

def refine_triangle(triangle, field_func, alpha_r, max_depth, current_depth):
    """
    Recursively refine a triangle based on normal angles.
    triangle: object with vertices [v0, v1, v2] and neighbors [n0, n1, n2]
    Returns list of new triangles.
    """
    if current_depth >= max_depth:
        return [triangle]

    # 1. Compute normals
    normals_v = [compute_normal(field_func, v) for v in triangle.vertices]
    centroid = sum(triangle.vertices) / 3.0
    normal_t = compute_normal(field_func, centroid)
    
    # Neighbor normals (assume available or computed)
    # Note: In a full implementation, neighbor normals must be fetched
    neighbor_normals = [compute_normal(field_func, n.centroid) for n in triangle.neighbors]

    # 2. Mark edges
    # Edges: e0=(v0,v1), e1=(v1,v2), e2=(v2,v0)
    # Neighbors: n0 opposite v0 (across e0), etc.
    marked_edges = [False, False, False]

    # Check vertex normal angles
    for i in range(3):
        v1_idx = i
        v2_idx = (i + 1) % 3
        if angle_between(normals_v[v1_idx], normals_v[v2_idx]) > alpha_r:
            marked_edges[i] = True

    # Check triangle normal angles
    for i in range(3):
        # If angle between current triangle and neighbor i is high,
        # mark the OTHER edges of current triangle
        if angle_between(normal_t, neighbor_normals[i]) > alpha_r:
            # Mark edges not shared with neighbor i
            # Neighbor i is across edge i. So mark (i+1)%3 and (i+2)%3
            marked_edges[(i + 1) % 3] = True
            marked_edges[(i + 2) % 3] = True

    # 3. Split
    new_triangles = []
    if not any(marked_edges):
        return [triangle]

    # Determine split type
    num_marked = sum(marked_edges)
    
    # Create new vertices
    new_verts = [None, None, None]
    is_ghost = [False, False, False]

    for i in range(3):
        if marked_edges[i]:
            mid = (triangle.vertices[i] + triangle.vertices[(i + 1) % 3]) / 2.0
            # Check if this edge is shared with a neighbor that is also being refined
            # In the "two edges marked" case, the third edge is marked but NOT projected
            # This logic is simplified; full implementation requires checking neighbor state
            # For simplicity, we assume the "ghost" logic is handled by the caller or specific case
            
            # Standard projection for most cases
            projected = project_to_surface(field_func, mid)
            new_verts[i] = projected
            is_ghost[i] = False

    # Special case: Two edges marked
    if num_marked == 2:
        # Find the unmarked edge index
        unmarked_idx = marked_edges.index(False)
        # The new vertex for the unmarked edge is a GHOST point
        # It is placed at the midpoint but NOT projected
        mid_unmarked = (triangle.vertices[unmarked_idx] + triangle.vertices[(unmarked_idx + 1) % 3]) / 2.0
        new_verts[unmarked_idx] = mid_unmarked
        is_ghost[unmarked_idx] = True
        
        # Choose split direction that best fits local curvature
        # This usually means splitting the triangle into 4 parts
        # Vertices: v0, v1, v2, mid0, mid1, mid2
        # But one mid is ghost
        
        # Construct 4 triangles
        # Example: edges 0 and 1 marked, edge 2 unmarked (ghost)
        # Triangles: (v0, mid0, mid2_ghost), (v1, mid1, mid0), (v2, mid2_ghost, mid1), (mid0, mid1, mid2_ghost)
        # Note: Exact topology depends on which edges are marked.
        # General formula for 4 triangles from midpoints m0, m1, m2:
        # T1: v0, m0, m2
        # T2: v1, m1, m0
        # T3: v2, m2, m1
        # T4: m0, m1, m2
        
        m0 = new_verts[0]
        m1 = new_verts[1]
        m2 = new_verts[2]
        
        new_triangles.append(Triangle([triangle.vertices[0], m0, m2], neighbors=[]))
        new_triangles.append(Triangle([triangle.vertices[1], m1, m0], neighbors=[]))
        new_triangles.append(Triangle([triangle.vertices[2], m2, m1], neighbors=[]))
        new_triangles.append(Triangle([m0, m1, m2], neighbors=[]))
        
        # Recurse
        return [refine_triangle(t, field_func, alpha_r, max_depth, current_depth + 1) for t in new_triangles]

    # One edge marked
    elif num_marked == 1:
        marked_idx = marked_edges.index(True)
        m = new_verts[marked_idx]
        # Split into 2 triangles
        # Edge i is (vi, v(i+1)%3). Split into (vi, v(i+1)%3, m) is wrong.
        # Split into (v0, v1, m) and (v0, m, v2) if edge 0 (v0-v1) is marked?
        # Correct split for edge i (vi, v(i+1)%3):
        # Triangles: (vi, v(i+1)%3, m) -> No, m is on the edge.
        # Triangles: (v0, v1, m) and (v0, m, v2) if m is on v0-v1?
        # Actually: Triangle (v0, v1, v2). Edge 0 is v0-v1. Mid m0.
        # New Triangles: (v0, v1, m0) is degenerate.
        # Correct: (v0, m0, v2) and (v1, m0, v2)
        
        v0, v1, v2 = triangle.vertices
        m0, m1, m2 = new_verts[0], new_verts[1], new_verts[2]
        
        if marked_idx == 0:
            new_triangles.append(Triangle([v0, m0, v2], neighbors=[]))
            new_triangles.append(Triangle([v1, m0, v2], neighbors=[]))
        elif marked_idx == 1:
            new_triangles.append(Triangle([v0, m1, v1], neighbors=[])) # Wait, v1 is vertex
            # Edge 1 is v1-v2. Mid m1.
            # Triangles: (v0, v1, m1) and (v0, m1, v2)
            new_triangles.append(Triangle([v0, v1, m1], neighbors=[]))
            new_triangles.append(Triangle([v0, m1, v2], neighbors=[]))
        elif marked_idx == 2:
            # Edge 2 is v2-v0. Mid m2.
            # Triangles: (v0, v1, m2) and (v1, v2, m2)
            new_triangles.append(Triangle([v0, v1, m2], neighbors=[]))
            new_triangles.append(Triangle([v1, v2, m2], neighbors=[]))
            
        return [refine_triangle(t, field_func, alpha_r, max_depth, current_depth + 1) for t in new_triangles]

    # Three edges marked
    else:
        m0, m1, m2 = new_verts
        new_triangles.append(Triangle([triangle.vertices[0], m0, m2], neighbors=[]))
        new_triangles.append(Triangle([triangle.vertices[1], m1, m0], neighbors=[]))
        new_triangles.append(Triangle([triangle.vertices[2], m2, m1], neighbors=[]))
        new_triangles.append(Triangle([m0, m1, m2], neighbors=[]))
        return [refine_triangle(t, field_func, alpha_r, max_depth, current_depth + 1) for t in new_triangles]

    return new_triangles
```

**Edge Cases and Pitfalls:**
*   **Ghost Points (T-Vertices):** When two edges are refined, the third edge is refined with a "ghost" point (not projected) to match the neighbor. These are T-vertices and can be noticeable when zooming very close [S2, unofficial].
*   **Missed Refinement:** The normal-based criterion may miss areas needing refinement (e.g., gaps or hernias) if all normals are aligned. Solutions include using a finer mechanical resolution or a Hessian-based criterion [S1][S2, unofficial].
*   **Infinite Recursion:** Recursive depth must be limited to avoid infinite recursion in degenerate areas [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of refinement passes $N_p$ and the refinement rate. Each pass involves gradient computations and Newton-Raphson iterations. Newton-Raphson convergence is generally quadratic but depends on the initial guess [S1].
*   **Memory Complexity:** Increases with the number of triangles. The mesh grows adaptively.
*   **Strengths:** Produces anisotropic geometry suited to curvature directions; T-vertex-free in most cases (except the specific two-edge split case) [S1].
*   **Limitations:** May miss refinement in regions with aligned normals (e.g., hernias) [S1][S2, unofficial]. Introduces T-vertices in specific cases [S2, unofficial]. Requires gradient computation for every vertex and centroid [S1].
*   **When to use:** When high-quality, curvature-adaptive meshing is required from a coarse mechanical mesh, and T-vertices are acceptable or can be handled by post-processing.
*   **When not to use:** When strict T-vertex-free meshes are required (use red-green refinement instead) [S1][S2, unofficial]. When the field function gradient is expensive or unavailable.

## Background
Geometric Mesh Refinement was introduced in the context of dynamic triangulation of implicit surfaces, particularly for handling topology changes and large deformations [S1][S2, unofficial]. It was developed to provide a high-quality geometric mesh from a coarse mechanical mesh (e.g., from a finite element method) without requiring a globally fine mesh [S1]. The method is part of a pipeline that includes mechanical simulation and implicit surface extraction [S1].

## Sources
[S1] Twinned meshes for dynamic triangulation of implicit surfaces, https://doi.org/10.1145/1268517.1268521, fetched 2026-10-26
[S2] Dynamic Triangulation of Implicit Surfaces: towards the handling of topology changes, https://inria.hal.science/inria-00132537, fetched 2026-10-26 (unofficial)
