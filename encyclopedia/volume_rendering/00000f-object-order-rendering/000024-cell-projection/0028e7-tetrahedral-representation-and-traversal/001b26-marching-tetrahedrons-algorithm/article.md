# Marching Tetrahedrons Algorithm

## Overview
The Marching Tetrahedrons Algorithm (MTA) is an iso-surface extraction method that converts a scalar field defined on a regular grid into a triangular mesh by subdividing each cubic voxel into tetrahedra and processing each tetrahedron independently [G]. It solves the problem of extracting explicit polygonal surfaces from volumetric data [G]. Unlike its sibling [Tetrahedral Volume Rendering](820-tetrahedral-volume-rendering.md), which focuses on direct volume rendering via ray casting, MTA generates an explicit mesh [G]. It differs from [Projected Tetrahedra](877-projected-tetrahedra.md) by constructing a connected triangle mesh rather than projecting volumetric primitives to the screen [G]. The algorithm unfortunately has directional ambiguities because it subdivides a cube with tetrahedrons, which cannot be done symmetrically [S1, unofficial].

## How it works
**Inputs:** A 3D scalar field $f(x,y,z)$ defined on a regular grid of voxels, where each voxel is a cube with 8 corner vertices [G]. An iso-value $I$ defining the surface threshold [G].
**Outputs:** A set of triangles representing the iso-surface [G].

**Data Structures:**
1. **Grid:** A 3D array of scalar values [G].
2. **Tetrahedral Decomposition:** Each cube voxel is decomposed into 6 tetrahedra [G]. The decomposition must be consistent across the entire grid to avoid gaps [G].
3. **Lookup/Edge Mapping:** A mapping of tetrahedron vertex indices to the 6 possible edges, used to determine intersection topology [G].

**Steps:**
1. **Decomposition:** For each voxel in the grid, decompose the cube into 6 tetrahedra. Let the vertices of a tetrahedron be $v_0, v_1, v_2, v_3$ with scalar values $f_0, f_1, f_2, f_3$ [G].
2. **Classification:** For each tetrahedron, determine the binary state of each vertex relative to the iso-value $I$. Let $s_i = 1$ if $f_i \ge I$ and $s_i = 0$ if $f_i < I$ [G]. This forms a 4-bit state index.
3. **Edge Intersection:** Determine which edges of the tetrahedron are intersected by the iso-surface. A tetrahedron has 6 edges. If both endpoints of an edge have different states, the iso-surface intersects that edge [G].
4. **Interpolation:** For each intersected edge, calculate the exact position of the intersection point using linear interpolation. If an edge connects vertices $v_a$ and $v_b$ with values $f_a$ and $f_b$, the intersection point $P$ is:
   $$ P = v_a + \frac{I - f_a}{f_b - f_a} (v_b - v_a) $$
   [G] Handle the case where $f_b = f_a$ by checking if $f_a = I$ (vertex is on surface) or skipping if the edge is entirely inside or outside [G].
5. **Triangle Generation:** Connect the intersection points to form triangles. For a tetrahedron, the iso-surface forms a planar polygon. If 3 edges intersect, they form a triangle. If 4 edges intersect, they form a quadrilateral, which is split into two triangles [G].
6. **Mesh Assembly:** Collect all generated triangles from all tetrahedra. Since the decomposition is consistent, vertices on shared faces/edges are generated at the same location, ensuring a watertight mesh [G].

**Parameters:**
* Iso-value $I$: Depends on the data [G].
* Decomposition scheme: Must be consistent across the grid [G].

