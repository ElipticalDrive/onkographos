# Hardware Caching for Projected Tetrahedra

Also called: Hardware Caching, Vertex Buffer Caching for PT

## Overview
Hardware Caching for Projected Tetrahedra optimizes the rendering pipeline by reducing redundant computations during the classification and decomposition of tetrahedral volume cells. It addresses the performance bottleneck of recalculating geometric attributes (edge lengths, face areas, and inverse matrices) for every tetrahedron in a strip or under view changes. It sets itself apart from [On-the-fly Shell Geometry Construction](13927-on-the-fly-shell-geometry-construction.md) by pre-computing and storing these values in buffers rather than generating geometry dynamically, and from [Sorted Rasterization of Projected Tetrahedra](23441-sorted-rasterization-of-projected-tetrahedra.md) by focusing on data reuse rather than depth sorting.

## How it works
The method relies on two primary caching mechanisms: geometric attribute caching for strip traversal and matrix inversion caching for view transformations.

### Inputs and Outputs
*   **Inputs**: Tetrahedral mesh data (vertices, connectivity), current view matrix $F$, and strip traversal order.
*   **Outputs**: Rendered volume with correct shading and classification, utilizing cached buffers to minimize GPU/CPU workload.

### Data Structures
1.  **Vertex Buffer**: A buffer holding the four vertices of the current tetrahedron plus a fifth vertex (the "thick" vertex created by the PT algorithm) [S1, unofficial].
2.  **Attribute Cache**: A storage structure for edge lengths and face areas computed for the current tetrahedron [S1, unofficial].
3.  **Matrix Cache**: A cache storing the inverse tetrahedron matrix $M_T^{-1}$ for each tetrahedron [S2].

### Steps

#### 1. Geometric Attribute Caching (Strip Traversal)
When rendering a tetrahedral strip:
1.  Maintain the vertex buffer containing the current tetrahedron's four vertices and the fifth thick vertex [S1, unofficial].
2.  Swap vertices in the buffer as necessary to follow the direction of the strip [S1, unofficial].
3.  Compute edge lengths and face areas for the current tetrahedron.
4.  Cache these values. Since each tetrahedron in a strip shares edges and faces with its neighbors, these cached values are reused by the next tetrahedron in the strip, avoiding redundant square root calculations [S1, unofficial].

#### 2. View Transform and Matrix Caching
To handle view changes and morphing efficiently:
1.  **Initial Setup**: For each tetrahedron, compute and cache $M_T^{-1}$ [S2].
2.  **View Change Handling**: If the view matrix $F$ changes but the mesh does not, do not recompute $M_T^{-1}$. Instead, use the cached $M_T^{-1}$ combined with the new view matrix [S2].
    *   **Perspective Projection**: Compute source barycentric coordinates $\mathbf{t}$ using:
        $$ \mathbf{t} = M_T^{-1}(F^{-1}\mathbf{s}) $$
        Where $\mathbf{s} = (0, 0, 0, 1)^T$. Since $F^{-1} = [\mathbf{g}_0 \ \mathbf{g}_1 \ \mathbf{g}_2 \ \mathbf{g}_3]$, then $F^{-1}\mathbf{s} = \mathbf{g}_3$. Thus:
        $$ \mathbf{t} = M_T^{-1}\mathbf{g}_3 $$
    *   **Orthographic Projection**: Compute the difference vector $\mathbf{d}$ using:
        $$ \mathbf{d} = M_T^{-1}\mathbf{g}_2 $$
        Where $\mathbf{g}_2$ is the third column of the inverse view matrix $F^{-1}$ [S2].
3.  **Morphing Handling**: If the mesh undergoes morphing (shape deformation), the cached $M_T^{-1}$ values become invalid. Recompute $M_T^{-1}$ for affected tetrahedra using a $4 \times 4$ Gauss-Jordan elimination routine [S2].

### Pitfalls
*   **Square Root Cost**: Computing edge lengths and face areas requires square roots, which significantly impacts running time and complexity. Caching is essential to mitigate this [S1, unofficial].
*   **Morphing Overhead**: Matrix inversions are expensive. If morphing is frequent, the cost of recomputing $M_T^{-1}$ via Gauss-Jordan elimination may offset the benefits of caching [S2].

## Subtopics

### Cached View Transform Inversion
Optimizes rendering by caching the inverse tetrahedron matrix $M_T^{-1}$ for each tetrahedron, avoiding recomputation when the view transform changes but the mesh does not [S2]. When morphing is applied, $M_T^{-1}$ is recomputed for affected tetrahedra using a C++ routine performing $4 \times 4$ Gauss-Jordan elimination [S2]. It is chosen when dynamic view changes are frequent but mesh topology remains static [S2].

## Operating characteristics
*   **Time Complexity**: Reduced by eliminating redundant square root calculations for shared edges/faces in strips [S1, unofficial] and avoiding matrix inversions during view changes [S2].
*   **Memory Complexity**: Increased due to the storage of cached edge lengths, face areas, and inverse matrices for each tetrahedron [S1, unofficial][S2].
*   **Strengths**: Significant performance gains in interactive rendering where view angles change frequently or strips are long [S1, unofficial][S2].
*   **Limitations**: Requires additional memory for caches. Morphing events trigger expensive recomputations [S2].
*   **When to use**: Interactive tetrahedral volume rendering with static meshes and dynamic views [S2].
*   **When not to use**: Scenes with frequent mesh morphing where the cost of recomputing inverses outweighs the caching benefits [S2].

## Background
The concept of hardware caching for tetrahedral strips was proposed to support interactive rendering of tetrahedral volume data [S1, unofficial]. The specific optimization of caching the inverse tetrahedron matrix $M_T^{-1}$ for view transform handling was introduced in the "Projected Tetrahedra Revisited" paper [S2]. These methods build on the foundational Projected Tetrahedra algorithm [Parent](7484-gpu-based-projected-tetrahedra.md).

## Sources
[S1] (PDF) An Architecture For Interactive Tetrahedral Volume ..., https://www.researchgate.net/publication/221257851_An_Architecture_For_Interactive_Tetrahedral_Volume_Rendering, fetched 2026-11-01 (unofficial)
[S2] PDF Projected Tetrahedra Revisited: A Barycentric Formulation Applied to ..., https://www.cs.jhu.edu/~rht//RHT%20Papers/2006/Projected%20Tetrahedra%20Revisited.pdf, fetched 2026-10-27
