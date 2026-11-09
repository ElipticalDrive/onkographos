# Adaptive Delaunay Update

Also called: Delaunay Update Frequency

## Overview
Adaptive Delaunay Update is a strategy for maintaining the topological validity of a tetrahedral mesh during optimization-based volume rendering, specifically within frameworks like Radiance Meshes. It addresses the problem of mesh degeneracy caused by vertex movement during training by periodically recomputing the Delaunay triangulation to restore the empty-circumsphere property. It sets itself apart from incremental update methods by trading continuous topological accuracy for significant GPU acceleration speed, relying on batched recomputation rather than real-time edge flipping [S1, unofficial].

## How it works
The method operates within an iterative optimization loop (e.g., differentiable rendering training). The core input is a set of tetrahedral vertices with associated attributes (position, color, density) and a current Delaunay topology. The output is an updated mesh topology and vertex positions valid for the next rendering pass.

**Data Structures:**
*   **Vertices:** Array of 3D positions $\mathbf{v}_i$ and attributes.
*   **Topology:** List of tetrahedra, each defined by 4 vertex indices.
*   **Acceleration Structure:** Instant-NGP hash grid for spatial queries [S1, unofficial].

**Parameters:**
*   **Update Frequency ($F$):** The number of optimization iterations between full Delaunay recomputations. Typical value: $F=10$ [S1, unofficial].

**Steps:**
1.  **Initialization:** Construct an initial Delaunay tetrahedralization of the input points.
2.  **Optimization Step:** Perform one iteration of gradient descent. Update vertex positions by a small displacement $\epsilon$.
3.  **Check Condition:** If the current iteration count $k$ is a multiple of $F$ ($k \mod F == 0$):
    a.  **Recompute:** Discard the current tetrahedral connectivity.
    b.  **Triangulate:** Compute a new Delaunay tetrahedralization of the updated vertex positions using a standard 3D Delaunay algorithm.
    c.  **Update Topology:** Replace the old tetrahedra list with the new one.
4.  **Render:** Project the tetrahedra and perform volume rendering using the current topology and vertex attributes.
5.  **Repeat:** Return to Step 2.

**Edge Cases and Pitfalls:**
*   **Cospherical Degeneracy:** When vertices move by $\epsilon$, they may enter the circumsphere of a neighboring tetrahedron. The resulting flips involve points that are nearly cospherical, as the geometric configurations before and after the flip share an almost identical circumsphere [S1, unofficial]. This can lead to numerical instability in standard incremental flip algorithms, justifying the periodic full recomputation approach.
*   **Staleness:** Between updates, the mesh is not strictly Delaunay. This may cause minor topological inaccuracies in ray-tetrahedron intersection or sorting, but the small $\epsilon$ ensures visual continuity [S1, unofficial].

## Subtopics

### Delaunay Edge Flipping (Pachner Moves)
A method to form a Delaunay triangulation by flipping edges of triangles that are not Delaunay, specifically those where the circumsphere contains a vertex. In 3D, these edge-flips are called Pachner moves [S1, unofficial]. During optimization, if a vertex moves by a small amount $\epsilon$ and enters the circumsphere of a neighboring tetrahedron, a topological flip is triggered. The pitfall is that these flips involve sets of points that are nearly cospherical because the vertex was already nearby the circumsphere boundary prior to the move, and the geometric configurations before and after the flip share an almost identical circumsphere [S1, unofficial].

### Hybrid Rendering with Multi-Resolution Tetrahedral Mesh
Extends multi-triangulation to support multiple visualization techniques simultaneously within a single data structure, allowing independent mesh adaptation per technique [S2]. It stores one Directed Acyclic Graph (DAG) per technique to control refinement and coarsening. Each tetrahedron stores three face-incidency entries per face (one for each technique). The cross-patch face hash map is tripled. This allows the renderer to adapt the mesh to each technique independently, enabling direct volume rendering to use fewer tetrahedra than isosurface extraction [S2].

## Operating characteristics
*   **Time Complexity:** The cost is dominated by the periodic Delaunay triangulation. For a scene with 1.2 million tetrahedra, the Delaunay computation takes approximately 92 ms [S1, unofficial]. Rendering takes ~19 ms, and the backward pass ~143 ms [S1, unofficial].
*   **Memory Complexity:** Linear in the number of tetrahedra and vertices. Hybrid rendering variants increase memory usage by a factor of 3 for face-incidency data and hash maps [S2].
*   **Strengths:** Significantly faster than incremental updating on GPU due to batched computation and avoidance of complex conditional logic for edge flips. Balances GPU acceleration speed with accuracy [S1, unofficial].
*   **Limitations:** The mesh is not Delaunay between updates. Requires careful tuning of the update frequency $F$ to balance visual quality and performance.
*   **When to use:** When using differentiable rendering with mesh optimization (e.g., Radiance Meshes) where full topological correctness is less critical than training speed.
*   **When not to use:** When strict topological validity is required at every frame (e.g., for precise geometric analysis) or when the update frequency cannot be amortized over multiple iterations.

## Background
The concept originates from the Radiance Meshes framework for volumetric reconstruction, which decouples topology (Delaunay mesh) from attribute parameterization (Instant-NGP) [S1, unofficial]. The specific update frequency strategy was introduced to mitigate the slowness of GPU-accelerated Delaunay recomputation, which is still relatively expensive compared to rendering [S1, unofficial]. The underlying geometric principle of using circumcircles for visibility sorting in Delaunay meshes was also detailed in the same work [S1, unofficial].

## Sources
[S1] Radiance Meshes for Volumetric Reconstruction, https://arxiv.org/abs/2512.04076v1, fetched 2026-10-26 (unofficial)
[S2] Segment-based tetrahedral meshing and rendering, https://doi.org/10.1145/1174429.1174507, fetched 2026-11-06
