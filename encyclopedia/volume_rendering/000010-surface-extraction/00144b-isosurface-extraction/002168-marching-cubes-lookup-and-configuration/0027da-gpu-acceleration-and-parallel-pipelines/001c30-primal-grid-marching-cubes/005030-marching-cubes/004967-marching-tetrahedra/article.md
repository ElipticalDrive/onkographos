# Marching Tetrahedra

Also called: Marching tetras, Tetrahedral Marching, Tetrahedral Marching Cubes, marching tets

## Overview
Marching Tetrahedra (MT) is an isosurface extraction algorithm that decomposes each cubic voxel into tetrahedra to resolve the topological ambiguities inherent in [Marching Cubes](774-marching-cubes.md) [S1, unofficial][S3][S17, unofficial]. By processing tetrahedra independently, it guarantees a manifold mesh without cracks or ambiguous configurations, producing higher-quality surfaces at the cost of increased triangle count and computational overhead [S2][S5, unofficial][S6].

## How it works
**Inputs:** A 3D scalar field $V$ defined on a regular grid of dimensions $N_x \times N_y \times N_z$, and an isovalue $I$.
**Outputs:** A triangle mesh representing the isosurface $V(x,y,z) = I$.

### Data Structures
1.  **Scalar Field:** A 1D array or 3D array storing scalar values. Indexing for grid coordinates $(x,y,z)$ is typically $idx = x + N_x(y + N_z z)$ [S14, unofficial].
2.  **Vertex Lookup Cache:** A dictionary or hash map mapping unique edge keys (defined by the global coordinates of the two endpoints) to the generated vertex index. This ensures shared edges between adjacent tetrahedra produce identical intersection points, preventing holes [S8, S17, unofficial].
3.  **Tetrahedron Decomposition Table:** A static array defining the 4 vertex indices (0-7) for each tetrahedron within a voxel. The standard 6-tetrahedron decomposition uses the following vertex indices (where 0 is the origin of the voxel):
    *   $T_0: \{0, 7, 3, 2\}$
    *   $T_1: \{0, 7, 2, 6\}$
    *   $T_2: \{0, 4, 6, 7\}$
    *   $T_3: \{0, 6, 1, 2\}$
    *   $T_4: \{0, 6, 1, 4\}$
    *   $T_5: \{5, 6, 1, 4\}$
    [S17, unofficial]
    *Note: Other decompositions exist, such as the 5-tetrahedron scheme [S1, S5, unofficial], but the 6-tet scheme is preferred for shared faces between neighboring voxels [S5, unofficial].*
4.  **Edge-Vertex Mapping:** A static array `tetrahedronEdgeConnection[6][2]` mapping the 6 edges of a tetrahedron to its 4 vertices. Standard ordering:
    *   Edge 0: $\{0, 1\}$, Edge 1: $\{1, 2\}$, Edge 2: $\{2, 0\}$
    *   Edge 3: $\{0, 3\}$, Edge 4: $\{1, 3\}$, Edge 5: $\{2, 3\}$
    [S7, S20, unofficial]
5.  **Triangle Lookup Table:** A static array `tetrahedronTriangles[16][7]` mapping the 16 possible vertex sign configurations (4-bit index) to the resulting triangles. Each entry contains up to two triangles (3 edge indices each), terminated by -1.
    *   Example for Flag 1 (Vertex 0 inside): `{0, 3, 2, -1, -1, -1, -1}` [S20, unofficial].
    *   The table is symmetric; configurations $k$ and $15-k$ produce the same geometry with inverted normals [S20, unofficial].

### Algorithm Steps

