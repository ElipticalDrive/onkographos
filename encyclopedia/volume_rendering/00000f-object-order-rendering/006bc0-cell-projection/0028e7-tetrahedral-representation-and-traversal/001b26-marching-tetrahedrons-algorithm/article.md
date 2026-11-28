# Marching Tetrahedrons Algorithm
## Overview
The Marching Tetrahedrons Algorithm (MTA) is an iso-surface extraction method that decomposes cubic grid cells into tetrahedra to determine material boundaries [S1, unofficial]. It builds on the shared principles of [Cell projection](26-cell-projection.md). Unlike [Projected Tetrahedra](877-projected-tetrahedra.md), which projects primitives to screen space, MTA operates in object space [G]. The method introduces directional ambiguities because a cube cannot be subdivided into tetrahedra symmetrically [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field defined on a regular 3D grid of voxels [G].
*   **Output:** A polygonal mesh representing the material boundaries or isosurface [S2].

**Data Structures:**
*   **Grid:** A 3D array of vertices. Each vertex $V_i$ has coordinates $(x_i, y_i, z_i)$ and a scalar value $v_i$ [G].
*   **Tetrahedra:** Each cubic cell is decomposed into tetrahedra using a fixed, non-symmetric pattern [S1, unofficial].

**Steps:**
1.  **Grid Traversal:** Iterate through every cubic cell in the grid [S2].
2.  **Tetrahedral Decomposition:** Subdivide the current cubic cell into tetrahedra [G].
3.  **Vertex Classification:** For each tetrahedron, determine the state of its 4 vertices relative to the isosurface threshold $T$ [G]. A vertex is "inside" if $v_i < T$ (or $v_i > T$, depending on convention) and "outside" otherwise [G]. This yields $2^4 = 16$ possible configurations [G].
4.  **Intersection Calculation:** For each edge of the tetrahedron where the scalar values change sign, calculate the intersection point $P$ using linear interpolation [G].
5.  **Triangle Generation:** Based on the configuration of inside/outside vertices, select the pre-defined set of triangles that connect the intersection points [G].
6.  **Mesh Assembly:** Collect all generated triangles [G].

**Formulas:**
Intersection point $P$ on an edge between vertices $V_a$ and $V_b$ with scalar values $v_a$ and $v_b$:
$$ P = V_a + \frac{T - v_a}{v_b - v_a} (V_b - V_a) $$
where $T$ is the isosurface threshold [G].

**Parameters:**
*   **Isosurface Threshold ($T$):** The scalar value defining the surface [G].
*   **Tetrahedral Decomposition Pattern:** The specific way cubes are split [G].

**Edge Cases and Pitfalls:**
*   **Directional Ambiguities:** Because a cube cannot be subdivided into tetrahedra symmetrically, the MTA exhibits directional ambiguities [S1, unofficial].
*   **Missed Boundaries:** The algorithm can miss material boundaries if the isosurface enters a tetrahedron but only intersects one edge, as detection relies on vertex information [S2].
*   **Not given in the sources:** Exact lookup table sizes, specific decomposition patterns, or handling of degenerate triangles.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_tetrahedrons(grid, threshold):
    triangles = []
    # Standard lookup table mapping 4-bit vertex configurations to edge pairs
    # Not given in the sources.
    LOOKUP_TABLE = { ... } 
    
    for cell in grid.cells:
        # Decompose cubic cell into tetrahedra [G]
        tetrahedra = decompose_cell_into_tetrahedra(cell)
        
        for tet in tetrahedra:
            # Vertex classification [G]
            config = 0
            for i, vertex in enumerate(tet.vertices):
                if vertex.value > threshold:
                    config |= (1 << i)
            
            # Triangle generation [G]
            edge_pairs = LOOKUP_TABLE.get(config, [])
            for edge_pair in edge_pairs:
                v0, v1 = tet.vertices[edge_pair[0]], tet.vertices[edge_pair[1]]
                v2, v3 = tet.vertices[edge_pair[2]], tet.vertices[edge_pair[3]]
                
                p1 = interpolate(v0, v1, threshold)
                p2 = interpolate(v2, v3, threshold)
                triangles.append([p1, p2])
                
    return triangles

def interpolate(va, vb, threshold):
    # Linear interpolation [G]
    t = (threshold - va.value) / (vb.value - va.value)
    return va.coords + t * (vb.coords - va.coords)

def decompose_cell_into_tetrahedra(cell):
    # Fixed non-symmetric decomposition [G]
    # Returns list of tetrahedra objects with vertex references
    pass
```
## Subtopics

### [Regularised Marching Tetrahedra](1301-regularised-marching-tetrahedra.md)
Regularised Marching Tetrahedra (RMT) is an iso-surface extraction algorithm that combines the topological consistency of the Marching Tetrahedrons Algorithm with vertex clustering to generate simplified surfaces with improved triangle aspect ratios. Unlike standard Marching Tetrahedra, which can produce thin, degenerate triangles that degrade interpolated shading, RMT reduces the triangle count by approximately 70% while preserving the original topology by basing clustering on the tetrahedral lattice. Choose this when shading quality and mesh simplification are priorities over raw geometric fidelity. [Regularised Marching Tetrahedra](1301-regularised-marching-tetrahedra.md)

### [Subgrid Pipeline](1640-subgrid-pipeline.md)
The Subgrid Pipeline extracts isosurfaces from tetrahedral grids by resolving topological ambiguities within individual tetrahedra using subgrid-accurate intersection data. It distinguishes itself from standard Marching Tetrahedrons by explicitly handling open, scoop, and normal boundary curves to process complex intersection configurations. It builds on [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md) by replacing the fixed lookup table with a local reconstruction procedure that processes boundary curves. Choose this for complex topologies where standard MTA fails to capture fine details. [Subgrid Pipeline](1640-subgrid-pipeline.md)

### [Tetrahedral Normal Calculation](1941-tetrahedral-normal-calculation.md)
This method computes smooth isosurface normals for tetrahedral meshes by calculating vertex normals weighted by the volume of connected tetrahedra. It solves the problem of severe visual discontinuities that arise when using the constant cell normal $N_t$ directly, which decreases rendering quality significantly. It differs from its siblings, such as [Regularised Marching Tetrahedra](1301-regularised-marching-tetrahedra.md) which modifies the topology, by retaining the standard Marching Tetrahedrons topology but enhancing the shading data through pre-computed vertex attributes. Choose this when smooth shading is required without altering the mesh geometry. [Tetrahedral Normal Calculation](1941-tetrahedral-normal-calculation.md)

### [McGrids](2289-mcgrids.md)
McGrids is an iso-surface extraction method that constructs adaptive, non-uniform tetrahedral grids using an iterative Monte Carlo process to estimate a probability density function (PDF) based on the implicit field, followed by a refinement step and mesh extraction via [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md). It solves the problem of inefficient uniform grid sampling by concentrating grid points near surfaces and geometric details, significantly reducing the number of implicit field queries and memory usage compared to uniform grids. Unlike [Regularised Marching Tetrahedra](1301-regularised-marching-tetrahedra.md) or [Subgrid Pipeline](1640-subgrid-pipeline.md) which may rely on fixed or hierarchical structures, McGrids dynamically adapts the grid topology through probabilistic sampling and Centroidal Voronoi Tessellation (CVT) relaxation. Choose this for large datasets with sparse features to optimize performance. [McGrids](2289-mcgrids.md)

### [Exact Fiber Surface Extraction](3445-exact-fiber-surface-extraction.md)
Exact Fiber Surface Extraction computes the pre-image of a 1-manifold path (Fiber Surface Control Polygon, FSCP) in the range of a bivariate field $f: \mathbb{R}^3 \to \mathbb{R}^2$ over a tetrahedral mesh. It solves the problem of approximating these surfaces using signed distance fields, which fail to capture sharp bends and complex topologies. Unlike [Regularised Marching Tetrahedra](1301-regularised-marching-tetrahedra.md) which smooths geometry, this method guarantees planar pre-images within each tetrahedron by decomposing the extraction into base surface generation and precise clipping. Choose this for bivariate field visualization requiring exact topological preservation. [Exact Fiber Surface Extraction](3445-exact-fiber-surface-extraction.md)

### [Marching Tetrahedra Reconstruction](3516-marching-tetrahedra-reconstruction.md)
Marching Tetrahedra Reconstruction is a surface reconstruction method that converts a set of input vertices and normals into a 3D mesh by computing an implicit field using the Welland function, sampling it on a tetrahedral grid, and extracting the zero-level set via Marching Tetrahedra. It differs from standard Marching Cubes by using a tetrahedral decomposition of the grid cells, which eliminates the ambiguity cases inherent in cube-based extraction. It builds on [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md) but replaces the standard linear interpolation with a Welland-based implicit field evaluation. Choose this for point cloud reconstruction where topological correctness is critical. [Marching Tetrahedra Reconstruction](3516-marching-tetrahedra-reconstruction.md)

### [Raster Algorithm for Fiber Surface Extraction](4960-raster-algorithm-for-fiber-surface-extraction.md)
The Raster Algorithm for Fiber Surface Extraction approximates fiber surfaces by extracting the 0 level-set of a signed range distance field to the Fiber Surface Control Polygon (FSCP). It reduces the computational cost of distance field evaluation by rasterizing the scalar field range into a 2D grid, trading geometric accuracy for speed. Unlike exact methods, it imposes a lower bound on Hausdorff distance near FSCP bends and only handles closed FSCPs. Choose this for real-time applications where approximate fiber surfaces are sufficient. [Raster Algorithm for Fiber Surface Extraction](4960-raster-algorithm-for-fiber-surface-extraction.md)

### [Keppel Heuristic](10336-keppel-heuristic.md)
The Keppel Heuristic is a local decision method used in contour tiling to determine how to connect two parallel contour segments during volume rendering. It resolves the ambiguity of the next vertex to connect by comparing the volumes of two candidate tetrahedra formed with interior reference points. Choose this when resolving contour connections in tetrahedral-based contouring. [Keppel Heuristic](10336-keppel-heuristic.md)

### [Tetrahedralization Schemes](10948-tetrahedralization-schemes.md)
Tetrahedralization Schemes define how a cubic voxel cell is decomposed into tetrahedra to facilitate isosurface extraction, addressing the topological ambiguities and artifacts inherent in standard cube-based methods like Marching Cubes [G]. Unlike sibling methods that may use fixed lookup tables or specific lattice constraints, this family focuses on the geometric partitioning strategy itself—determining vertex counts, tetrahedron shapes, and neighbor dependencies [G]. It sets itself apart by offering alternatives to the standard 5-tetrahedron or 6-tetrahedron decompositions, such as schemes using 12 or 24 tetrahedra, or those based on non-cubic lattices like BCC or Diamond, to improve surface smoothness or reduce lookup table memory [G]. Choose this when the underlying grid structure is non-cubic or requires specific connectivity properties. [Tetrahedralization Schemes](10948-tetrahedralization-schemes.md)

### [MTCut](11818-mtcut.md)
MTCut is a real-time volume cutting algorithm that modifies isosurfaces generated by the Marching Tetrahedra (MT) algorithm using a user-defined cutting tool. Unlike traditional tetrahedral mesh cutting strategies that require a pre-computed tetrahedrization of the entire volume, MTCut computes the tetrahedrization of only the intersected cells on-the-fly, enabling interactive performance for complex meshes. It produces a watertight manifold model, distinguishing it from approaches like Zachow et al. [ZGSZ03] that yield open surfaces with borders. Choose this for interactive volume cutting applications. [MTCut](11818-mtcut.md)

### Nielson-Franke Generalization
The Nielson-Franke Generalization is a direct generalization of Nielson's and Franke's algorithm for multi-material interfaces. It reduces to the Nielson-Franke algorithm when material fractions are restricted such that exactly one $l_i=1$ at each vertex. It uses a clipping algorithm (e.g., BSP) to calculate material boundaries within tetrahedra and maps them back to physical space. In the three-material case, it assumes a "Y" shaped subdivision in the barycentric triangle centered at $c = (1/3, 1/3, 1/3)$. Choose this for multi-material volume rendering where material interfaces need to be explicitly resolved. [Nielson-Franke Generalization](1301-regularised-marching-tetrahedra.md)

## Operating characteristics
*   **Time Complexity:** The algorithm runs in effectively the same time as the marching cubes/tetrahedra algorithm, i.e., linear with respect to the number of cells in the grid [S2].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Directional ambiguities due to non-symmetric tetrahedral decomposition [S1, unofficial]. The algorithm can miss material boundaries if the isosurface enters a tetrahedron but only intersects one edge [S2].
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.
## Background
The development of the algorithm is discussed alongside the history of the Marching Cubes Algorithm, which initially overlooked cases that resulted in accidental holes in generated surfaces [S1, unofficial]. The MTA decomposes cubes into tetrahedra but introduces directional ambiguities because a cube cannot be subdivided symmetrically [S1, unofficial]. Not given in the sources: specific authors, dates, or primary papers for the original MTA.
## Sources
[S1] Volume-Enclosing Surface Extraction, https://arxiv.org/abs/1011.1787v4, fetched 2026-10-26 (unofficial)
[S2] Constructing Material Interfaces from Data Sets with Volume-Fraction Information, https://digital.library.unt.edu/ark:/67531/metadc739076/?q=marching cubes, fetched 2026-10-28
