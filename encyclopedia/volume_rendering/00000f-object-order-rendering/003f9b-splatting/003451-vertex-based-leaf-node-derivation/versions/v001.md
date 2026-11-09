# Vertex-Based Leaf Node Derivation

Also called: QSplat Strategy

## Overview
Vertex-Based Leaf Node Derivation is a method for constructing leaf nodes in hierarchical volume rendering trees by mapping mesh vertices directly to bounding spheres [S1, unofficial]. It solves the problem of defining geometric primitives for rendering when the underlying data is a vertex cloud, lacking explicit topological connectivity for primitive generation [S1, unofficial]. It sets itself apart from sibling methods like TetSplat by deriving centers and field values directly from vertex attributes rather than computing circum-centers of tetrahedra or triangles, requiring explicit radius calculation based on neighbor topology to prevent rendering holes [S1, unofficial].

## How it works
**Inputs:** A set of mesh vertices $\mathcal{V}$, where each vertex $v_i$ has position $\mathbf{p}_i$ and field values $\mathbf{f}_i$. A set of adjacent triangles $\mathcal{T}$ or tetrahedra $\mathcal{U}$ defining the local topology around each vertex.
**Outputs:** A list of leaf nodes, each defined by a center $\mathbf{c}_i$, radius $r_i$, normal $\mathbf{n}_i$ (for Shell nodes), and field values $\mathbf{f}_i$.

**Data Structures:**
*   **Leaf Node:** A struct containing:
    *   `center`: `vec3` (position $\mathbf{c}_i$)
    *   `radius`: `float` ($r_i$)
    *   `normal`: `vec3` ($\mathbf{n}_i$, only for Shell)
    *   `field_values`: `array` or `struct` (interpolated or vertex attributes $\mathbf{f}_i$)

**Steps:**
1.  **Initialize Leaf Nodes:** For each vertex $v_i \in \mathcal{V}$, create a leaf node.
2.  **Set Center and Fields:**
    *   Set center $\mathbf{c}_i = \mathbf{p}_i$.
    *   Set field values $\mathbf{f}_i$ directly from the vertex attributes. [G] If field values are not stored per-vertex, they must be interpolated from adjacent elements, but the source specifies direct assignment for QSplat [S1, unofficial].
3.  **Calculate Radius:**
    *   Identify all triangles $\mathcal{T}_i$ and/or tetrahedra $\mathcal{U}_i$ that share vertex $v_i$.
    *   Compute the circum-radius $r_{tri}$ for each triangle in $\mathcal{T}_i$.
    *   Compute the circum-radius $r_{tet}$ for each tetrahedron in $\mathcal{U}_i$ (if available).
    *   Set $r_i = \max(\{r_{tri} \mid t \in \mathcal{T}_i\} \cup \{r_{tet} \mid u \in \mathcal{U}_i\})$.
    *   *Note:* Using tetrahedra is more conservative (larger radius) than using triangles [S1, unofficial].
    *   *Formula for Triangle Circum-radius:* For a triangle with vertices $\mathbf{a}, \mathbf{b}, \mathbf{c}$, the circum-center $\mathbf{q}$ and radius $r$ are found by solving $|\mathbf{q}-\mathbf{a}|^2 = |\mathbf{q}-\mathbf{b}|^2 = |\mathbf{q}-\mathbf{c}|^2 = r^2$.
4.  **Calculate Normal (Shell Only):**
    *   For vertices belonging to the Shell (surface), compute the normal $\mathbf{n}_i$ as the normalized average of the face normals of all adjacent triangles.
    *   $\mathbf{n}_i = \text{normalize}\left(\sum_{t \in \mathcal{T}_i} \mathbf{n}_t\right)$.
5.  **Construct Hierarchy:** Group these leaf nodes into a hierarchical tree structure (e.g., bounding sphere hierarchy) for culling and traversal [S1, unofficial].

**Parameters:**
*   **Radius Strategy:** "Triangles" or "Tetrahedra". Tetrahedra is safer but computationally more expensive if tetrahedral data is not pre-computed [S1, unofficial].
*   **Typical Values:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Holes:** If the radius is too small, gaps appear between splats. The radius must be large enough that bounding spheres touch or overlap for connected vertices [S1, unofficial].
*   **Degenerate Elements:** The source mentions removing degenerate tetrahedra (needles, spindles) during mesh preprocessing, which affects the set of adjacent elements used for radius calculation [S1, unofficial].
*   **Topology Dependency:** Unlike pure point clouds, this method requires access to the adjacency graph (triangles/tetrahedra) to define the radius, despite the leaf node itself being vertex-based [S1, unofficial].

## Subtopics

### TetSplat Leaf Node Derivation
A variant where leaf nodes are derived from Shell and Solid sub-meshes. Each leaf node requires a bounding sphere (center and radius) and storage for all field values of interest. It explicitly distinguishes between Shell (surface) and Solid (volume) components, requiring normal vectors for Shell nodes and field value storage for both [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Radius calculation requires iterating over adjacent elements. For a vertex with degree $d$, computing circum-radii is $O(d)$. Total preprocessing is $O(N \cdot d_{avg})$.
*   **Memory Complexity:** $O(N)$ for leaf nodes, plus storage for adjacency information.
*   **Strengths:** Simple implementation; leverages existing vertex data; avoids complex geometric primitive construction (tetrahedra/triangles) for the leaf node definition itself.
*   **Limitations:** Requires topology for radius calculation; radius may be larger than necessary if using tetrahedra, leading to overdraw; normals are approximate (averaged) for Shell nodes.
*   **When to use:** When the input is a vertex-based representation and a quick hierarchical structure is needed, or when implementing the original QSplat algorithm.
*   **When not to use:** When precise geometric primitives (tetrahedra/triangles) are required for accurate field interpolation or when minimizing overdraw is critical and tetrahedral data is available.

## Background
The method originates from the QSplat algorithm, which uses vertices to define leaf nodes in a hierarchical bounding sphere structure for real-time volume rendering [S1, unofficial]. It was developed to handle large datasets by enabling efficient culling and adaptive traversal. The approach is part of the broader TetSplat framework, which partitions tetrahedral meshes into Shell and Solid components for rendering [S1, unofficial].

## Sources
[S1] TetSplat: Real-time Rendering and Volume Clipping of Large ... - Museth, https://ken.museth.org/Publications_files/Museth-Lombeyda_VIS04.pdf, fetched 2026-10-28 (unofficial)
