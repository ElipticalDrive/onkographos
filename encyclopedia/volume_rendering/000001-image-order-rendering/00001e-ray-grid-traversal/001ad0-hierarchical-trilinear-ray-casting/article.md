# Hierarchical Trilinear Ray Casting

Also called: Adaptive ray casting using trilinear basis functions, Hierarchical Data Structures for Ray Casting, Hierarchical Ray Casting with Trilinear Coefficients, Hierarchically Accelerated Ray Casting with Controlled Error, Ray casting with mdh tree

## Overview
Hierarchical Trilinear Ray Casting accelerates volume rendering by traversing a multi-dimensional hierarchy (mdh) to adaptively select the resolution level for evaluating volume data, based on a user-defined error tolerance and data importance weighting [S2, unofficial]. Unlike standard ray-grid traversal methods that step at fixed intervals or use simple homogeneity checks, this method approximates data regions using least-squares trilinear coefficients to decide whether to refine the ray traversal or evaluate the data directly [S2, unofficial]. It generalizes the concept of homogeneity from nearly constant values to least-squares fits with a set of basis functions, specifically trilinear functions, allowing for large steps over homogeneous regions while maintaining high fidelity in critical areas [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; a hierarchical data structure (mdh tree) where each node contains trilinear coefficients, nodal error, and importance values; a user-defined error tolerance $\epsilon$; and a data importance weighting function $I(v)$ [S2, unofficial].
*   **Outputs:** The accumulated color and opacity for the ray, computed by integrating from the entry point to the exit point of the volume [S2, unofficial].

**Data Structures:**
The hierarchy uses a BON tree structure [S2, unofficial]. Each node in the tree contains:
1.  **Data Model:** Trilinear coefficients representing the best fit to the trilinear functions of the cells covered by the node. This provides $C^0$ continuity at shared faces [S2, unofficial].
2.  **Evaluation Information:**
    *   **Nodal Error:** The average deviation of the data model stored with the node from the actual data within the region covered [S2, unofficial].
    *   **Importance:** A user-defined transfer function value describing the importance of data values within the volume. Regions with low importance (e.g., air) allow for larger errors [S2, unofficial].

**Algorithm Steps:**
The core traversal is performed by the recursive function `rayCastRegion(ray, region, entryPt, exitPt)` [S2, unofficial]:

1.  **Termination Check:** If the current `region` is at the bottom level (leaf node containing original data) OR the nodal error (weighted by importance) is within the user-defined tolerance $\epsilon$:
    *   Render the ray segment directly from `entryPt` to `exitPt` using `renderRay(ray, entryPt, exitPt)` [S2, unofficial].
    *   *Note:* If stopping at a node higher than the bottom level, evaluate data, color, and opacity at the intersection points of the region's bounding box using the node's trilinear coefficients [S2, unofficial]. If at the data level, look up the eight corner data values of the cell/voxel and compute colors/opacity at the intersection points of the cell boundary [S2, unofficial].
2.  **Recursive Refinement:** If not at the bottom level and error exceeds tolerance:
    *   Initialize `subEntry` to `entryPt` [S2, unofficial].
    *   Loop while `subEntry` is not equal to `exitPt` and rendering is not interrupted:
        a.  **Find Subregion:** Identify the specific child subregion containing `subEntry` using `findRegion(region, subEntry)` [S2, unofficial].
            *   *Optimization:* For orthographic projection, a single permutation provides a front-to-back topological order for all subregions, making `findRegion` simple and efficient. For perspective projection, rays may require varying permutations, but the permutation is fixed within an individual ray [S2, unofficial].
        b.  **Find Exit:** Calculate the exit point of the ray from the current `subRegion` using `findExit(ray, subRegion, subEntry)` [S2, unofficial].
        c.  **Recurse:** Recursively call `rayCastRegion` for the identified `subRegion`, passing `subEntry` and the calculated `subExit` [S2, unofficial].
        d.  **Advance:** Update `subEntry` to `subExit` to advance the ray to the next segment [S2, unofficial].

**Integration and Sampling:**
Instead of regular sampling, the method integrates directly from `entryPt` to `exitPt`, approximately solving the underlying differential equation [S2, unofficial]. The approximation uses the same method as coherent projection: finding the color at the thickest point of a cell's projection to determine the "emission" contribution [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_cast_region(ray, region, entry_pt, exit_pt, error_tolerance, importance_func):
    """
    Recursively cast a ray through a hierarchical region.
    
    Args:
        ray: Ray object with origin and direction.
        region: Current node in the mdh tree.
        entry_pt: Entry point of the ray into the region.
        exit_pt: Exit point of the ray from the region.
        error_tolerance: User-defined maximum allowed error.
        importance_func: Function mapping data value to importance weight.
        
    Returns:
        Accumulated color and opacity.
    """
    # Check if we can stop refinement
    # Nodal error is precomputed and stored in region
    # Importance is evaluated at the region's data values
    weighted_error = region.nodal_error * importance_func(region.data_model)
    
    if region.is_bottom_level or weighted_error <= error_tolerance:
        # Render directly using trilinear coefficients or leaf data
        return render_ray_direct(ray, entry_pt, exit_pt, region)
    else:
        # Refine: traverse children
        sub_entry = entry_pt
        accumulated_color = (0, 0, 0, 0)
        
        while sub_entry != exit_pt and not interrupted():
            # Find which child subregion contains sub_entry
            sub_region = find_region(region, sub_entry)
            
            # Find where the ray exits this subregion
            sub_exit = find_exit(ray, sub_region, sub_entry)
            
            # Recursively process the subregion
            color, opacity = ray_cast_region(ray, sub_region, sub_entry, sub_exit, error_tolerance, importance_func)
            
            # Accumulate result (front-to-back compositing)
            accumulated_color = composite(accumulated_color, color, opacity)
            
            # Advance to the next segment
            sub_entry = sub_exit
            
        return accumulated_color

def find_exit(ray, sub_region, sub_entry):
    """
    Generalization of Snyder and Barr method for regularly tessellated grids.
    Maintains ray parameters tx, ty, tz for imminent intersections with grid planes.
    """
    # Initialize tx, ty, tz based on sub_entry and sub_region bounds
    tx, ty, tz = initialize_ray_params(ray, sub_entry, sub_region)
    
    # Determine the next intersection
    # In uniform case, update is fixed amount (e.g., tx += a * dtx)
    # In adaptive case, updates are nondestructive and scaled by subregion width wx
    # wx is the width (in grid points) of the subregion being jumped over
    
    min_param = min(tx, ty, tz)
    axis = argmin(tx, ty, tz)
    
    if axis == x:
        # Update tx by a * dtx * wx
        tx += a * dtx * sub_region.width_x
    elif axis == y:
        ty += a * dty * sub_region.width_y
    else:
        tz += a * dtz * sub_region.width_z
        
    # Recalculate min_param if necessary or return the point corresponding to min_param
    return get_point_on_ray(ray, min_param)

def render_ray_direct(ray, entry_pt, exit_pt, region):
    """
    Integrate directly from entry to exit using trilinear approximation.
    Uses coherent projection approximation: find color at thickest point of cell's projection.
    """
    # If region is bottom level, use 8 corner data values
    # If region is internal node, use trilinear coefficients
    if region.is_bottom_level:
        data_values = region.get_corner_data()
        color, opacity = compute_color_opacity_from_corners(ray, entry_pt, exit_pt, data_values)
    else:
        coeffs = region.trilinear_coefficients
        color, opacity = compute_color_opacity_from_trilinear(ray, entry_pt, exit_pt, coeffs)
        
    return color, opacity
```

**Edge Cases and Pitfalls:**
*   **Gradient Discontinuity:** The gradient calculated using the derivative of the trilinear function may not be continuous at shared cell faces, introducing potential error [S2, unofficial]. An alternative is to use finite differences for the gradient within the data, but this is not the default [S2, unofficial].
*   **Perspective Permutations:** For perspective projection, the topological order of subregions may vary between rays, requiring dynamic permutation calculation, though it remains fixed for a single ray [S2, unofficial].

## Subtopics

### [Adaptive Volume Ray Casting](8347-adaptive-volume-ray-casting.md)
A volume rendering technique where one ray is shot through the volume for each pixel, traversing an octree to accumulate physical values along the ray path. It is a specific application of hierarchical ray casting that focuses on the adaptive traversal of an octree structure to balance performance and accuracy based on data homogeneity.

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** The method is noticeably slower than ray casting a non-hierarchical rectilinear volume due to the overhead of tree traversal and coefficient evaluation [S2, unofficial]. However, it is usually faster than regular grid ray casting on volumes where importance functions can limit traversal [S2, unofficial].
*   **Memory:** The BON tree structure provides a space-saving strategy for storing the hierarchy [S2, unofficial].

**Measured Performance:**
On an SGI Reality Engine II with four 150 MHz processors:
*   **Hipip dataset ($64^3$):** With 10% error and no importance weighting, hierarchical ray casting took 27.0 seconds, compared to 97.0 seconds for regular grid ray casting [S2, unofficial].
*   **CT Head dataset ($256 \times 256 \times 113$):** With 1% error and importance weighting, hierarchical ray casting took 56.0 seconds, compared to 205.0 seconds for regular grid ray casting [S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Provides error-controlled rendering; allows user-defined importance weighting to prioritize critical regions; easily parallelizable as each ray's computations are nearly independent [S2, unofficial].
*   **Limitations:** Slower than non-hierarchical methods; gradient discontinuities at shared faces may introduce artifacts; requires precomputation of trilinear coefficients and nodal errors [S2, unofficial].

## Background
Hierarchical Trilinear Ray Casting was developed by Gelder, Kim, and Wilhelms at the University of California, Santa Cruz [S2, unofficial]. The method was introduced in the technical report "Hierarchically Accelerated Ray Casting for Volume Rendering with Controlled Error" (UCSC-CRL-95-31) in March 1995 [S2, unofficial]. It builds upon earlier work on multi-dimensional hierarchical systems (mdh) [WVG91, WVG92, WVG94] and extends previous ray-casting acceleration techniques that relied on regular sampling or simple homogeneity checks [S2, unofficial]. The method is part of a broader family of hierarchical volume rendering techniques that use data structures like octrees, K-d trees, or pyramids to accelerate ray intersection finding [S1, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S2] Hierarchically Accelerated Ray Casting for Volume Rendering with Controlled Error, https://tr.soe.ucsc.edu/sites/default/files/technical-reports/UCSC-CRL-95-31.pdf, fetched 2026-10-27 (unofficial)
[S3] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
