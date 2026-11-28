# Gallagher-Nagtegaal Bicubic Method
Also called: GN Bicubic, Unstructured mesh marching cubes variant
## Overview
The Gallagher-Nagtegaal Bicubic Method is an isosurface extraction technique that generates visually continuous bicubic polynomial patches instead of triangular facets [S1, unofficial]. It extends the marching cubes concept to work on both unstructured and rectilinear meshes [S1, unofficial]. The method processes each volume element in isolation, minimizing information accessed in the inner loop [S1, unofficial].
## How it works
**Inputs:** A scalar field defined on an unstructured or rectilinear grid; an isovalue [G].
**Outputs:** A mesh of bicubic polynomial patches [G].

**Data Structures:**
Each volume element (cell) contains scalar values at its vertices [G]. The method supports arbitrary cell topologies on unstructured or rectilinear grids [S1, unofficial].

**Steps:**
1. Traverse the grid cells [G].
2. Evaluate the scalar field at the vertices of each cell [G].
3. Compare vertex values to the isovalue to detect surface intersections [G].
4. Compute intersection points on the cell edges [G].
5. Fit a visually continuous bicubic polynomial patch to the intersection geometry within the cell [S1, unofficial].
6. Output the patch as part of the final surface representation [G].
7. Connect adjacent patches to form a coherent mesh [G].

**Formulas:**
Not given in the sources.

**Parameters:**
Isovalue threshold; grid type (unstructured or rectilinear) [S1, unofficial].

**Edge cases and pitfalls:**
Not given in the sources.
## Subtopics

### [Surface Patches](5930-surface-patches.md)
Surface patches are a class of surface representation techniques that use non-linear polynomials or splines to map a given surface, allowing for the natural rendering of curved geometry.

## Operating characteristics
**Time and memory complexity:**
Processing each volume element in isolation minimizes information accessed in the inner loop, which is desirable for efficient microcoding on a single workstation [S1, unofficial].

**Measured performance:**
Not given in the sources.

**Scaling:**
The independent treatment of elements makes the algorithm suitable for parallel implementations [S1, unofficial].

**Strengths and limitations:**
Strengths: Visually continuous bicubic polynomials; works on unstructured and rectilinear meshes; efficient inner loop; parallelizable [S1, unofficial].
Limitations: Not given in the sources.

**When to use it and when not:**
Use when high-quality, smooth surface rendering is required on unstructured grids and parallelization is desired [S1, unofficial]. Not given in the sources for when not to use.
## Background
The method was described by Gallagher and Nagtegaal in 1989 [S1, unofficial]. It builds upon the marching cubes method but adapts it for unstructured and rectilinear meshes using bicubic polynomials [S1, unofficial].
## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
