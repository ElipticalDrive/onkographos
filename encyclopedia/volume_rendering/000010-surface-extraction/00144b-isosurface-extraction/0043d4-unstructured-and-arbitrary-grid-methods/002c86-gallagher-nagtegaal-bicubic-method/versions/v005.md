# Gallagher-Nagtegaal Bicubic Method
Also called: GN Bicubic, Unstructured mesh marching cubes variant
## Overview
The Gallagher-Nagtegaal Bicubic Method is an isosurface extraction technique that generates visually continuous bicubic polynomial patches instead of triangular facets [S1, unofficial]. It extends the marching cubes concept to work on both unstructured and rectilinear meshes [S1, unofficial]. Unlike polygonal approximations, it defines the surface analytically, allowing for high-quality shading without low-tessellation artifacts [S1, unofficial]. It processes each volume element in isolation, minimizing inner-loop data access and enabling efficient microcoding and parallelization [S1, unofficial].

## How it works
**Inputs:** A scalar field defined on an unstructured or rectilinear grid; an isovalue.
**Outputs:** A mesh of bicubic polynomial patches.

**Data Structures:**
Each volume element (cell) contains scalar values at its vertices. The specific topology (hexahedral, tetrahedral, etc.) is not restricted to regular grids [S1, unofficial].

**Steps:**
1. Traverse the grid cells in a predetermined order (e.g., front-to-back) [S1, unofficial].
2. For each cell, evaluate the scalar field at the vertices.
3. Determine if the isosurface intersects the cell by comparing vertex values to the isovalue.
4. If an intersection exists, compute the intersection points on the cell edges.
5. Fit a bicubic polynomial patch to the intersection geometry within the cell. The patch must be visually continuous with adjacent patches [S1, unofficial].
6. Output the patch as part of the final surface representation.

**Formulas:**
Not given in the sources. The method uses bicubic polynomials, but the specific basis functions and continuity constraints are not detailed in the provided text [S1, unofficial].

**Parameters:**
Isovalue: The threshold for surface extraction.
Grid type: Unstructured or rectilinear [S1, unofficial].

**Edge cases and pitfalls:**
Continuity between patches must be ensured to avoid visual artifacts [S1, unofficial]. The method's reliance on processing elements in isolation requires careful handling of shared boundaries to maintain global smoothness [S1, unofficial].

## Subtopics
## Operating characteristics
**Time and memory complexity:**
Processing each volume element in isolation minimizes information accessed in the inner loop, which is desirable for efficient microcoding on a single workstation [S1, unofficial]. The independent treatment of elements makes the algorithm suitable for parallel implementations [S1, unofficial].

**Measured performance:**
Not given in the sources.

**Scaling:**
The algorithm scales well for parallel implementations due to the independent processing of volume elements [S1, unofficial].

**Strengths and limitations:**
Strengths: Visually continuous bicubic polynomials; works on unstructured and rectilinear meshes; efficient inner loop; parallelizable [S1, unofficial].
Limitations: Not given in the sources.

**When to use it and when not:**
Use when high-quality, smooth surface rendering is required on unstructured grids and parallelization is desired [S1, unofficial]. Not given in the sources for when not to use.

## Background
The method was described by Gallagher and Nagtegaal in 1989 [S1, unofficial]. It builds upon the marching cubes method but adapts it for unstructured and rectilinear meshes using bicubic polynomials [S1, unofficial].

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
