# Exact AMR Rendering

Also called: AMRHealpixRenderer, AMRImageRenderer

## Overview
Exact AMR Rendering treats Adaptive Mesh Refinement (AMR) gas cells as piecewise-constant geometric cubes rather than smoothing them as SPH kernels [S1]. It solves the problem of bias and smoothing inherent in SPH rendering by providing exact pixel-averaged cross-sections for slices, projections, and perspective images, ensuring mass conservation to round-off [S1]. It differs from its sibling [TET-AMR Data Structure](1094-tet-amr-data-structure.md) by assuming cubic, axis-aligned (or rigidly rotated) cells with constant values, whereas TET-AMR handles unstructured tetrahedral elements [S1]. It differs from [Approximate perspective projection of AMR cells](4400-approximate-perspective-projection-of-amr-cells.md) by avoiding orthographic approximations for large cells or when `approximate_fast=False`, ensuring geometric fidelity where perspective distortion is significant [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** AMR cell data (positions, sizes `smooth`, densities `rho`, masses, and scalar quantities `q`). Cells are defined as cubes of side `smooth` with edges aligned to the original simulation axes [S1].
*   **Output:** A 2D image (slice, projection, or perspective) or a HEALPix map [S1].
*   **Configuration:** Opt-in via `use_true_amr=True` or config `use-true-amr` in `[sph]` [S1].

**Data Structures:**
*   **AMR Families:** Declared in `SimSnap._amr_cell_families` (e.g., `'gas'` for Ramses) [S1].
*   **Renderer:** `AMRImageRenderer` (subclass of `ImageRenderer`) handles array units and wrapping; `AMRHealpixRenderer` handles HEALPix maps [S1].
*   **Corner Function Table:** For rotated projections, a tabulated function $F(X,Y)$ stores the volume of a reference cell with $x \le X, y \le Y$ [S1].

**Steps:**

1.  **Initialization:**
    *   Verify snapshot declares AMR cells. Raise `ValueError` if mixing families, if a kernel is given, or if `target='volume'` [S1].
    *   Apply orientation via `Transformable.net_rotation_matrix` [S1].

2.  **Slices:**
    *   Compute exact pixel-averaged cross-section [S1].
    *   **Aligned Cells:** Use separable overlaps [S1].
    *   **Rotated Cells:** Use polygon clipping [S1].
    *   **Boundary Condition:** A slice lying exactly on a cell face consistently includes the $+z$ cell [S1].

3.  **Orthographic Projections:**
    *   **Aligned Cells:** Use exact separable overlaps [S1].
    *   **Rotated Cells (Approximate):** If `approximate_fast=True` and cell size $\le 16$ px:
        *   Interpolate the tabulated corner function $F(X,Y)$ at pixel corners [S1].
        *   Compute pixel value as the difference of $F(X,Y)$ across the pixel corners [S1].
        *   Formula:
            $$ \text{Pixel Value} = \sum_{\text{corners}} \sigma_i F(X_i, Y_i) $$
            where $\sigma_i$ are the signed weights of the pixel corners relative to the cell projection [S1].
        *   This is mass-exact with typical errors $10^{-5}$ and max $10^{-3}$ [S1].
    *   **Rotated Cells (Exact):** If `approximate_fast=False`, cell size $> 16$ px, or cut by depth limits:
        *   Use an exact row-difference method [S1].
        *   Cost scales with cell perimeter [S1].
        *   Exact corner volumes derived from the divergence theorem [S1].
        *   Accumulate second differences along each pixel row [S1].

4.  **Perspective Projections:**
    *   Compute $\int q \, dl$ along each ray from the camera [S1].
    *   **Geometry:** Under perspective mapping, faces remain planar. The depth $z$ where a ray crosses a face is $K/L(X,Y)$ with $L$ affine [S1].
    *   **Integration:**
        *   A pixel's integral is a sum of $\int 1/L$ over convex polygons formed by clipping the cube to the pixel's cone [S1].
        *   Use series from polygon moments where they converge, otherwise use exact Hermite–Genocchi formulas on triangles [S1].
        *   Integrate the $\sec\theta$ path-length factor to second order per pixel [S1].
        *   Shared faces cancel, ensuring exact tiling [S1].
    *   **Approximation:** If `approximate_fast=True`:
        *   If the perspective image differs from the scaled orthographic image by $< \text{perspective\_tolerance}$ (default 0.1 px) at every corner, use orthographic tables [S1].
        *   RMS errors $\approx 10^{-4}$ of the peak [S1].
    *   **Perspective Slices:** Rendered as orthographic slices of the plane, rescaled [S1].

5.  **HEALPix Projections:**
    *   Each pixel receives $q V(\text{cell} \cap \text{cone}_p) / \Omega_p$ [S1].
    *   **Exact Clipping:** Clip each cube's faces to the planes through the pixel's corners [S1].
    *   **Correction:** Apply corrections for slivers between HEALPix pixel sides and great circles (area correction to $3 \times 10^{-5}$) [S1].
    *   **Thin Shells:** If `shell_distance` is set, sample sub-pixels. Accept sub-pixels wholly inside/outside; compute edge fractions using the tangent plane of the shell [S1].

6.  **Edge Cases and Pitfalls:**
    *   **Out-of-bounds:** Skip cells with non-finite positions/sizes or non-positive sizes [S1].
    *   **Round-off:** Skip cells just beyond the image edge (right/top) that compute pixel index $n_x$ or $n_y$ after bounding-box tests [S1].
    *   **SPH Bugs Fixed:** Perspective integration now uses impact parameter perpendicular to ray (fixing $\cos\theta$ faintness); slices use slice plane depth, not particle depth [S1].

**Parameters:**
*   `approximate_fast`: Boolean. Default `True`. Allows table interpolation for small rotated cells and orthographic approximation for perspective [S1].
*   `perspective_tolerance`: Float. Default `0.1` px. Threshold for using orthographic tables in perspective projections [S1].

## Subtopics

### [Exact perspective projection of AMR cells](4428-exact-perspective-projection-of-amr-cells.md)
Renders perspective images by computing the pixel average of the integral of the quantity along each ray. It avoids orthographic approximations for cells where perspective distortion is significant, ensuring geometric fidelity. Choose this when high-fidelity perspective views of large or near-field AMR cells are required.

## Operating characteristics
*   **Time Complexity:**
    *   Slices: Faster than SPH (0.13s vs 0.58s for 500² Ramses slice) [S1].
    *   Orthographic Projections: Comparable to SPH for small cells; slower for large cells if `approximate_fast=False` (0.63s vs 0.45s for 500² Ramses projection) [S1].
    *   Perspective Projections: Slower than SPH for small cells (4.31s vs 5.90s for 500² Ramses perspective) but faster than exact SPH in some configurations; exact mode (`approximate_fast=False`) is significantly slower (3.72s vs 4.31s) [S1].
    *   HEALPix: Slower than SPH (4.55s vs 3.59s for nside 64) [S1].
*   **Memory Complexity:** In-place accumulation reduces peak memory to 12 bytes per pixel for rotated projections [S1].
*   **Accuracy:** Mass conservation error for HEALPix is $2 \times 10^{-8}$ (vs $5 \times 10^{-3}$ for SPH) [S1]. Rotated projection errors are typically $10^{-5}$, max $10^{-3}$ [S1].
*   **Strengths:** Exact mass conservation, no smoothing bias, correct perspective geometry.
*   **Limitations:** Slower than SPH for large-scale projections; requires opt-in.

## Background
Exact AMR Rendering was introduced in pynbody version 2.9.0 as part of the `pynbody.imaging` package, replacing the previous SPH-centric rendering for AMR gas (e.g., Ramses) [S1]. It was developed to address the bias and smoothing of SPH kernels when rendering AMR data, and to fix bugs in SPH perspective rendering (incorrect path length integration and slice depth placement) [S1].

## Sources
[S1] Opt-in exact AMR rendering, new pynbody.imaging package, and SPH perspective fixes, https://github.com/pynbody/pynbody/pull/1048, fetched 2026-10-26
