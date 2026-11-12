# Adaptive Delaunay Update

Also called: Delaunay Update Frequency

## Overview
Adaptive Delaunay Update is a strategy for maintaining the topological connectivity of a tetrahedral mesh during optimization-based volume rendering, specifically when vertex positions change dynamically. It addresses the problem of keeping the mesh valid (Delaunay) without incurring the prohibitive cost of full recomputation at every iteration. It differs from static triangulation methods by incrementally updating topology based on vertex movement thresholds, balancing GPU acceleration speed with geometric accuracy [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A current valid Delaunay tetrahedral mesh, a set of vertex positions $\{v_i\}$, and an iteration counter.
*   **Output:** An updated Delaunay tetrahedral mesh reflecting the new vertex positions.

**Data Structures:**
*   **Mesh Topology:** Stores tetrahedra defined by four vertex indices.
*   **Vertex Positions:** 3D coordinates for each vertex.
*   **Iteration Counter:** Tracks the number of optimization steps.

**Parameters:**
*   **Update Frequency ($F$):** The number of optimization iterations between full Delaunay recomputations. Typical value: $F = 10$ [S1, unofficial].
*   **Vertex Displacement ($\epsilon$):** The magnitude of vertex movement during optimization. If a vertex moves by $\epsilon$, it may enter the circumsphere of a neighboring tetrahedron [S1, unofficial].

**Steps:**
1.  **Optimization Step:** Update vertex positions $\{v_i\}$ based on gradient descent or other optimization routines. Let the displacement be $\epsilon$ [S1, unofficial].
2.  **Check Update Condition:** If the current iteration number is divisible by the update frequency $F$, proceed to Step 3. Otherwise, skip to Step 5 [S1, unofficial].
3.  **Detect Topological Violations:** Identify tetrahedra where the Delaunay condition is violated. A tetrahedron is non-Delaunay if a vertex lies inside its circumsphere [S1, unofficial].
4.  **Perform Topological Flips:** Execute edge flips (Pachner moves in 3D) to restore the Delaunay property. Specifically, if a vertex enters the circumsphere of a neighboring tetrahedron, trigger a flip involving the shared edge and the two tetrahedra [S1, unofficial].
5.  **Render:** Use the current mesh topology for volume rendering. Note that between updates, the mesh topology remains static despite vertex movement [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_delaunay_update(mesh, vertices, iteration, update_frequency=10):
    """
    Updates the Delaunay mesh topology if the iteration count matches the frequency.
    
    Args:
        mesh: Current tetrahedral mesh structure.
        vertices: Array of 3D vertex positions.
        iteration: Current optimization iteration number.
        update_frequency: Number of iterations between topology updates.
        
    Returns:
        Updated mesh and vertices.
    """
    # Step 1: Vertex positions are updated by the optimizer before this function is called.
    # Let epsilon be the displacement applied to vertices.
    
    # Step 2: Check if update is needed
    if iteration % update_frequency == 0:
        # Step 3: Detect violations
        # Identify tetrahedra where a vertex is inside the circumsphere of a neighbor.
        # This requires computing circumspheres for each tetrahedron.
        violated_tets = []
        for tet in mesh.tetrahedra:
            circumsphere = compute_circumsphere(tet, vertices)
            for neighbor in mesh.get_neighbors(tet):
                if is_inside_circumsphere(neighbor.vertex, circumsphere):
                    violated_tets.append((tet, neighbor))
        
        # Step 4: Perform flips (Pachner moves)
        # Flip edges to restore Delaunay condition.
        # Note: Flips involve points that are nearly cospherical [S1, unofficial].
        for tet, neighbor in violated_tets:
            flip_edge(tet, neighbor, mesh, vertices)
            
        # Recompute mesh connectivity explicitly if the flip function doesn't do it
        mesh.rebuild_connectivity()
        
    return mesh, vertices

def compute_circumsphere(tet, vertices):
    """
    Computes the circumsphere of a tetrahedron defined by 4 vertices.
    Returns center and radius squared.
    """
    # Standard geometric calculation for circumsphere center and radius
    # Given vertices v1, v2, v3, v4
    v1, v2, v3, v4 = vertices[tet.v1], vertices[tet.v2], vertices[tet.v3], vertices[tet.v4]
    
    # Calculate center using determinant formula or linear algebra
    # D = 2 * det([v2-v1, v3-v1, v4-v1])
    # Center = v1 + ... (complex formula involving cross products)
    # For implementation, use a robust geometric library or explicit formula.
    # [G] Explicit formula omitted for brevity, but standard in computational geometry.
    center, radius_sq = calculate_circumsphere_center_and_radius(v1, v2, v3, v4)
    return {'center': center, 'radius_sq': radius_sq}

def is_inside_circumsphere(vertex, circumsphere):
    """
    Checks if a vertex is inside a circumsphere.
    """
    dist_sq = distance_squared(vertex, circumsphere['center'])
    return dist_sq < circumsphere['radius_sq']

def flip_edge(tet, neighbor, mesh, vertices):
    """
    Performs a Pachner move (edge flip) to restore Delaunay property.
    """
    # Identify the shared edge and the two tetrahedra
    # Replace the two tetrahedra with two new ones sharing the new diagonal
    # [G] Specific implementation depends on mesh data structure.
    # Generally involves removing old tets and adding new ones with updated vertex indices.
    pass
```

**Edge Cases and Pitfalls:**
*   **Nearly Cospherical Points:** Flips involve sets of points that are nearly cospherical because the vertex was already nearby the circumsphere boundary prior to the move. The geometric configurations before and after the flip share an almost identical circumsphere [S1, unofficial]. This can lead to numerical instability if precision is insufficient.
*   **Stale Topology:** Between updates, the mesh topology is static. If vertices move significantly, the mesh may become highly distorted or non-Delaunay, potentially affecting rendering quality until the next update [S1, unofficial].

## Subtopics

### Delaunay Edge Flipping (Pachner Moves)
A method to form a Delaunay triangulation by flipping edges of triangles that are not Delaunay, specifically those where the circumsphere contains a vertex. In 3D, these edge-flips are called Pachner moves [S1, unofficial]. During optimization, if a vertex moves by a small amount $\epsilon$ and enters the circumsphere of a neighboring tetrahedron, a topological flip is triggered [S1, unofficial]. A key pitfall is that these flips involve sets of points that are nearly cospherical, as the vertex was already nearby the circumsphere boundary prior to the move, and the geometric configurations before and after the flip share an almost identical circumsphere [S1, unofficial].

### Hybrid Rendering with Multi-Resolution Tetrahedral Mesh
Extends multi-triangulation to support multiple visualization techniques simultaneously within a single data structure, allowing independent mesh adaptation per technique [S2]. It stores one Directed Acyclic Graph (DAG) per technique to control refinement and coarsening. Each tetrahedron stores three face-incidency entries per face (one for each technique), and the cross-patch face hash map is tripled [S2]. This allows the renderer to adapt the mesh to each technique independently, enabling direct volume rendering to use fewer tetrahedra than isosurface extraction [S2].

### Delaunay Cover Refinement
An iterative refinement process for the Delaunay cover. It refines triangles using a 1-to-4 triangle split if any side length is greater than $h$. New vertices are midpoints of triangle edges. The process repeats until all sides are less than $h$. The refinement sequence is $DC_0(P) \rightarrow DC_1(P) \rightarrow \dots \rightarrow DC_\kappa(P)$, producing a refined point cloud $P_\kappa$ [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Full Delaunay triangulation is $O(N \log N)$ on average, where $N$ is the number of vertices. By updating only every $F$ iterations, the amortized cost per iteration is reduced. Incremental updates (flips) are local and faster than global recomputation [S1, unofficial].
*   **Memory Complexity:** Depends on the mesh storage. Hybrid rendering with multi-resolution adds overhead by tripling face-incidency entries and hash maps [S2].
*   **Measured Performance:** On a scene with 1.2 million tetrahedra, rendering takes 19 ms, instant-NGP query takes 13 ms, backwards pass takes 143 ms, and Delaunay triangulation computation takes 92 ms [S1, unofficial].
*   **Strengths:** Balances GPU acceleration speed with accuracy by avoiding frequent full recomputations [S1, unofficial].
*   **Limitations:** Topology is static between updates, which may lead to distorted elements if vertices move significantly. Numerical instability can occur during flips due to nearly cospherical points [S1, unofficial].
*   **When to Use:** When vertex positions change frequently during optimization, but full recomputation is too slow. Suitable for real-time or near-real-time volume rendering with dynamic meshes [S1, unofficial].
*   **When Not to Use:** When high geometric fidelity is required at every iteration, or when vertex movements are large and unpredictable, leading to excessive topological changes [S1, unofficial].

## Background
Adaptive Delaunay Update is primarily discussed in the context of Radiance Meshes for volumetric reconstruction, where it is used to maintain the Delaunay property of a tetrahedral mesh during optimization [S1, unofficial]. The concept of forming Delaunay triangulations by flipping edges (Pachner moves in 3D) is a classical computational geometry technique [S1, unofficial]. The specific application to adaptive updating in volume rendering with a fixed frequency (e.g., every 10 iterations) is detailed in recent work on Radiance Meshes [S1, unofficial].

## Sources
[S1] Radiance Meshes for Volumetric Reconstruction, https://arxiv.org/abs/2512.04076v1, fetched 2026-10-26 (unofficial)
[S2] Segment-based tetrahedral meshing and rendering, https://doi.org/10.1145/1174429.1174507, fetched 2026-11-06
[S3] , https://www.scitepress.org/PublishedPapers/2007/20798/pdf/index.html, fetched 2026-10-27 (unofficial)
