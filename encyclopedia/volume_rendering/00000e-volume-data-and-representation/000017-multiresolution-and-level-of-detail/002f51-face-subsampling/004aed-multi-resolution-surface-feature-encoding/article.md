# Multi-resolution Surface Feature Encoding

Also called: Per-face LOD feature encoding

## Overview
Multi-resolution Surface Feature Encoding is a strategy to increase feature density in regions requiring high-frequency detail by creating virtual vertices within mesh faces, controlled by a per-face Level of Detail (LOD) factor $k_i$, without modifying the actual mesh topology [S1, unofficial]. It addresses the limitation of sparse vertex distributions in neural global illumination, where insufficient geometric resolution hinders the representation of high-frequency illumination details [S1, unofficial]. Unlike standard barycentric interpolation which uses only the three mesh vertices, this method subdivides each face into a regular triangular grid of virtual vertices, allowing for adaptive refinement based on training loss [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A mesh geometry defined by triangles $F$, where each triangle $i$ has vertices $v_i, v_j, v_k$ and associated trainable neural features $g_i, g_j, g_k$. A spatial query point $x$ with barycentric coordinates $(u, v)$ relative to triangle $i$. Per-face LOD factors $k_i$ for all triangles.
*   **Output:** The interpolated neural feature $f_x$ at point $x$.

**Data Structures**
*   **LOD Factors:** A scalar $k_i \in \mathbb{Z}^+$ for each triangle $i$. Initial value $k_i = 1$.
*   **Virtual Vertices:** For a triangle with LOD factor $k_i$, the face is subdivided into a regular triangular grid. The number of vertices $N(i)$ associated with the subdivided mesh is given by:
    $$ N(i) = (k_i + 1)(k_i + 2)/2 \times (k_i + 1) \text{ is incorrect, source says: } N(i) = (k_i + 1)^2 (k_i + 2) $$
    *Correction based on source [S1, unofficial]:* The source explicitly states $N(i) = (k_i + 1)^2 (k_i + 2)$. Note: Standard triangular grid vertex count for subdivision level $k$ is usually $(k+1)(k+2)/2$. The source formula $(k_i + 1)^2 (k_i + 2)$ appears to be a specific definition or potential typo in the source text, but we must follow the source exactly. *Re-reading source:* "computed as $N(i) = (k_i + 1)^2 (k_i + 2)$". This is mathematically unusual for a 2D grid (which scales as $k^2$), but we cite the source's explicit formula.
*   **Features:** Trainable neural features $g$ are assigned to each virtual vertex.

**Steps**
1.  **Initialization:** Set all per-face LOD factors $k_i = 1$. This implies no virtual vertices are added initially, and standard barycentric interpolation is used.
2.  **Adaptive Refinement (Training):** Monitor training loss. Adaptively increase $k_i$ for triangles where loss is high. When $k_i$ is increased:
    *   Generate new virtual vertices on the regular triangular grid defined by $k_i$.
    *   Initialize features for new virtual vertices via interpolation from existing representations (features of existing vertices in the previous grid or boundary vertices) [S1, unofficial].
    *   Replace original features with the refined set.
3.  **Query Processing for Point $x$ in Triangle $i$:**
    *   Let $(u, v)$ be the barycentric coordinates of $x$ within triangle $i$.
    *   Compute integer grid coordinates and remainders using the LOD factor $k_i$:
        $$ \bar{u} = \lfloor k_i u \rfloor $$
        $$ \tilde{u} = k_i u - \bar{u} $$
        $$ \bar{v} = \lfloor k_i v \rfloor $$
        $$ \tilde{v} = k_i v - \bar{v} $$
    *   Determine the local barycentric coordinates $(u', v')$ within the virtual sub-triangle containing $x$ based on the sum of remainders:
        $$
        (u', v') =
        \begin{cases}
        (\tilde{u}, \tilde{v}) & \text{if } \tilde{u} + \tilde{v} \le 1 \\
        (1 - \tilde{u}, 1 - \tilde{v}) & \text{otherwise}
        \end{cases}
        $$
    *   Identify the virtual sub-triangle using integer grid coordinates $(\bar{u}, \bar{v})$.
    *   Perform barycentric interpolation using the features of the three corresponding virtual vertices to get the final feature $f_x$.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_resolution_feature_encoding(triangle, x_barycentric_u, x_barycentric_v, lod_factor_k, virtual_features_grid):
    """
    triangle: Object containing vertices and their features.
    x_barycentric_u, x_barycentric_v: Barycentric coordinates (u, v) of query point x.
    lod_factor_k: Integer k_i for the triangle.
    virtual_features_grid: A data structure storing features for all virtual vertices.
                           For k=1, this is just the 3 mesh vertices.
                           For k>1, it contains (k+1)^2*(k+2) vertices? 
                           Note: Source formula N(i) = (k_i + 1)^2 (k_i + 2) is cited.
                           Implementation note: The grid structure mapping (u_bar, v_bar) to specific vertex indices 
                           is not explicitly detailed in the source beyond the grid coordinates. 
                           We assume a standard triangular grid mapping exists.
    """
    k = lod_factor_k
    
    # Step 1: Compute integer grid coordinates and remainders
    u_bar = floor(k * x_barycentric_u)
    u_tilde = k * x_barycentric_u - u_bar
    v_bar = floor(k * x_barycentric_v)
    v_tilde = k * x_barycentric_v - v_bar
    
    # Step 2: Determine local barycentric coordinates (u', v')
    if u_tilde + v_tilde <= 1:
        u_prime = u_tilde
        v_prime = v_tilde
    else:
        u_prime = 1 - u_tilde
        v_prime = 1 - v_tilde
        
    # Step 3: Identify the virtual sub-triangle and its vertices
    # The sub-triangle is identified by integer grid coordinates (u_bar, v_bar).
    # We need to retrieve the features of the three virtual vertices of this sub-triangle.
    # The source does not specify the exact indexing scheme for the virtual_features_grid.
    # However, standard triangular grid subdivision implies a deterministic mapping.
    # Let's assume a helper function get_virtual_vertices(u_bar, v_bar, k) returns the 3 vertex features.
    
    v1_feat, v2_feat, v3_feat = get_virtual_vertices(u_bar, v_bar, k, virtual_features_grid)
    
    # Step 4: Barycentric interpolation
    # The local coordinates (u', v') are used to interpolate between the 3 virtual vertices.
    # The third local coordinate w' is 1 - u' - v'.
    w_prime = 1 - u_prime - v_prime
    
    f_x = w_prime * v1_feat + u_prime * v2_feat + v_prime * v3_feat
    
    return f_x

def get_virtual_vertices(u_bar, v_bar, k, grid):
    """
    Placeholder for the specific indexing logic of the virtual grid.
    The source states vertices are on a regular triangular grid.
    For a given k, the grid has specific vertices.
    The sub-triangle at (u_bar, v_bar) has 3 corners.
    This function must map (u_bar, v_bar) to the 3 vertex indices/features.
    Since the source does not provide the exact indexing formula, 
    this is a necessary implementation detail [G].
    """
    # Example logic for a standard triangular grid (not from source, but standard):
    # Vertices are indexed by (i, j) where 0 <= j <= i <= k.
    # This is a guess [G] as the source lacks the indexing detail.
    pass
```

**Parameters**
*   $k_i$: Per-face LOD factor for triangle $i$. Specifies the number of segments each edge is divided into. Initial value is 1.
*   $\alpha$: A factor controlling the weighted combination of area-based and vertex-based sampling probabilities (used during training/sampling, not direct encoding) [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Precision:** Floating point errors in calculating $\lfloor k u \rfloor$ near boundaries ($u=1$ or $v=1$) can lead to incorrect sub-triangle identification. Guard against $u, v$ slightly outside $[0,1]$ due to numerical noise.
*   **Degenerate Input:** If $k_i$ is very large, the number of virtual vertices grows quadratically, potentially causing memory issues. The source formula $N(i) = (k_i + 1)^2 (k_i + 2)$ suggests cubic growth if taken literally, which is extremely steep. If the source meant $(k_i+1)(k_i+2)/2$, it is quadratic. We stick to the source's explicit formula but note the potential for high memory usage [S1, unofficial].
*   **Initialization:** When increasing $k_i$, new virtual vertices must be initialized. Interpolation from existing features is required to avoid random initialization artifacts [S1, unofficial].

## Subtopics

### [Adaptive Multi-resolution LOD for Vertex Features](12898-adaptive-multi-resolution-lod-for-vertex-features.md)
A strategy that refines features where training loss is high, enabling recovery of high-frequency lighting in undersampled areas by allocating virtual vertices adaptively. It differs by focusing on the adaptive allocation mechanism based on loss statistics, whereas Multi-resolution Surface Feature Encoding defines the geometric subdivision and interpolation logic.

## Operating characteristics
*   **Time Complexity:** Query time is $O(1)$ per point, involving basic arithmetic and interpolation. However, identifying the correct virtual vertices requires a mapping from grid coordinates to vertex indices, which depends on the grid implementation.
*   **Memory Complexity:** The number of features scales with the sum of $N(i)$ over all triangles. For a triangle with LOD $k_i$, $N(i) = (k_i + 1)^2 (k_i + 2)$ [S1, unofficial]. This can lead to significant memory overhead if many faces have high $k_i$.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Allows high-frequency detail recovery without modifying mesh topology. Adaptive refinement focuses computational resources on challenging regions.
*   **Limitations:** Increased memory usage for high LOD factors. Potential for artifacts if interpolation is not smooth across sub-triangle boundaries (though barycentric interpolation ensures continuity within sub-triangles, discontinuities may exist at sub-triangle boundaries if features are not consistent).
*   **When to use:** When neural global illumination requires high-frequency detail representation on sparse meshes.
*   **When not to use:** When memory is constrained and high LOD factors are necessary.

## Background
*   **Origin:** Proposed in "Vertex Features for Neural Global Illumination" by Rui Su et al. [S1, unofficial].
*   **History:** The method addresses the issue of sparse vertex distributions hindering high-frequency illumination details in neural radiosity and global illumination contexts [S1, unofficial].
*   **Usage:** Used in neural global illumination systems to encode spatial information directly at vertices, supporting downstream applications like neural radiosity [S1, unofficial].

## Sources
[S1] [2508.07852] Vertex Features for Neural Global Illumination, https://arxiv.org/abs/2508.07852, fetched 2026-10-27 (unofficial)
