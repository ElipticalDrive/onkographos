# Ray-Tetrahedron Intersection


## Overview
Ray-tetrahedron intersection determines the entry and exit points of a ray as it traverses a tetrahedral cell, which is the fundamental geometric primitive for rendering unstructured volumetric data. It solves the problem of efficiently computing the intersection segment $[t_{enter}, t_{exit}]$ required for volume integration, distinguishing itself from sibling methods by handling the specific topology of four triangular faces and the potential for non-convex or deformed geometries.

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model [G].
**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$ [G]. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model [G].
**Parallel Execution**
Computations are mapped to parallel threads [G]. Each thread typically processes one or more pixels or samples [G]. Memory access patterns are optimized for coalescing [G]. Synchronization is used to manage data dependencies between threads [G].

## Subtopics

### [Fast Convex Polygon Wrapper (FCPW) Accelerator](1664-fast-convex-polygon-wrapper-fcpw-accelerator.md)
The Fast Convex Polygon Wrapper (FCPW) Accelerator is a geometric query structure designed to efficiently compute ray-tetrahedron edge intersections and closest-point projections on polygonal meshes. It solves the problem of robustly determining intersection parameters and surface normals for tetrahedral cells defined by polygonal boundaries, serving as the core geometric engine for the [MeshQueryHandler](1664-fast-convex-polygon-wrapper-fcpw-accelerator.md). It sets itself apart from sibling methods by wrapping convex polygonal geometry into an accelerated structure that supports both single-edge intersection finding and closest-point queries, enabling efficient rasterization of ambient occlusion volumes via the [Convex Polygon Falloff Function](10023-convex-polygon-falloff-function.md).

### [Plücker-based Tetrahedral Cell Intersection](1767-pl-cker-based-tetrahedral-cell-intersection.md)
This method determines the entry and exit points of a ray intersecting a tetrahedron by representing the ray and tetrahedron edges as Plücker coordinates, computing permuted inner products to classify faces, and deriving scaled barycentric coordinates directly from these products. It sets itself apart from plane-intersection approaches by avoiding explicit plane calculations and divisions until the final normalization step, and from ray-triangle decomposition methods (like Moller-Trumbore) by leveraging shared edge computations and early termination logic specific to the tetrahedron's topology.

### [Point-in-Tetrahedron Test via Barycentric Coordinates](2591-point-in-tetrahedron-test-via-barycentric-coordinates.md)
This method determines whether a set of query points lies inside a set of tetrahedra by solving for barycentric coordinates. It serves as the core geometric primitive for [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md), allowing volume rendering engines to identify which cells contain sample points for density accumulation. Unlike [Plücker-based Tetrahedral Cell Intersection](1767-pl-cker-based-tetrahedral-cell-intersection.md) which operates on ray primitives, this method operates on explicit point positions, making it suitable for post-intersection validation or direct point-in-volume queries.

### [Tetrahedron Face Extraction](3742-tetrahedron-face-extraction.md)
Computes the four planar faces (unit normals and distances to the origin) from a four-vertex description of a tetrahedron. It serves as the geometric preprocessing step for [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md), converting vertex-based topology into plane-based primitives suitable for analytic ray intersection tests. Unlike sibling methods that optimize traversal or use Plücker coordinates, this method focuses strictly on the algebraic derivation of face planes from vertex coordinates.

### [Skeleton Parameter Minimization for Ray Intersection](3975-skeleton-parameter-minimization-for-ray-intersection.md)
This method computes the intersection of a ray with a Scale-Invariant Integral Surface (SCALIS) primitive by minimizing the squared distance from the ray to the skeleton curve, normalized by the local radius, rather than solving the implicit surface equation directly. It sets itself apart from direct sphere-cone intersection approaches by reducing the problem to finding the root of a linear equation, offering superior numerical stability and lower computational cost.

### [Deformed element ray intersection](5148-deformed-element-ray-intersection.md)
This method determines the ray segments that intersect deformed finite elements, addressing the problem where the mapping from a standard element shape (e.g., a regular tetrahedron) to physical simulation space is non-linear. It is distinct from linear tetrahedron intersection methods because it handles higher-order polynomial mappings, requiring the calculation of multiple intersection segments and an approximate inverse mapping to evaluate scalar functions.