1.  **Initialize:** Create empty vertex list, index list, and edge cache.
2.  **Iterate Voxels:** Loop through all voxels $(x, y, z)$ in the grid.
3.  **Sample Vertices:** Retrieve scalar values $v_0 \dots v_7$ and positions $p_0 \dots p_7$ for the 8 corners of the current voxel.
4.  **Decompose:** For each of the 6 tetrahedra in the voxel:
    a.  **Get Vertices:** Retrieve the 4 vertex indices $(i_0, i_1, i_2, i_3)$ from the decomposition table.
    b.  **Compute Case Index:** Determine the 4-bit configuration mask `flagIndex`:
        ```python
        flagIndex = 0
        if v[i_0] < I: flagIndex |= 1
        if v[i_1] < I: flagIndex |= 2
        if v[i_2] < I: flagIndex |= 4
        if v[i_3] < I: flagIndex |= 8
        ```
        [S10, S17, unofficial]
    c.  **Lookup Triangles:** Retrieve the triangle definitions from `tetrahedronTriangles[flagIndex]`. If the first entry is -1, continue to the next tetrahedron.
    d.  **Generate Vertices:** For each triangle in the lookup result (up to 2):
        i.  Get the 3 edge indices $(e_a, e_b, e_c)$ from the lookup table.
        ii. For each edge index, determine the two global vertex indices $(u, v)$ using `tetrahedronEdgeConnection`.
        iii. Calculate the intersection point on the edge. If the edge has not been cached, compute it and store it in the cache using the key $(\min(u,v), \max(u,v))$.
        iv. **Interpolation:** Compute the intersection point $P$ on the edge between vertices $A$ and $B$ with values $f_A$ and $f_B$:
            $$ t = \frac{I - f_A}{f_B - f_A} $$
            $$ P = A + t(B - A) $$
            [S9, S17, unofficial]
            *Guard:* If $|f_B - f_A| < \epsilon$ (e.g., $10^{-6}$), return the midpoint $(A+B)/2$ to avoid division by zero [S7, S14, unofficial].
        v.  Add the vertex to the mesh if new, or retrieve its index from the cache.
    e.  **Add Triangle:** Add the 3 vertex indices to the mesh index list. Ensure consistent winding order (e.g., using a global `WindingOrder` array) [S7, S20, unofficial].

### Parameters
*   **Isovalue ($I$):** The threshold scalar value. Default 0.0 [S7, S8, unofficial].
*   **Decomposition Scheme:** 6 tetrahedra per voxel is standard for continuity [S5, S17, unofficial]. 5 tetrahedra is an alternative but requires alternating schemes to maintain continuity [S1, S19, unofficial].
*   **Epsilon ($\epsilon$):** Tolerance for degenerate cases (value == isovalue or equal edge values). Default $10^{-3}$ or $10^{-6}$ [S8, S14, unofficial].

### Edge Cases and Pitfalls
*   **Ambiguity Resolution:** Unlike Marching Cubes, MT has no topological ambiguities because a tetrahedron has only 4 vertices, reducing the case logic to 16 symmetric configurations [S13, S17, unofficial].
*   **Mesh Holes:** Without caching edge intersections, floating-point errors can cause gaps where tetrahedra share edges. Caching is mandatory for a manifold mesh [S17, unofficial].
*   **Degenerate Triangles:** If a vertex lies exactly on the isovalue, the interpolation parameter $t$ may be 0 or 1, resulting in duplicate vertices. These should be filtered or handled by the epsilon guard [S9, S12, unofficial].
*   **Winding Order:** Incorrect vertex ordering in the decomposition table or lookup table leads to inverted normals. The winding order must be consistent across all tetrahedra [S10, S20, unofficial].
*   **Performance:** MT generates more triangles than Marching Cubes (approx. 2-3x) and requires more memory for the vertex cache [S5, S14, unofficial].

## Subtopics

### Tetrahedralization-based Mesh Extraction
A variant that decouples mesh resolution from Gaussian distribution, enabling fine meshing of restricted scene segments. It performs tetrahedralization of vertices, classifies tets as inside/outside based on sampled interior points, and extracts separating faces. It prioritizes smoothness and artifact reduction over raw metric gains, producing topologically clean surfaces for complex scenes like those in MipNeRF 360 [S21].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels. Each voxel is processed independently, making it highly parallelizable [S4, unofficial].
*   **Memory Complexity:** $O(N)$ for the scalar field. Additional memory is required for the output mesh, which is larger than Marching Cubes. The edge cache adds overhead proportional to the number of unique edges [S5, S17, unofficial].
*   **Performance:** Generally slower than Marching Cubes due to the higher number of tetrahedra (6 vs 1) and the cost of edge caching. Benchmarks show MT can take 3-4x longer than MC for similar resolutions [S4, S5, unofficial].
*   **Strengths:** Topologically correct, no cracks, works on irregular tetrahedral meshes [S2][S17, unofficial].
*   **Limitations:** Higher triangle count, more complex implementation due to caching, potential for degenerate geometry if not handled [S5, S12, unofficial].
*   **When to Use:** When topological correctness is critical (e.g., medical imaging, simulation) and cracks are unacceptable. When working with unstructured tetrahedral grids [S2].
*   **When Not to Use:** When performance is the primary constraint and approximate surfaces are acceptable.

