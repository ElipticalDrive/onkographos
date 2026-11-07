# Exact AMR Rendering

Also called: AMRHealpixRenderer, AMRImageRenderer

## Overview
Exact AMR Rendering treats Adaptive Mesh Refinement (AMR) gas cells as piecewise-constant geometric cubes rather than smoothing them as SPH kernels [S1]. It solves the problem of biased, smoothed representations in standard SPH rendering by providing exact pixel-averaged cross-sections for slices, exact volume integrals for projections, and exact mass-conserving maps for HEALPix [S1]. It differs from its sibling [Approximate perspective projection of AMR cells](4400-approximate-perspective-projection-of-amr-cells.md) by avoiding orthographic approximations for perspective views, ensuring geometric fidelity where perspective distortion is significant [S1]. It builds on [Exact AMR Rendering](4399-exact-amr-rendering.md) by treating cells as piecewise-constant cubic volumes rather than particles [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** AMR cell data (positions, sizes `smooth`, densities `rho`, masses, and scalar quantities `q`), camera parameters (view direction, distance, field of view), and image geometry (resolution, pixel size).
*   **Output:** A 2D array (image) or HEALPix map containing the integrated quantity per pixel.

**Data Structures:**
*   **AMR Cell Definition:** Each cell is a cube of side length `smooth`, with volume $V = \text{mass}/\rho$, and edges aligned to the original simulation axes [S1].
*   **Family Declaration:** Snapshots declare AMR families via `SimSnap._amr_cell_families` (e.g., set to `'gas'` for Ramses) [S1].
*   **Renderer Selection:** The renderer is selected automatically for AMR gas if `amr=True` or if the snapshot consists only of AMR cells and no kernel is specified [S1]. It is a subclass of `ImageRenderer`, inheriting array handling, units, and periodic wrapping, but rejects any kernel setting [S1].

**Parameters:**
*   `use_true_amr`: Boolean. Enables AMR rendering. Default is `False` (opt-in) [S1].
*   `approximate_fast`: Boolean. If `True`, allows approximations for speed (e.g., tabulated orthographic projections for small rotated cells). Default is `True` [S1].
*   `perspective_tolerance`: Float. Threshold (in pixels) for using orthographic tables in perspective projections. Default is `0.1` px [S1].

**Steps:**

1.  **Initialization:**
    *   Verify snapshot format declares AMR cells. Raise `ValueError` if families are mixed, a kernel is given, or `target='volume'` is specified [S1].
    *   Apply orientation via `Transformable.net_rotation_matrix` [S1].

2.  **Filtering:**
    *   Skip cells with non-finite positions, non-finite sizes, or non-positive sizes [S1].
    *   Skip cells lying entirely outside the image bounding box. Handle round-off errors where a cell just beyond the right/top edge might compute an index of `nx` or `ny` by skipping such cells after bounding-box tests [S1].

3.  **Orthographic Projections (Slices and Projections):**
    *   **Aligned Cells:** Use exact separable overlaps [S1].
    *   **Rotated Cells (Small, $\le 16$ px):** Use the approximate fast path. Interpolate a tabulated corner function $F(X,Y)$, defined as the volume of the cell with $x \le X, y \le Y$ [S1].
        *   The integral of a cell's column over a pixel is a difference of $F(X,Y)$ across the pixel corners [S1].
        *   $F(X,Y)$ is tabulated exactly once per orientation and bilinearly interpolated [S1].
        *   This method is mass-exact, with pixel errors typically $10^{-5}$ and at most $10^{-3}$ [S1].
    *   **Rotated Cells (Large or Exact Path):** Use the exact row-difference method. Cost scales with the cell's perimeter. Exact corner volumes are derived from the divergence theorem [S1].
    *   **Slices:** Provide exact pixel-averaged cross-sections. A slice lying exactly along cell faces consistently takes the $+z$ cells [S1].

4.  **Perspective Projections:**
    *   Compute the integral of quantity $q$ along the path length $dl$ for each ray from the camera: $\int q \, dl$ [S1].
    *   **Geometry:** Under the perspective map, each face stays planar. The depth where a ray crosses a face is $K/L(X,Y)$ with $L$ affine [S1].
    *   **Integration:** A pixel's integral is a sum of $\int 1/L$ over convex polygons. These integrals use series from polygon moments where they converge, otherwise exact Hermite–Genocchi formulas on triangles [S1].
    *   **Path Length:** The $\sec\theta$ path-length factor is integrated to second order per pixel [S1].
    *   **Approximation:** If `approximate_fast=True`, cells whose perspective image differs from their scaled orthographic image by less than `perspective_tolerance` (0.1 px) at every corner use the orthographic tables [S1]. This yields r.m.s. errors around $10^{-4}$ of the peak for small cells [S1].
    *   **Conservation:** Shared faces cancel, so cells tile exactly [S1].

5.  **HEALPix Projections:**
    *   Each pixel receives $q V(\text{cell} \cap \text{cone}_p) / \Omega_p$, where $\Omega_p$ is the pixel solid angle [S1].
    *   **Exact Clipping:** Clip each cube to the pixel's cone using planes through the pixel's corners [S1].
    *   **Correction:** Apply corrections for slivers between HEALPix pixel sides and great circles (pixel areas correct to $3 \times 10^{-5}$) [S1].
    *   **Thin Shells:** If `shell_distance` is set, sample sub-pixels. Accept sub-pixels wholly inside or outside a cell. Compute the fraction inside at edges using the tangent plane of the shell [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_amr_cells(cells, image_shape, camera, approximate_fast=True, perspective_tolerance=0.1):
    """
    Render AMR cells as exact geometric cubes.
    
    Args:
        cells: List of AMR cell objects with attributes:
            - pos: (x, y, z) position
            - smooth: side length of the cube
            - mass, rho: for volume calculation
            - q: scalar quantity to render
        image_shape: (height, width) in pixels
        camera: Camera object with methods for ray projection
        approximate_fast: Boolean, allow tabulated approximations
        perspective_tolerance: Float, threshold for perspective approximation
        
    Returns:
        image: 2D numpy array of rendered values
    """
    image = np.zeros(image_shape)
    
    # Pre-compute orientation if needed
    # Apply net_rotation_matrix to cell positions and sizes
    
    for cell in cells:
        # Skip malformed cells
        if not np.isfinite(cell.pos).all() or not np.isfinite(cell.smooth) or cell.smooth <= 0:
            continue
            
        # Bounding box check
        if not cell_in_bounding_box(cell, image_shape):
            continue
            
        # Determine projection type based on camera
        if camera.is_perspective:
            render_perspective_cell(cell, image, camera, approximate_fast, perspective_tolerance)
        else:
            render_orthographic_cell(cell, image, camera, approximate_fast)
            
    return image

def render_perspective_cell(cell, image, camera, approximate_fast, perspective_tolerance):
    """
    Render a single AMR cell in perspective view.
    """
    # Check if approximation is valid
    if approximate_fast and is_approximation_valid(cell, camera, perspective_tolerance):
        # Use orthographic tables
        render_orthographic_cell_approx(cell, image, camera)
    else:
        # Exact perspective integration
        # 1. Project cell faces to image plane
        # 2. For each pixel intersecting the projected cell:
        #    a. Determine the convex polygon of intersection
        #    b. Compute integral of 1/L over the polygon
        #    c. Integrate sec(theta) to second order
        #    d. Add to image pixel
        exact_perspective_integral(cell, image, camera)

def render_orthographic_cell_approx(cell, image, camera):
    """
    Render using tabulated corner function F(X,Y).
    """
    # F(X,Y) = vol(cell with x <= X, y <= Y)
    # Tabulate F once per orientation
    # Bilinearly interpolate F for cell corners
    # Compute pixel values as differences of F across pixel corners
    pass

def exact_perspective_integral(cell, image, camera):
    """
    Compute exact perspective integral using Hermite-Genocchi formulas.
    """
    # For each pixel:
    #   - Clip cell faces to pixel cone
    #   - Sum integrals over convex polygons
    #   - Use polygon moments series or Hermite-Genocchi on triangles
    pass

def is_approximation_valid(cell, camera, tolerance):
    """
    Check if perspective image differs from scaled orthographic by < tolerance.
    """
    # Compare corner projections
    pass
```

## Subtopics

### [Exact perspective projection of AMR cells](4428-exact-perspective-projection-of-amr-cells.md)
A method to render perspective images of Adaptive Mesh Refinement (AMR) cells exactly by computing the pixel average of the integral of the quantity along each ray from the camera. It differs from its sibling [Approximate perspective projection of AMR cells](4400-approximate-perspective-projection-of-amr-cells.md) by avoiding orthographic approximations, ensuring geometric fidelity for cells where perspective distortion is significant. It builds on [Exact AMR Rendering](4399-exact-amr-rendering.md) by treating cells as piecewise-constant cubic volumes rather than particles.

## Operating characteristics
*   **Time Complexity:** Scales with the number of cells and their projected perimeter/area. For orthographic projections, small rotated cells are $O(1)$ per cell (table lookup), while large cells scale with perimeter [S1]. Perspective projections are more expensive, scaling with the number of pixels covered by the cell [S1].
*   **Memory Complexity:** Low for orthographic approximations (tabulated function cached per orientation). Exact perspective and row-difference methods require temporary buffers, peaking at 12 bytes per pixel for large rotated cells [S1].
*   **Measured Performance:** On Ramses test data (1.75M cells, 60 kpc, 500² resolution):
    *   Slice: 0.13s (AMR) vs 0.58s (SPH) [S1].
    *   Projection: 0.45s (AMR) vs 0.30s (SPH) [S1].
    *   Perspective Projection: 4.31s (AMR) vs 5.90s (SPH) [S1].
    *   HEALPix Projection: 4.55s (AMR) vs 3.59s (SPH) [S1].
    *   Exact mode (`approximate_fast=False`) is slower: 0.63s for projection, 3.72s for perspective [S1].
*   **Strengths:** Exact mass conservation (error $2 \times 10^{-8}$ for HEALPix vs $5 \times 10^{-3}$ for SPH) [S1]. No bias or smoothing. Exact tiling of subdivided cells [S1].
*   **Limitations:** Slower than SPH for projections and HEALPix maps [S1]. Approximations may introduce small errors ($10^{-4}$ r.m.s. for perspective) [S1].
*   **When to Use:** When exact geometric representation is required, mass conservation is critical, or when comparing against analytical solutions. Avoid when performance is paramount and SPH smoothing is acceptable.

## Background
Exact AMR Rendering was introduced in pynbody version 2.9.0 as part of a new `pynbody.imaging` package [S1]. It was developed to address the limitations of SPH rendering for AMR data, specifically the bias and smoothing introduced by kernel interpolation [S1]. The implementation includes fixes for SPH perspective rendering bugs (integration along z instead of oblique ray, incorrect slice depth placement) [S1]. The core logic is implemented in Cython modules `_render_amr.pyx` and `_render_amr_healpix.pyx` [S1].

## Sources
[S1] Opt-in exact AMR rendering, new pynbody.imaging package, and SPH perspective fixes, https://github.com/pynbody/pynbody/pull/1048, fetched 2026-10-26