### [Mesh Shader Tetrahedron Intersection Optimization](5174-mesh-shader-tetrahedron-intersection-optimization.md)
This method optimizes the ray-tetrahedron intersection test by leveraging mesh shader wave intrinsics to share geometric data among the four vertices of a single tetrahedron. It reduces redundant memory loads by a factor of four and lowers the computational cost of the intersection routine to approximately 12 floating-point operations (FLOPs). Unlike naive implementations that load tetrahedron data independently for each vertex, this approach precomputes plane parameters in the mesh shader and interpolates them in the fragment shader, building on the standard Cyrus-Beck algorithm.

### [Raytracing Cell-Organized Irregular Volumes](5257-raytracing-cell-organized-irregular-volumes.md)
This method renders volumetric data defined by unstructured or irregular cell grids (such as tetrahedra or hexahedra) by traversing the cell structure along a ray. It solves the problem of efficiently visiting cells intersected by a ray in non-uniform geometries where regular grid acceleration structures (like uniform grids or octrees) are not applicable or efficient. It differs from sibling methods by explicitly handling the topological connectivity of irregular cells and managing concave dataset boundaries, which can cause rays to exit and re-enter the volume multiple times.

### Subgrid-style Intersection Query
- **Definition**: An interface method that computes edge intersection t-values and normals for a tetrahedron in a subgrid context [S1, unofficial].
- **Parameters**: `useRobust` (bool, default false) enables robust intersection computation; `recordNormals` (bool, default true) skips normal calculation when false to save computation [S1, unofficial].
- **Layout**: Input is `std::array<Vector3,4> tet_positions` (or `tet_indices` + `positions`). Output is `std::array<std::vector<double>,6> edge_isect_ts` and `std::array<std::vector<Vector3>,6> edge_isect_normals`, where each array has size 6 corresponding to the tetrahedron edges [S1, unofficial].
- **Code**: `virtual void query_intersections(const std::array<Vector3,4>& tet_positions, std::array<std::vector<double>,6>& edge_isect_ts, std::array<std::vector<Vector3>,6>& edge_isect_normals, bool useRobust = false, bool recordNormals = true) = 0;` [S1, unofficial].

### Barycentric Region Ray-Tetrahedron Intersection
- **Definition**: A method that uses the geometric property that if a ray misses the first face ($F_3$), it intersects the supporting plane in one of six regions defined by edge lines, determining potential entry/exit faces based on barycentric coordinates [S2, unofficial].
- **Steps**: If the ray does not intersect $F_3$ ($V_0V_1V_2$), determine which of the six regions (A–F) on the supporting plane the intersection point lies in, based on barycentric coordinates [S2, unofficial]. Determine if the ray 'enters' the plane using the test: $\sum_{j=0}^{2} (\boldsymbol{\pi}_r \odot \boldsymbol{\pi}_{3j}) > 0$, which is equivalent to $\mathbf{L} \cdot \mathbf{N} < 0$ where $\mathbf{N}$ is the face normal [S2, unofficial]. Based on the region and ray direction, restrict potential entry/exit faces; e.g., entering in region (E) allows entry only through $F_1$ ($V_2V_3V_0$), entering in region (A) allows exit only through $F_0$ ($V_3V_2V_1$) [S2, unofficial].
- **Performance**: The modifications have a negative impact on performance compared to the optimized Plücker-based algorithm because it requires computing all three products $\boldsymbol{\pi}_r \odot \boldsymbol{\pi}_{3j}$ and involves an involved examination of barycentric coordinates [S2, unofficial].

### Polyhedron-Tetrahedron Clipping
- **Definition**: Clips a polyhedron against four clip planes, effectively finding its intersection with a tetrahedron defined by those planes [S3, unofficial].
- **Parameters**: `poly` is the polyhedron to be clipped, where distances to the clip plane and bit flags in `poly.verts[...].orient` must be set prior to calling [S3, unofficial]. `andcmp` is a set of bit flags; face `f` will be skipped if `andcmp & (1 << f)` evaluates to true [S3, unofficial].