## Background
Marching Tetrahedra was developed as an extension to Marching Cubes to circumvent the Marching Cubes patent (US 4,710,876) and to solve the topological ambiguity problem present in some Marching Cubes configurations [S3, S6]. The patent expired in 2005 [S3]. The method was popularized by researchers such as Doi and Koide, who also applied it natively to tetrahedral meshes [S18, unofficial]. It is widely used in scientific visualization and medical imaging where mesh quality and topology are paramount [S2][S5, unofficial].

## Sources
[S1] An Overview of Rendering from Volume Data --- including Surface and Volume Rendering, https://core.ac.uk/download/63614.pdf, fetched 2026-10-30 (unofficial)
[S2] Adaptive Multi-Valued Volume Data Visualization using Data-Dependent Error Metrics, https://doi.org/10.2316/journal.205.2005.2.205-4291, fetched 2026-10-26
[S3] Marching Cubes algorithm, https://en.wikipedia.org/wiki/Marching_cubes, fetched 2026-10-26
[S4] How to speed up marching cubes?, https://stackoverflow.com/questions/868831/how-to-speed-up-marching-cubes, fetched 2026-10-26 (unofficial)
[S5] Two Methods for Iso-Surface Extraction from Volumetric Data and Their Comparison, https://arxiv.org/abs/2201.03446v1, fetched 2026-10-26 (unofficial)
[S6] Isosurface, https://en.wikipedia.org/wiki/Isosurface, fetched 2026-10-26
[S7] Scrawk/Marching-Cubes, https://github.com/Scrawk/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S8] JuliaGeometry/Meshing.jl, https://github.com/JuliaGeometry/Meshing.jl, fetched 2026-10-27 (unofficial)
[S9] andresbejarano/MarchingTetrahedra, https://github.com/andresbejarano/MarchingTetrahedra, fetched 2026-10-27 (unofficial)
[S10] d3x0r/MarchingTetrahedra, https://github.com/d3x0r/MarchingTetrahedra, fetched 2026-10-27 (unofficial)
[S11] AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, https://github.com/AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, fetched 2026-10-27 (unofficial)
[S12] dborah123/marching-tetrahedra, https://github.com/dborah123/marching-tetrahedra, fetched 2026-10-27 (unofficial)
[S13] mwrazam/marchingtets-reconstruction, https://github.com/mwrazam/marchingtets-reconstruction, fetched 2026-10-27 (unofficial)
[S14] Torbenhjl/Marching-Tetrahedra-on-CT-headscan, https://github.com/Torbenhjl/Marching-Tetrahedra-on-CT-headscan, fetched 2026-10-27 (unofficial)
[S15] RobertBoettcherSF/Ada-Marching-Tetrahedrons, https://github.com/RobertBoettcherSF/Ada-Marching-Tetrahedrons, fetched 2026-10-27 (unofficial)
[S16] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
[S17] 2453852330/VoxelHelper, https://github.com/2453852330/VoxelHelper, fetched 2026-10-27 (unofficial)
[S18] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S19] A Comparison of Fundamental Methods for Iso-surface Extraction, https://doi.org/10.48550/arxiv.2301.01715, fetched 2026-10-29 (unofficial)
[S20] Marching Cubes algorithm, https://github.com/thejasonfisher/marchingCubes/blob/master/MarchingCubes.cs, fetched 2026-10-30 (unofficial)
[S21] From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians, https://doi.org/10.1007/978-3-032-37095-2_4, fetched 2026-10-29