**Edge Cases and Pitfalls:**
* **Directional Ambiguities:** Because the cube is subdivided into tetrahedra in a non-symmetric way, the algorithm produces directional ambiguities [S1, unofficial].
* **Degenerate Cases:** If a vertex lies exactly on the iso-surface ($f_i = I$), the classification is ambiguous. Standard practice is to treat $f_i \ge I$ as inside [G].
* **Precision:** Linear interpolation assumes the field is linear within the tetrahedron. For highly non-linear fields, this introduces geometric error [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vec3(x, y, z):
    return (float(x), float(y), float(z))

def lerp(a, b, t):
    return (a[0] + t*(b[0]-a[0]), a[1] + t*(b[1]-a[1]), a[2] + t*(b[2]-a[2]))

def marching_tetrahedrons(grid, iso_value):
    triangles = []
    # Cube vertices: 0=000, 1=100, 2=010, 3=110, 4=001, 5=101, 6=011, 7=111
    # Standard decomposition into 6 tetrahedra
    tetrahedra = [
        (0, 1, 2, 5), (0, 2, 6, 5), (0, 6, 7, 5),
        (0, 4, 7, 5), (0, 3, 7, 5), (0, 1, 3, 5)
    ]
    # Edge indices for a tetrahedron (v0,v1,v2,v3): 0:(0,1), 1:(0,2), 2:(0,3), 3:(1,2), 4:(1,3), 5:(2,3)
    tet_edges = [(0,1), (0,2), (0,3), (1,2), (1,3), (2,3)]

    for x in range(grid.width - 1):
        for y in range(grid.height - 1):
            for z in range(grid.depth - 1):
                verts = [
                    grid.get(x, y, z), grid.get(x+1, y, z),
                    grid.get(x, y+1, z), grid.get(x+1, y+1, z),
                    grid.get(x, y, z+1), grid.get(x+1, y, z+1),
                    grid.get(x, y+1, z+1), grid.get(x+1, y+1, z+1)
                ]
                positions = [
                    vec3(x, y, z), vec3(x+1, y, z),
                    vec3(x, y+1, z), vec3(x+1, y+1, z),
                    vec3(x, y, z+1), vec3(x+1, y, z+1),
                    vec3(x, y+1, z+1), vec3(x+1, y+1, z+1)
                ]

                for tet in tetrahedra:
                    v_idx = list(tet)
                    states = [1 if verts[i] >= iso_value else 0 for i in v_idx]
                    state_index = sum(s << i for i, s in enumerate(states))

                    if state_index == 0 or state_index == 15:
                        continue

                    intersections = {}
                    for e_idx, (va, vb) in enumerate(tet_edges):
                        if states[va] != states[vb]:
                            fa, fb = verts[v_idx[va]], verts[v_idx[vb]]
                            t = (iso_value - fa) / (fb - fa) if fb != fa else 0.0
                            intersections[e_idx] = lerp(positions[v_idx[va]], positions[v_idx[vb]], t)

                    edge_list = sorted(intersections.keys())
                    if len(edge_list) == 3:
                        triangles.append((intersections[edge_list[0]], intersections[edge_list[1]], intersections[edge_list[2]]))
                    elif len(edge_list) == 4:
                        triangles.append((intersections[edge_list[0]], intersections[edge_list[1]], intersections[edge_list[2]]))
                        triangles.append((intersections[edge_list[0]], intersections[edge_list[2]], intersections[edge_list[3]]))

    return triangles
```

## Subtopics

### [Regularised Marching Tetrahedra](1301-regularised-marching-tetrahedra.md)
Regularised Marching Tetrahedra (RMT) is an iso-surface extraction algorithm that combines the topological consistency of the Marching Tetrahedra method with vertex clustering to simplify the resulting mesh. Unlike standard MT, which often produces thin, poorly shaped triangles that degrade interpolated shading, RMT generates surfaces with near-regular triangles and significantly reduced triangle counts (typically 70% fewer than MT) while preserving the original topology. It is chosen when mesh simplification and shading quality are prioritized over raw extraction speed.

### [Error Analysis via Marching Tetrahedra](1308-error-analysis-via-marching-tetrahedra.md)
Error Analysis via Marching Tetrahedra is a geometric error metric used to determine whether two child tetrahedra in a hierarchical volume representation can be fused into a single parent tetrahedron without exceeding a specified approximation tolerance. It solves the problem of balancing mesh simplification against geometric fidelity during isosurface extraction. Unlike sibling methods that may rely on algebraic residuals or view-dependent projections, this method explicitly evaluates the displacement of the isosurface intersection points along specific segments within the tetrahedron using the Marching Tetrahedra topology. It is chosen when a physically meaningful, viewpoint-independent error bound is required for multiresolution isosurface rendering.

### [Subgrid Pipeline](1640-subgrid-pipeline.md)
The Subgrid Pipeline is a method for extracting watertight triangle soups from tetrahedral grids by explicitly resolving topological ambiguities in isosurface extraction. It distinguishes itself from standard Marching Tetrahedrons by processing individual tetrahedra to generate open, scoop, and normal boundary curves, ensuring topological correctness where standard algorithms fail. It is chosen when strict topological correctness is required for complex or noisy scalar fields.

### [McGrids](2289-mcgrids.md)
McGrids is an iso-surface extraction method that constructs adaptive, non-uniform tetrahedral grids using an iterative Monte Carlo process to estimate a probability density function based on the implicit field, followed by a refinement step and mesh extraction via Marching Tetrahedrons. It solves the problem of inefficient uniform grid sampling by concentrating grid points near surfaces and geometric details, significantly reducing the number of implicit field queries and memory usage compared to uniform grids. Unlike sibling methods that may rely on fixed or hierarchical structures, McGrids dynamically adapts the grid topology through probabilistic sampling and Centroidal Voronoi Tessellation relaxation. It is chosen when memory efficiency and adaptive resolution are critical.

### [BCC Marching Tetrahedra with Permutation-Based Lookup](3414-bcc-marching-tetrahedra-with-permutation-based-lookup.md)
A GPU-based isosurface extraction method for Body-Centered Cubic volume datasets that generates an explicit, connected triangle mesh rather than a triangle soup. It solves the memory and performance overhead of storing large lookup tables for all BCC cell types by using a single reference tetrahedron and applying 3D permutation transformations to handle the remaining five cell types. Unlike sibling methods that may rely on large static tables or post-processing to connect triangles, this method builds connectivity directly during extraction, enabling efficient applications like curvature rendering. It is chosen for high-performance GPU rendering of BCC-structured data.

### [Exact Fiber Surface Extraction](3445-exact-fiber-surface-extraction.md)
Exact Fiber Surface Extraction computes the pre-image of a 1-manifold path in the range of a bivariate field over a tetrahedral mesh. It solves the problem of approximating these surfaces using signed distance fields, which fail to capture sharp bends and complex topologies. Unlike Regularised Marching Tetrahedra, which smooths geometry, this method guarantees planar pre-images within each tetrahedron by decomposing the extraction into base surface generation and precise clipping. It is chosen when exact topological preservation of bivariate field fibers is required.

### [Marching Tetrahedra Reconstruction](3516-marching-tetrahedra-reconstruction.md)
Marching Tetrahedra Reconstruction is a surface reconstruction method that converts a set of input vertices and normals into a 3D mesh by computing an implicit field using the Welland function, sampling it on a tetrahedral grid, and extracting the zero-level set via Marching Tetrahedra. It differs from standard Marching Cubes by using a tetrahedral decomposition of the grid cells, which eliminates the ambiguity cases inherent in cube-based extraction. It builds on Marching Tetrahedrons but replaces the standard linear interpolation with a Welland-based implicit field evaluation. It is chosen when reconstructing surfaces from scattered point clouds with normals.

### [Marching Tetrahedra 3 (DLC-Plane)](3632-marching-tetrahedra-3-dlc-plane.md)
Marching Tetrahedra 3 is a volume meshing method that generates isosurfaces from voxel or point cloud data using a diamond crystal lattice structure. It decomposes the grid into cells containing exactly 5 tetrahedra, eliminating the need for fabricated center points. Unlike standard Marching Cubes or earlier Marching Tetrahedra variants that require synthesizing internal vertices, this method uses only the 5 existing grid points per cell, reducing boundary creases and improving results for opening void areas. It operates by slicing the data plane-by-plane, generating a single calculation for all shared points to reduce total point count and computational overhead. It is chosen for voxel data with void regions or when minimizing synthetic vertices is required.

### [Raster Algorithm for Fiber Surface Extraction](4960-raster-algorithm-for-fiber-surface-extraction.md)
Approximates fiber surfaces by extracting the 0 level-set of a signed range distance field to the Fiber Surface Control Polygon, using a 2D range raster to reduce the number of distance field evaluations. It is chosen when performance optimization for fiber surface approximation is prioritized over exact geometric evaluation.

### Tetrahedral Normal Calculation
A method to calculate smooth isosurface normals for tetrahedral meshes by computing vertex normals weighted by the volume of connected tetrahedra [S2]. The plane normal $N_t = (a, b, c)^T$ for each tetrahedron is constant within the cell because the orientation of the plane describing the iso-surface is independent of the iso-value [S2]. These vertex normals can be pre-computed and stored in the data file [S2]. The vertex normal $N_v$ is calculated as the sum of the tetrahedron normals $N_t$ weighted by their volume $V(h_t)$:
$$ N_v = \sum_{t=0}^{n} N_t V(h_t) $$
where $n$ is the number of tetrahedra connected to the vertex [S2]. The resulting vector is normalized [S2]. The 2D barycentric coordinates of the intersection point within the tetrahedron are extended to 3D and used to weight the vertex normals to obtain a smooth isosurface normal [S2]. Using the constant cell normal $N_t$ directly results in severe discontinuities between tetrahedra surfaces, decreasing rendering quality [S2]. It is chosen when smooth shading is required for tetrahedral meshes [S2].

### Centered Cubic Lattice
A method that splits the cube cell into 24 tetrahedra by introducing a new data value at the center of gravity of the processed cell [S3, unofficial]. It introduces a new data value to the center of gravity of the processed cell, computed as the arithmetic mean or weighted mean of the cell's vertices [S3, unofficial]. Each tetrahedron is then processed separately in the same way as in MT5 or MT6 methods [S3, unofficial]. As well as in previous methods this kind of splitting introduces new edges at which the interpolation will be made, specifically edges among adjacent central points [S3, unofficial]. In this division scheme, all the 24 tetrahedra are the same as to the dimensions, similarly to the MT6 method [S3, unofficial]. It is chosen when uniform tetrahedral dimensions and shared central interpolation are required [S3, unofficial].

## Operating characteristics
**Time Complexity:** $O(N)$, where $N$ is the number of voxels, as each voxel is processed independently [G].
**Memory Complexity:** $O(N)$ for the grid and $O(M)$ for the output mesh, where $M$ is the number of triangles [G].
**Performance:** The algorithm is computationally efficient and highly parallelizable [G].
**Strengths:** Topologically unambiguous per tetrahedron, simpler lookup logic than Marching Cubes, and straightforward implementation [G].
**Limitations:** Directional ambiguities due to asymmetric tetrahedral decomposition [S1, unofficial]. The resulting triangles can be poorly shaped if the decomposition is not chosen carefully [G].
**When to use:** When an explicit, watertight mesh is required from a regular grid and performance is prioritized [G].
**When not to use:** When symmetric decomposition is critical to avoid directional bias, or when adaptive resolution is needed without additional grid generation steps [G].

## Background
The Marching Tetrahedrons Algorithm was developed as an improvement over the Marching Cubes algorithm to resolve topological ambiguities [S1, unofficial]. The method was initially overlooked, resulting in the accidental generation of holes in some data sets, but this problem has since been fixed [S1, unofficial]. The MTA has directional ambiguities because it subdivides a cube with tetrahedrons, which cannot be done symmetrically [S1, unofficial]. The MLA and similar techniques are not template based but require further processing to resolve cell ambiguities, while the CMSA uses templates for 2×2×2 voxel neighborhoods [S1, unofficial].

## Sources
[S1] Volume-Enclosing Surface Extraction, https://arxiv.org/abs/1011.1787v4, fetched 2026-10-26 (unofficial)
[S2] Fast Ray Traversal of Tetrahedral and Hexahedral Meshes for Direct Volume Rendering, https://doi.org/10.2312/vissym/eurovis06/235-242, fetched 2026-10-24
[S3] A Comparison of Fundamental Methods for Iso-surface Extraction, https://doi.org/10.48550/arxiv.2301.01715, fetched 2026-10-29 (unofficial)