### Point-in-Tetrahedron Test
- **Definition**: A geometric test to determine if a point $P$ lies in the interior of a tetrahedron $ABCD$ by evaluating the signs of determinants of sub-vectors formed by the tetrahedron's vertices and $P$ [S4, unofficial].
- **Steps**: Calculate the signed volume of the tetrahedron using one sixth of the determinant of the three vectors spanning it [S4, unofficial]. Compute $D_a, D_b, D_c,$ and $D_d$ as determinants of matrices formed by the other three vertices for each respective vertex (e.g., $D_a$ uses $b, c, d$) [S4, unofficial].
- **Formula**: $D_a = \det(V_a)$ where $V_a = \{b, c, d\}$; $D_b = \det(V_b)$ where $V_b = \{a, c, d\}$; $D_c = \det(V_c)$ where $V_c = \{a, b, d\}$; $D_d = \det(V_d)$ where $V_d = \{a, b, c\}$ [S4, unofficial].

### Ray-casting Interior/Exterior Classification for Tetrahedral Meshes
- **Definition**: A method to classify tetrahedra in a volume mesh as interior or exterior to a closed surface by casting a ray from each tetrahedron's centroid and counting intersections with the surface mesh [S5, unofficial].
- **Steps**: For each tetrahedron in the mesh, compute its centroid [S5, unofficial]. Cast a ray from the centroid in the specified direction against all triangles of the closed surface mesh [S5, unofficial]. Count only forward intersections where the intersection parameter $t > 0$ [S5, unofficial]. Apply the parity test: if the total intersection count is odd, the tetrahedron is interior; if even, it is exterior [S5, unofficial]. Remove all tetrahedra classified as exterior (even hit count) from the mesh [S5, unofficial]. Rebuild adjacency structures for the remaining tetrahedra to maintain mesh consistency [S5, unofficial].
- **Parameters**: Ray direction is a fixed vector $[1.0, 10^{-4}, 10^{-8}]$ used for all rays to minimize edge/vertex grazing ambiguities [S5, unofficial]. Intersection tolerance ($\epsilon$) is $10^{-12}$, used for numerical stability in the Möller–Trumbore algorithm [S5, unofficial].
- **Accuracy**: The computed volume of the interior mesh matches analytical values within 5% for a unit box and a unit sphere [S5, unofficial].

## Background
The problem of ray-tetrahedron intersection is central to the visualization of unstructured grid data, such as that produced by finite element analysis (FEA) or computational fluid dynamics (CFD). Early methods relied on decomposing the tetrahedron into four triangles and applying ray-triangle intersection tests, such as the Möller–Trumbore algorithm, which is computationally redundant due to shared edges.

The use of Plücker coordinates was introduced to optimize this process by representing lines in 6D space, allowing for the computation of face orientations and intersection parameters without explicit plane equations or divisions until the final step [S2, unofficial]. This approach leverages the topological connectivity of the tetrahedron to enable early termination and shared edge computations.

In the context of GPU rendering, mesh shaders have been utilized to optimize the intersection test by sharing geometric data among the four vertices of a single tetrahedron within a wave group, significantly reducing memory bandwidth requirements [S1, unofficial].

For deformed elements, where the mapping from a reference element to physical space is non-linear, the intersection problem becomes more complex, requiring the solution of polynomial equations and the use of approximate inverse mappings to evaluate scalar fields [S1, unofficial].

Additionally, ray-tetrahedron intersection is used in mesh processing tasks, such as classifying tetrahedra as interior or exterior to a surface by ray-casting from centroids, which is a critical step in generating valid volumetric meshes for simulation [S5, unofficial].

## Sources
[S1] hbaktash/subgrid-marching, https://github.com/hbaktash/subgrid-marching, fetched 2026-10-27 (unofficial)
[S2] Fast Ray–Tetrahedron Intersection using Plücker ..., https://users.uop.gr/~nplatis%20/files/PlatisTheoharisRayTetra.pdf, fetched 2026-10-29 (unofficial)
[S3] devonmpowell/r3d, https://github.com/devonmpowell/r3d, fetched 2026-10-27 (unofficial)
[S4] Robust Voxelization and Visualization by Improved Tetrahedral Mesh Generation, https://doi.org/10.48550/arxiv.2106.01326, fetched 2026-10-26 (unofficial)
[S5] Implement interior/exterior classification for tetrahedral meshes, https://github.com/mkofler96/KoFEM/pull/51, fetched 2026-10-29 (unofficial)
