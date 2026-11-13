# Gallagher-Nagtegaal Bicubic Method

Also called: GN Bicubic, Unstructured mesh marching cubes variant

## Overview
The Gallagher-Nagtegaal Bicubic Method is an isosurface extraction technique that replaces the triangular facets of traditional marching cubes with visually continuous bicubic polynomial patches [S1, unofficial]. It operates on unstructured or rectilinear grids by processing each volume element in isolation [S1, unofficial]. Unlike sibling methods that rely on linear interpolation or global mesh coherence, this method prioritizes local smoothness and independent element processing to facilitate efficient microcoding and parallelization [S1, unofficial]. It builds on [Unstructured and Arbitrary Grid Methods](11103-unstructured-and-arbitrary-grid-methods.md) and differs from [Isosurface extraction on arbitrary polyhedral grids](951-isosurface-extraction-on-arbitrary-polyhedral-grids.md) by using bicubic patches instead of linear primitives.

## How it works
**Inputs:**
* Scalar field data defined on an unstructured or rectilinear grid [S1, unofficial].
* An isovalue threshold $\psi$ [G].

**Outputs:**
* A set of bicubic surface patches representing the isosurface [S1, unofficial].

**Data Structures:**
* **Cell Data:** For each cell, the scalar values at the vertices are stored [G]. The specific topology of the cell determines the number of vertices [G].
* **Polynomial Coefficients:** A set of coefficients defining the bicubic polynomial for the patch within the cell [S1, unofficial].

**Steps:**
1. **Traversal:** Iterate over each volume element (cell) in the grid [S1, unofficial].
2. **Isosurface Detection:** Evaluate the scalar field against the isovalue to detect intersections at the cell vertices [G].
3. **Coefficient Calculation:** Calculate the coefficients of the bicubic polynomial that interpolates the scalar values at the cell vertices [S1, unofficial].
4. **Patch Generation:** Generate the surface patch using the calculated bicubic polynomial [S1, unofficial].
5. **Output:** Add the generated patch to the output mesh [G].

**Formulas:**
Not given in the sources.

**Parameters:**
* **Isovalue:** The threshold scalar value defining the surface [G].
* **Continuity:** The method aims for visual continuity across cell boundaries [S1, unofficial].

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### Surface Patches
Surface patches are a class of surface representation using non-linear polynomials or splines to map a given surface [S2, unofficial]. The most widely used mappings are the B-spline and the bi-cubic families [S2, unofficial]. Tessellation methods can be used to define the control points (or nodes) of the surface patch [S2, unofficial]. Catmull's subdivision procedure recursively divides the region into sub-patches until the transformed sub-region covers only one pixel [S2, unofficial]. Scan-line implementations of this algorithm were developed by Blinn and Lane-Carpenter [S2, unofficial].

## Operating characteristics
* **Time Complexity:** The method processes each cell independently, leading to $O(N)$ complexity where $N$ is the number of cells [S1, unofficial].
* **Memory Complexity:** Memory usage is proportional to the number of cells, as each cell is processed in isolation [S1, unofficial].
* **Performance:** The independence of cell processing makes the algorithm suitable for parallel implementations [S1, unofficial].
* **Strengths:** Produces visually continuous surfaces without the faceted appearance of triangular meshes. Efficient for parallelization and microcoding due to local data access [S1, unofficial].
* **Limitations:** Not given in the sources.

## Background
The method was described by Gallagher and Nagtegaal in 1989 [S1, unofficial]. It builds upon the "marching cubes" method to work on unstructured as well as rectilinear meshes [S1, unofficial]. The key innovation is the use of visually continuous bicubic polynomials instead of triangles [S1, unofficial].

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S2] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
