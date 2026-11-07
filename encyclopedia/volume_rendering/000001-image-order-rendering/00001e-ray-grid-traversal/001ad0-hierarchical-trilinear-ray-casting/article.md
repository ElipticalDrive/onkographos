# Hierarchical Trilinear Ray Casting

Also called: Adaptive ray casting using trilinear basis functions, Hierarchical Data Structures for Ray Casting, Hierarchical Ray Casting with Trilinear Coefficients, Hierarchically Accelerated Ray Casting with Controlled Error, Ray casting with mdh tree

## Overview
Hierarchical Trilinear Ray Casting is a volume rendering method that accelerates ray casting by traversing a multi-dimensional hierarchy (mdh) and adaptively selecting the resolution level for data evaluation based on a user-defined error tolerance and data importance weighting [S2, unofficial]. It differs from sibling methods like [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) or [Hierarchical Voxel Traversal](2204-hierarchical-voxel-traversal.md) by using least-squares trilinear basis functions to model data regions, allowing the algorithm to decide whether to refine traversal or evaluate the region directly based on the approximation error [S2, unofficial]. Unlike methods that rely on simple homogeneity (constant values), this approach generalizes homogeneity to least-squares fits, providing a more robust metric for heterogeneous but smooth regions [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A multi-dimensional hierarchy (mdh) of the volume data, a ray defined by origin and direction, an error tolerance $\epsilon$, and an optional data importance weighting function $I(v)$ [S2, unofficial].
*   **Output:** The accumulated color and opacity for the ray [S2, unofficial].

**Data Structures:**
The hierarchy uses a BON tree structure [S2, unofficial]. Each node contains:
1.  **Data Model:** Trilinear coefficients representing a least-squares fit to the data within the region covered by the node [S2, unofficial].
2.  **Evaluation Information:**
    *   **Nodal Error:** The average deviation of the node's trilinear model from the actual data within the region [S2, unofficial].
    *   **Importance:** A value derived from the user-defined importance function, indicating the criticality of the data values in that region [S2, unofficial].

**Algorithm Steps:**
The core logic is implemented via a recursive function `rayCastRegion(ray, region, entryPt, exitPt)` [S2, unofficial].

1.  **Termination Check:** If the current `region` is at the bottom level (leaf) or the nodal error (weighted by importance) is within the user-defined tolerance $\epsilon$, the ray is rendered directly from `entryPt` to `exitPt` using the region's data model [S2, unofficial].
2.  **Refinement:** If not at the bottom level and error exceeds tolerance:
    a.  Initialize `subEntry` to `entryPt` [S2, unofficial].
    b.  Loop while `subEntry` is not equal to `exitPt` and rendering is not interrupted:
        i.  **Find Subregion:** Determine the specific child `subRegion` containing `subEntry` using `findRegion(region, subEntry)` [S2, unofficial]. For orthographic projection, a single permutation provides a front-to-back topological order for all subregions, making this efficient [S2, unofficial]. For perspective projection, the permutation is fixed per ray but may vary between rays [S2, unofficial].
        ii. **Find Exit:** Calculate the exit point `subExit` of the ray from `subRegion` starting at `subEntry` using `findExit(ray, subRegion, subEntry)` [S2, unofficial].
        iii. **Recurse:** Call `rayCastRegion(ray, subRegion, subEntry, subExit)` [S2, unofficial].
        iv. **Advance:** Update `subEntry` to `subExit` [S2, unofficial].

**Intersection Calculation (`findExit`):**
`findExit` uses a generalization of the Snyder and Barr method for regularly tessellated grids [S2, unofficial].
*   Maintain three ray parameter values $t_x, t_y, t_z$, representing the most imminent intersections with the x, y, and z grid planes of the current subregion [S2, unofficial].
*   Select the minimum of $t_x, t_y, t_z$ as the next intersection parameter [S2, unofficial].
*   **Uniform Case:** Update the selected parameter by a fixed amount (e.g., if $t_x$ is selected, update by $\Delta t_x$) [S2, unofficial].
*   **Adaptive Case:** Updates are nondestructive because the same parameter value (`subEntry`) may be updated at several tree levels [S2, unofficial]. The update amount is multiplied by $w_x$, the width (in grid points) of the subregion being jumped over [S2, unofficial].

**Rendering at Termination:**
When the algorithm stops refinement at a node higher than the bottom level:
*   Evaluate data, color, and opacity at the intersection points of the region's bounding box using the node's trilinear coefficients [S2, unofficial].
*   Integrate directly from `entryPt` to `exitPt` by approximately solving the underlying differential equation. The approximation finds the color at the thickest point of the cell's projection to determine the "emission" contribution, similar to the coherent projection method [S2, unofficial].

If the algorithm goes down to the data level (leaf):
*   Look up the eight corner data values of the cell/voxel [S2, unofficial].
*   Compute colors and opacity at the intersection points of the cell boundary [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_cast_region(ray, region, entry_pt, exit_pt, tolerance, importance_func):
    """Recursively cast a ray through a hierarchical region."""
    # Termination check: bottom level or error within tolerance
    if region.is_leaf() or region.weighted_error(importance_func) <= tolerance:
        return render_region_direct(ray, region, entry_pt, exit_pt)
    
    sub_entry = entry_pt
    # Loop until ray exits the region or is interrupted
    while sub_entry < exit_pt - 1e-6:
        sub_region = find_region(region, sub_entry)
        sub_exit = find_exit(ray, sub_region, sub_entry)
        contribution = ray_cast_region(ray, sub_region, sub_entry, sub_exit, tolerance, importance_func)
        accumulate(contribution)
        sub_entry = sub_exit

def find_exit(ray, region, entry_pt):
    """Calculate exit point using generalized Snyder & Barr traversal."""
    # Initialize intersection parameters for x, y, z planes of the region
    tx, ty, tz = ray.init_intersection_params(region, entry_pt)
    
    # Select minimum parameter as next intersection
    t_next = min(tx, ty, tz)
    
    # Determine selected axis and compute adaptive update amount
    if t_next == tx:
        axis = 'x'
        update = region.width_in_grid_points('x') * ray.grid_plane_step('x')
    elif t_next == ty:
        axis = 'y'
        update = region.width_in_grid_points('y') * ray.grid_plane_step('y')
    else:
        axis = 'z'
        update = region.width_in_grid_points('z') * ray.grid_plane_step('z')
        
    # Nondestructive update of the selected parameter
    if axis == 'x': tx += update
    elif axis == 'y': ty += update
    else: tz += update
        
    return ray.point_at(entry_pt, t_next)

def render_region_direct(ray, region, entry_pt, exit_pt):
    """Render region directly using its data model."""
    if region.is_leaf():
        # Look up eight corner data values
        data_values = region.get_leaf_data()
        color, opacity = compute_color_opacity_leaf(data_values, ray, entry_pt, exit_pt)
    else:
        # Use trilinear coefficients
        coeffs = region.get_trilinear_coeffs()
        color, opacity = compute_color_opacity_trilinear(coeffs, ray, entry_pt, exit_pt)
    return color, opacity

def find_region(region, point):
    """Determine child subregion containing point using topological permutation."""
    # Returns the appropriate child node index based on point coordinates
    return region.children[region.permutation_index(point)]

def accumulate(contribution):
    """Add contribution to global ray accumulator."""
    global_accumulator.color += contribution.color * contribution.opacity
    global_accumulator.opacity += contribution.opacity * (1.0 - global_accumulator.opacity)

def compute_color_opacity_leaf(data_values, ray, entry_pt, exit_pt):
    """Compute color/opacity from 8 corner values at cell boundary intersections."""
    # Standard trilinear interpolation of corner values at entry/exit points
    return ray.sample_volume(data_values, entry_pt), ray.sample_volume(data_values, exit_pt)

def compute_color_opacity_trilinear(coeffs, ray, entry_pt, exit_pt):
    """Compute color/opacity from trilinear coefficients at bounding box intersections."""
    return ray.sample_trilinear(coeffs, entry_pt), ray.sample_trilinear(coeffs, exit_pt)

def interrupted():
    """Check for early termination flags (e.g., max opacity reached)."""
    return global_accumulator.opacity >= 1.0
```

**Parameters:**
*   **Error Tolerance ($\epsilon$):** A user-defined parameter controlling the maximum allowed error. Larger values allow rendering at higher tree levels (coarser resolution), increasing speed but reducing quality [S2, unofficial].
*   **Data Importance Weighting Function:** A user-supplied function that weights the error metric. It allows small errors in critical data ranges to be considered serious, while substantial errors in uninteresting ranges (e.g., air around medical data) are acceptable [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Gradient Discontinuity:** The gradient calculated using the derivative of the trilinear function may not be continuous at shared cell faces, introducing potential error in shading calculations [S2, unofficial].
*   **Perspective Permutations:** For perspective projection, the topological order of subregions may vary between rays, requiring careful handling in `findRegion` [S2, unofficial].

## Subtopics

### Hierarchical Volume Rendering with GPU Ray-Casting
A single-pass volume rendering method compatible with a hierarchical data structure, extending Scharsach's ray-casting method [S3, unofficial]. It performs traversal and color accumulation in a fragment shader for each pixel, using a quad covering the display surface [S3, unofficial]. The vertex shader calculates ray position and direction in data space for each vertex of the display quad [S3, unofficial]. Rasterization interpolates these rays to the fragment shader [S3, unofficial]. For each ray, entry and exit points within the data volume bounds are calculated, handling cases where the ray starts outside or exits early due to max length [S3, unofficial]. If the ray does not intersect the volume, shader execution stops immediately, replacing early depth testing [S3, unofficial]. Tree traversal alternates horizontal passes (within node grid) and vertical passes (between hierarchy levels), using either a mini-stack or stack-less algorithm [S3, unofficial]. Samples are accumulated according to the volume rendering integral [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** The method is adaptive. In homogeneous regions, it renders at higher tree levels, significantly reducing the number of samples. In heterogeneous regions, it refines to the leaf level [S2, unofficial].
*   **Memory Complexity:** Depends on the BON tree structure, which is space-saving compared to full resolution storage [S2, unofficial].
*   **Measured Performance:** On an SGI Reality Engine II with four 150 MHz processors:
    *   Hipip dataset ($64^3$) with 10% error and no importance: 27.0 seconds (hierarchical) vs. 97.0 seconds (regular grid) [S2, unofficial].
    *   CT Head dataset ($256 \times 256 \times 113$) with 1% error and importance: 56.0 seconds (hierarchical) vs. 205.0 seconds (regular grid) [S2, unofficial].
*   **Strengths:** Provides error-controlled rendering; flexible importance weighting; easily parallelizable (each ray is nearly independent); supports perspective projection without significant penalty [S2, unofficial].
*   **Limitations:** Noticeably slower than ray casting a non-hierarchical rectilinear volume; slower than coherent projection or hardware 3D textures for interactive speeds; gradient discontinuities may affect shading quality [S2, unofficial].
*   **When to Use:** When image quality must be controlled by error tolerance and importance weighting, and when the volume has regions that can be approximated well by trilinear functions [S2, unofficial].
*   **When Not to Use:** When maximum performance is required and error control is not critical; when the volume data is highly discontinuous and trilinear approximation is poor [S2, unofficial].

## Background
The method was developed by Gelder, Kim, and Wilhelms at the University of California, Santa Cruz, and published in March 1995 [S2, unofficial]. It builds on previous work in multi-dimensional trees (mdh) and coherent projection [S2, unofficial]. The primary motivation was to provide a reference image for quantitative image comparisons with faster methods, offering a flexible interface for controlling image parameters [S2, unofficial]. It generalizes the concept of homogeneity from nearly constant values (used in works by Levoy, Danskin/Hanrahan, Westermann) to least-squares fits with trilinear basis functions [S2, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S2] Hierarchically Accelerated Ray Casting for Volume Rendering with ..., https://tr.soe.ucsc.edu/sites/default/files/technical-reports/UCSC-CRL-95-31.pdf, fetched 2026-10-27 (unofficial)
[S3] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
