# Marching Tetrahedrons Algorithm
Also called: MTA

## Overview
The Marching Tetrahedrons Algorithm (MTA) is an isosurface extraction method that decomposes volumetric data cells into tetrahedra to resolve topological ambiguities inherent in cube-based methods like Marching Cubes [S1, unofficial]. Unlike its siblings, MTA guarantees topological consistency by ensuring that the intersection of an isosurface with a tetrahedron is always a simple polygon, eliminating the need for complex case tables required by cubic decompositions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field defined on a regular grid of voxels, where each voxel is a cube with scalar values at its 8 vertices.
*   **Output:** A triangle soup representing the isosurface at a specified threshold value $T$.

**Data Structures:**
*   **Grid:** A 3D array of scalar values $V(x,y,z)$.
*   **Tetrahedron:** Defined by 4 vertices $(v_0, v_1, v_2, v_3)$ and their corresponding scalar values $(s_0, s_1, s_2, s_3)$.
*   **Lookup Table:** A mapping from the binary configuration of vertices (inside/outside the isosurface) to the set of edges intersected by the isosurface.

**Decomposition:**
Each cube cell is subdivided into 6 tetrahedra. The decomposition is not symmetric, which introduces directional ambiguities [S1, unofficial]. A common decomposition scheme splits the cube along its main diagonal and then subdivides the resulting prisms. For a cube with vertices indexed $0$ to $7$, the 6 tetrahedra are typically formed by connecting the vertices in a specific sequence (e.g., $0-1-2-5$, $0-2-3-5$, $0-3-7-5$, $0-7-6-5$, $0-6-4-5$, $0-4-1-5$), though the exact connectivity depends on the chosen diagonal [S1, S3, unofficial].

**Algorithm Steps:**
1.  **Iterate over Cells:** For each voxel in the grid:
2.  **Decompose:** Split the voxel into 6 tetrahedra.
3.  **Classify Vertices:** For each tetrahedron, determine the state of each vertex relative to the isosurface threshold $T$. If $s_i < T$, the vertex is "inside" (bit 1); otherwise, it is "outside" (bit 0). This creates an 8-bit index for the cube, but since each tetrahedron only has 4 vertices, the relevant configuration is a 4-bit index.
4.  **Lookup Intersections:** Use the 4-bit index to look up which edges of the tetrahedron are intersected by the isosurface. There are 16 possible configurations for a tetrahedron.
    *   If the index is 0 (all outside) or 15 (all inside), no intersection occurs.
    *   Otherwise, the isosurface intersects the tetrahedron.
5.  **Interpolate Points:** For each intersected edge, calculate the exact intersection point $P$ using linear interpolation between the two endpoints $A$ and $B$ with values $s_A$ and $s_B$:
    $$ P = A + \frac{T - s_A}{s_B - s_A} (B - A) $$
    [S1, unofficial][S2]
6.  **Generate Triangles:** Connect the intersection points to form triangles. The connectivity of the triangles is determined by the lookup table entry for the specific 4-bit configuration.
7.  **Handle Ambiguities:** Because the cube-to-tetrahedron decomposition is asymmetric, adjacent cubes may produce conflicting triangles if the decomposition direction is not consistent across the grid. This can lead to holes or artifacts [S1, unofficial]. Consistent decomposition direction (e.g., always splitting along the same main diagonal) is required to minimize, but not entirely eliminate, these ambiguities.

**Parameters:**
*   **Threshold $T$:** The scalar value defining the isosurface. Typical values depend on the data range.
*   **Decomposition Scheme:** The specific way cubes are split into tetrahedra.

**Edge Cases and Pitfalls:**
*   **Directional Ambiguities:** The MTA suffers from directional ambiguities because the cube cannot be subdivided into tetrahedra symmetrically [S1, unofficial]. This can result in topological inconsistencies or holes in the extracted surface if not handled carefully.
*   **Precision:** Linear interpolation assumes the isosurface is locally planar within the tetrahedron. For highly curved surfaces, this approximation may introduce geometric error.
*   **Boundary Conditions:** Vertices on the grid boundary must be handled correctly to avoid generating triangles outside the domain.

## Subtopics

### [Regularised Marching Tetrahedra](1301-regularised-marching-tetrahedra.md)
Regularised Marching Tetrahedra (RMT) combines MTA with vertex clustering to simplify the resulting mesh [1301-regularised-marching-tetrahedra.md]. It reduces triangle count by ~70% while preserving topology by clustering vertices based on the tetrahedral lattice. Choose when mesh simplification and improved triangle aspect ratios are needed.

### [Error Analysis via Marching Tetrahedra](1308-error-analysis-via-marching-tetrahedra.md)
This method uses MTA to evaluate geometric error for hierarchical volume representations [1308-error-analysis-via-marching-tetrahedra.md]. It measures displacement of isosurface points to determine if child tetrahedra can be fused. Choose when a viewpoint-independent error bound is required for multiresolution rendering.

### [Subgrid Pipeline](1640-subgrid-pipeline.md)
The Subgrid Pipeline resolves topological ambiguities in MTA by processing individual tetrahedra to generate boundary curves [1640-subgrid-pipeline.md]. It ensures topological correctness where standard MTA fails. Choose when watertight meshes are required from tetrahedral grids.

### [McGrids](2289-mcgrids.md)
McGrids uses MTA to extract isosurfaces from adaptive, non-uniform tetrahedral grids generated via Monte Carlo sampling [2289-mcgrids.md]. It concentrates grid points near surfaces, reducing memory usage compared to uniform grids. Choose when efficient sampling of complex implicit fields is needed.

