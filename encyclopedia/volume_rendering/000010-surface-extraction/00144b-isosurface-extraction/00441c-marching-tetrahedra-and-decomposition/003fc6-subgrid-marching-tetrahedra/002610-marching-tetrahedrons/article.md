# Marching Tetrahedrons

Also called: Case-based Marching Tetrahedra, IsosurfaceTet, Marching Tetrahedra (Case-based), Tetrahedra Marcher, Tetrahedral Marching

## Overview
Marching Tetrahedrons is an isosurface extraction algorithm that decomposes each cubic voxel into tetrahedra to resolve the topological ambiguities inherent in [Marching Cubes](52-marching-cubes.md) [S1, S3, S6, unofficial]. By operating on tetrahedral cells, it guarantees a manifold surface without the need for disambiguation tables [S1, S6, unofficial]. It sets itself apart from its parent [Subgrid Marching Tetrahedra](1737-subgrid-marching-tetrahedra.md) by using a fixed decomposition of the standard cubic grid rather than a subgrid structure [S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume dataset defined on a regular 3D grid (voxels). Each voxel has 8 corner vertices with associated scalar values $F$ and 3D coordinates $P$ [S3, unofficial].
*   **Output:** A polygonal mesh (list of triangles) representing the isosurface at a specified `isovalue`.

**Data Structures:**
*   **Tetrahedron Struct:** Represents a single tetrahedral cell [S1, unofficial].
    *   `X[4]`, `Y[4]`, `Z[4]`: `float` arrays storing the 3D coordinates of the 4 vertices [S1, unofficial].
    *   `F[4]`: `float` array storing the scalar field values at the 4 vertices [S1, unofficial].
*   **Lookup Table (`lup`):** A 2D array `lup[16][6]` mapping the 16 possible vertex sign combinations (case index) to the edges intersected by the isosurface [S1, unofficial].
    *   Each entry contains up to 5 edge indices (0–5, corresponding to the 6 edges of a tetrahedron) [S1, unofficial].
    *   Edge indices are -1 if not intersected [S1, unofficial].
*   **Edge Mapping:** The 6 edges of a tetrahedron with vertices 0, 1, 2, 3 are indexed as:
    *   0: (0, 1)
    *   1: (1, 2)
    *   2: (0, 2)
    *   3: (1, 3)
    *   4: (2, 3)
    *   5: (0, 3)

**Algorithm Steps:**

1.  **Decomposition:** For each voxel in the volume, decompose the cube into 6 tetrahedra [S5, unofficial]. The specific decomposition must be consistent across all voxels to ensure connectivity [G].
2.  **Classification:** For each tetrahedron, determine the case index `icase` (0–15) [S1, unofficial].
    *   For each vertex $i \in \{0, 1, 2, 3\}$, if $F[i] < \text{isovalue}$, the vertex is "inside" (bit 0); otherwise, it is "outside" (bit 1) [S4, unofficial].
    *   `icase` is the integer formed by these 4 bits: $\sum_{i=0}^{3} b_i 2^i$ [G].
3.  **Lookup:** Retrieve the number of segments `nsegments` from `numSegments[icase]` [S1, unofficial].
    *   If `nsegments` is 0, the tetrahedron contributes no triangles [G].
4.  **Edge Intersection:** For each intersected edge $e$ in `lup[icase]`:
    *   Identify the two vertices $v_0, v_1$ connected by edge $e$ [G].
    *   Compute the intersection point $P$ using linear interpolation:
        $$ P = v_0 + t (v_1 - v_0) $$
        $$ t = \frac{\text{isovalue} - F[v_0]}{F[v_1] - F[v_0]} $$
    *   **Pitfall:** If $F[v_1] - F[v_0] \approx 0$, clamp $t$ to 0.5 or handle as a degenerate case to avoid division by zero [S4, S5, unofficial]. Clamp $t$ to $[0, 1]$ to prevent extrapolation outside the edge segment [S4, unofficial].
5.  **Triangle Generation:**
    *   **Case 1 Segment (1 triangle):** Occurs when `icase` has 1 or 3 inside vertices [G].
        *   If 1 inside vertex: The 3 edges connecting the inside vertex to the 3 outside vertices are intersected [S4, unofficial]. Connect the 3 intersection points to form 1 triangle [S4, unofficial].
        *   If 3 inside vertices: The 3 edges connecting the single outside vertex to the 3 inside vertices are intersected [S4, unofficial]. Connect the 3 intersection points to form 1 triangle [S4, unofficial].
    *   **Case 2 Segments (2 triangles):** Occurs when `icase` has 2 inside and 2 outside vertices [S4, unofficial].
        *   The 4 intersected edges form a quadrilateral.
        *   Triangulate the quadrilateral into 2 triangles. The diagonal choice depends on `icase` to maintain manifold consistency:
            *   For `icase` 5, 6, 9, 10, or 12: Connect vertices 1-2-3.
            *   Otherwise: Connect vertices 0-2-3 [S1, unofficial].
6.  **Output:** Add the generated triangles to the output mesh.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_tetrahedrons(volume, isovalue, dimensions):
    """
    volume: 3D array of scalar values [X, Y, Z]
    isovalue: float threshold
    dimensions: tuple (xdim, ydim, zdim)
    Returns: list of triangles, each triangle is a list of 3 points (x,y,z)
    """
    triangles = []
    xdim, ydim, zdim = dimensions
    
    # Edge mapping for a tetrahedron with vertices 0,1,2,3
    edge_vertices = [(0,1), (1,2), (0,2), (1,3), (2,3), (0,3)]
    
    # Lookup table from [S1, unofficial]
    lup = [
        [-1,-1,-1,-1,-1,-1], # 0
        [0, 2, 5, -1, -1, -1], # 1
        [0, 1, 4, -1, -1, -1], # 2
        [0, 1, 2, 4, -1, -1], # 3
        [1, 2, 5, -1, -1, -1], # 4
        [0, 1, 2, 5, -1, -1], # 5
        [0, 2, 3, 4, -1, -1], # 6
        [3, 4, 5, -1, -1, -1], # 7
        [3, 4, 5, -1, -1, -1], # 8
        [0, 2, 3, 4, -1, -1], # 9
        [0, 1, 4, 5, -1, -1], # 10
        [1, 2, 4, -1, -1, -1], # 11
        [1, 2, 3, 5, -1, -1], # 12
        [0, 1, 3, -1, -1, -1], # 13
        [0, 2, 5, -1, -1, -1], # 14
        [-1,-1,-1,-1,-1,-1]  # 15
    ]
    num_segments = [0, 1, 1, 2, 1, 2, 2, 1, 1, 2, 2, 1, 2, 1, 1, 0]
    
    # Tetrahedra definitions for a voxel (consistent decomposition)
    # Indices refer to voxel corners 0-7.
    tetra_defs = [
        [0, 1, 2, 4], [1, 5, 2, 6], [2, 6, 3, 7],
        [4, 5, 6, 7], [1, 4, 5, 7], [0, 1, 4, 7]
    ]
    
    for z in range(zdim - 1):
        for y in range(ydim - 1):
            for x in range(xdim - 1):
                # Gather 8 voxel corners
                corners = []
                for i in range(8):
                    cx = x + (i & 1)
                    cy = y + ((i >> 1) & 1)
                    cz = z + ((i >> 2) & 1)
                    corners.append((cx, cy, cz, volume[cx, cy, cz]))
                
                for tetra in tetra_defs:
                    v_pos = [corners[i][:3] for i in tetra]
                    v_val = [corners[i][3] for i in tetra]
                    
                    # Compute case index
                    icase = 0
                    for i in range(4):
                        if v_val[i] >= isovalue:
                            icase |= (1 << i)
                            
                    nseg = num_segments[icase]
                    if nseg == 0:
                        continue
                    
                    # Compute intersection points
                    pts = []
                    for e_idx in lup[icase]:
                        if e_idx == -1:
                            break
                        v0, v1 = edge_vertices[e_idx]
                        p0, p1 = v_pos[v0], v_pos[v1]
                        f0, f1 = v_val[v0], v_val[v1]
                        
                        denom = f1 - f0
                        if abs(denom) < 1e-6:
                            t = 0.5
                        else:
                            t = (isovalue - f0) / denom
                            t = max(0.0, min(1.0, t))
                        pts.append(p0 + t * (p1 - p0))
                    
                    # Generate triangles
                    if nseg == 1:
                        triangles.append([pts[0], pts[1], pts[2]])
                    else: # nseg == 2
                        triangles.append([pts[0], pts[1], pts[2]])
                        if icase in [5, 6, 9, 10, 12]:
                            triangles.append([pts[1], pts[2], pts[3]])
                        else:
                            triangles.append([pts[0], pts[2], pts[3]])
    return triangles
```

## Subtopics

### [Adaptive Octree Marching Tetrahedra](3897-adaptive-octree-marching-tetrahedra.md)
An isosurface extraction method that uses an adaptive octree to subdivide a 3D volume only where the scalar field contains the target isovalue. At the leaf nodes (1x1x1 cubes), the volume is decomposed into tetrahedra, and the Marching Tetrahedra algorithm is applied to generate the mesh. It solves the problem of inefficient uniform-grid processing by reducing memory usage and computational cost for sparse or complex datasets. Unlike [Naive Isosurface Extraction](4328-naive-isosurface-extraction.md) which processes every voxel, this method explicitly constructs an octree data structure to manage resolution adaptivity before decomposing leaf nodes into tetrahedra for precise surface intersection.

### [Naive Isosurface Extraction](4328-naive-isosurface-extraction.md)
Naive Isosurface Extraction is a surface extraction algorithm that processes each tetrahedron independently to generate a collection of individual, unconnected polygons. It differs from standard Marching Tetrahedra by iterating through all tetrahedra and their edges to find intersections with an isosurface, rather than using the standard 14-case lookup table based on vertex signs.

### [GPU Vertex Shader Isosurface Extraction](6706-gpu-vertex-shader-isosurface-extraction.md)
GPU Vertex Shader Isosurface Extraction is a hardware-accelerated method for tessellating large unstructured tetrahedral grids by storing grid data in textures and performing isosurface extraction in the vertex shading unit. It solves the problem of data redundancy and memory limitations inherent in previous register-based GPU methods, which could not handle grids larger than ~1 million tetrahedra due to register pressure and PCI-Express transfer bottlenecks. Unlike its siblings, it uses indirect indexing via vertex texture lookup (Shader Model 3.0) to avoid sending redundant geometry for shared vertices, enabling the processing of millions of tetrahedra.

### [Tetrahedral Volume Integration](9701-tetrahedral-volume-integration.md)
Tetrahedral Volume Integration is a method for computing the scalar volume enclosed by an isosurface generated via tetrahedral decomposition. It solves the problem of quantifying the material volume within a grid cell by summing the volumes of the tetrahedra parts that lie inside the isosurface. Unlike surface extraction methods that only output geometry, this method provides a volumetric metric, distinguishing it from siblings like [Marching Tetrahedrons](3720-marching-tetrahedrons.md) which focus on triangle generation.

### [Barycentric Mapping for Material Interfaces](9849-barycentric-mapping-for-material-interfaces.md)
Barycentric Mapping for Material Interfaces is an algorithm for reconstructing piecewise two-manifold boundary surfaces between multiple materials in a grid where each cell contains fractional volumetric information for each material. It transforms the reconstruction problem to analyze a dual data set where vertices have associated barycentric coordinate tuples representing material fractions. It sets itself apart from standard isosurface extraction by operating directly on volume-fraction data in unstructured triangular meshes, using the geometric properties of Voronoi cells in barycentric space to define interfaces rather than scalar field thresholds. Unlike SLIC, Youngs, or Gueffier algorithms which produce generally discontinuous interfaces and are limited to 2D or specific normal estimates, this algorithm produces a crack-free piecewise two-manifold, handles any number of materials per element, works on any grid structure (via tetrahedral conversion), and does not rely on application-specific hydrodynamics knowledge.

### [3-phase Marching Tetrahedra](11053-3-phase-marching-tetrahedra.md)
An algorithm to extract isosurfaces from tetrahedral meshes on GPU that avoids sparse output arrays by using a prefix sum to determine output offsets, allowing separate handling of cells generating 1 vs 2 triangles. It builds on [Marching Tetrahedrons](3720-marching-tetrahedrons.md) by optimizing the memory layout and execution divergence of the triangle generation phase.

### Marching Tetrahedra (Second Algorithm)
An iso-surface extraction method that decomposes a voxel cube using a center cubic lattice (involving more tetrahedra per cube than the first method), resulting in more triangles, higher execution time, but better visual quality for complex models [S10, unofficial]. When averaging values of a tetrahedron with one vertex inside and one outside along a long edge, positions outside the cube have zero value and inside have threshold value, causing artifacts at edges for medium cubes [S10, unofficial]. It generates more triangles than the First Algorithm because each cube participates in a double amount of tetrahedra, leading to higher memory and execution time [S10, unofficial]. It produces higher radius error from vertices and worse area/volume approximation than the First Algorithm, but provides better visual impression for complex models due to human eye sensitivity to edges and gradient changes [S10, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of voxels. Each voxel is decomposed into a constant number of tetrahedra (typically 6), and each tetrahedron is processed in constant time.
*   **Memory Complexity:** $O(N)$ for the output mesh, as the number of triangles is proportional to the number of voxels.
*   **Performance:** It is significantly slower than Marching Cubes because it generates approximately 4x more geometry (triangles) [S6, unofficial][S8]. The increased vertex count also leads to higher memory bandwidth usage.
*   **Strengths:**
    *   Guarantees a manifold surface without topological ambiguities [S1, S6, unofficial].
    *   Works on unstructured tetrahedral grids as well as structured cubic grids [S7][S8].
    *   Simpler case analysis than Marching Cubes (only 8 cases, reducible to 3 by symmetry) [S7].
*   **Limitations:**
    *   Higher computational cost and memory usage due to increased triangle count [S6, S9, unofficial].
    *   Potential for degenerate triangles if interpolation is not handled carefully [S5, unofficial].
    *   Visual artifacts can occur with the "Second Algorithm" variant due to edge averaging [S9, unofficial].
*   **When to use:** When topological correctness is critical and the performance overhead is acceptable. Suitable for unstructured grids where cubic decomposition is not possible.
*   **When not to use:** When real-time performance is required and the grid is structured, in which case Marching Cubes is preferred.

## Background
Marching Tetrahedrons was introduced by Doi and Koide as an alternative to Marching Cubes to resolve the topological ambiguities present in the latter [S6, unofficial]. It works by decomposing each cubic voxel into tetrahedra, which eliminates the ambiguous cases (such as cases 6 and 7 in Marching Cubes) because a tetrahedron can only be split in one topologically consistent way for any given vertex configuration [S1, S6, unofficial]. The method has been widely studied for GPU acceleration due to its simplicity and parallelizability [S7][S8].

## Sources
[S1] adrian-sierra/Marching-Tetrahedrons, https://github.com/adrian-sierra/Marching-Tetrahedrons, fetched 2026-10-26 (unofficial)
[S2] Scrawk/Marching-Cubes, https://github.com/Scrawk/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S3] Volumetric Render Engine - GitHub, https://github.com/mikejernil/volumetric-render-engine, fetched 2026-10-27 (unofficial)
[S4] Torbenhjl/Marching-Tetrahedra-on-CT-headscan, https://github.com/Torbenhjl/Marching-Tetrahedra-on-CT-headscan, fetched 2026-10-27 (unofficial)
[S5] aaron-eugene/PlaneVoxelLab, https://github.com/aaron-eugene/PlaneVoxelLab, fetched 2026-10-27 (unofficial)
[S6] Smooth Voxel Terrain (Part 2) – 0 FPS, https://0fps.net/2012/07/12/smooth-voxel-terrain-part-2/, fetched 2026-10-28 (unofficial)
[S7] DAV Center: Marching Tetrahedra, https://daac.hpc.mil/gettingStarted/Marching_Tetrahedra.html, fetched 2026-10-29
[S8] Isosurface Extraction on GPU with CUDA - POLARIS, https://polaris.imag.fr/arnaud.legrand/teaching/2009/iso-cuda.html, fetched 2026-10-30
[S9] Two Methods for Iso-Surface Extraction from Volumetric Data and Their Comparison, https://arxiv.org/abs/2201.03446v1, fetched 2026-10-26 (unofficial)