### [BCC Marching Tetrahedra with Permutation-Based Lookup](3414-bcc-marching-tetrahedra-with-permutation-based-lookup.md)
This method extracts isosurfaces from Body-Centered Cubic (BCC) datasets using a single reference tetrahedron and 3D permutations [3414-bcc-marching-tetrahedra-with-permutation-based-lookup.md]. It reduces lookup table size and improves mesh connectivity. Choose when memory efficiency and connectivity are priorities for BCC data.

### [Exact Fiber Surface Extraction](3445-exact-fiber-surface-extraction.md)
This method computes pre-images of 1-manifold paths in bivariate fields over tetrahedral meshes [3445-exact-fiber-surface-extraction.md]. It guarantees planar pre-images within each tetrahedron. Choose when sharp bends and complex topologies in fiber surfaces must be captured exactly.

### [Marching Tetrahedra Reconstruction](3516-marching-tetrahedra-reconstruction.md)
This method reconstructs 3D meshes from vertices and normals by sampling a Welland-based implicit field on a tetrahedral grid and extracting it with MTA [3516-marching-tetrahedra-reconstruction.md]. It eliminates cube-based ambiguities. Choose when reconstructing surfaces from scattered point clouds with normals.

### [Marching Tetrahedra 3 (DLC-Plane)](3632-marching-tetrahedra-3-dlc-plane.md)
This method uses a diamond crystal lattice structure to decompose grids into cells with 5 tetrahedra, eliminating fabricated center points [3632-marching-tetrahedra-3-dlc-plane.md]. It reduces boundary creases. Choose when working with voxel or point cloud data and minimizing boundary artifacts is critical.

### [Raster Algorithm for Fiber Surface Extraction](4960-raster-algorithm-for-fiber-surface-extraction.md)
This algorithm approximates fiber surfaces by rasterizing a signed range distance field to the Fiber Surface Control Polygon [4960-raster-algorithm-for-fiber-surface-extraction.md]. It trades geometric accuracy for speed. Choose when real-time performance is more important than exact geometric fidelity.

### Tetrahedral Normal Calculation
A method to calculate smooth isosurface normals for tetrahedral meshes by computing vertex normals weighted by the volume of connected tetrahedra [S2].
*   **Definition:** Computes vertex normals $N_v$ by summing tetrahedron normals $N_t$ weighted by their volume $V(h_t)$ [S2].
*   **Steps:**
    1.  Calculate the constant plane normal $N_t = (a, b, c)^T$ for each tetrahedron [S2].
    2.  Pre-compute and store vertex normals in the data file [S2].
    3.  Calculate 2D barycentric coordinates of the intersection point, extended to 3D [S2].
    4.  Weight the pre-computed vertex normals with the barycentric coordinates and normalize the result [S2].
*   **Formula:** The vertex normal $N_v$ is computed as:
    $$ N_v = \sum_{t=0}^{n} N_t V(h_t) $$
    where $n$ is the number of tetrahedra connected to the vertex, $N_t$ is the normal of tetrahedron $t$, and $V(h_t)$ is its volume [S2].
*   **Pitfall:** Using the constant cell normal $N_t$ directly results in severe discontinuities between tetrahedra surfaces [S2].

### Centered Cubic Lattice
A method that splits the cube cell into 24 tetrahedra by introducing a new data value at the center of gravity of the processed cell [S3, unofficial].
*   **Definition:** Introduces a central sample computed as the arithmetic or weighted mean of the cell's vertices [S3, unofficial].
*   **Steps:**
    1.  Compute the central value as the mean of the 8 cube vertices [S3, unofficial].
    2.  Split the cube into 24 tetrahedra connecting the central point to the cube faces and edges [S3, unofficial].
    3.  Process each tetrahedron separately using standard MTA logic [S3, unofficial].
    4.  Perform interpolation on new edges introduced by the decomposition, specifically edges among adjacent central points [S3, unofficial].
*   **Comparison:** All 24 tetrahedra are identical in dimensions, similar to the MT6 method [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of voxels, as each voxel is processed independently.
*   **Memory Complexity:** $O(N)$ for storing the grid and the output triangle list.
*   **Strengths:** Topological consistency within each tetrahedron; no complex case tables like Marching Cubes; easier to handle non-cubic grids.
*   **Limitations:** Directional ambiguities due to asymmetric cube decomposition; potentially higher triangle count than optimized Marching Cubes implementations; interpolation error on curved surfaces.
*   **When to Use:** When topological correctness is critical and the data is naturally tetrahedral or can be easily decomposed; when avoiding the complexity of Marching Cubes case tables is desired.
*   **When Not to Use:** When maximum performance is required and the data is regular cubic (Marching Cubes may be faster); when directional artifacts are unacceptable without additional post-processing.

## Background
The Marching Tetrahedrons Algorithm was developed as an alternative to Marching Cubes to address topological ambiguities [S1, unofficial]. The ambiguities in Marching Cubes arise because a cube can be decomposed into tetrahedra in multiple ways, leading to inconsistent surface connectivity [S1, unofficial]. MTA resolves this by fixing the decomposition, ensuring that the intersection of an isosurface with a tetrahedron is always a simple polygon [S1, unofficial]. The method has been widely adopted in scientific visualization and medical imaging for its robustness [S1, unofficial][S2].

## Sources
[S1] Volume-Enclosing Surface Extraction, https://arxiv.org/abs/1011.1787v4, fetched 2026-10-26 (unofficial)
[S2] Fast Ray Traversal of Tetrahedral and Hexahedral Meshes for Direct Volume Rendering, https://doi.org/10.2312/vissym/eurovis06/235-242, fetched 2026-10-24
[S3] A Comparison of Fundamental Methods for Iso-surface Extraction, https://doi.org/10.48550/arxiv.2301.01715, fetched 2026-10-29 (unofficial)
