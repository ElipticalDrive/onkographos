# Hierarchical Trilinear Ray Casting

Also called: Adaptive ray casting using trilinear basis functions, Hierarchical Data Structures for Ray Casting, Hierarchical Ray Casting with Trilinear Coefficients, Hierarchically Accelerated Ray Casting with Controlled Error, Ray casting with mdh tree

## Overview
Hierarchical Trilinear Ray Casting is a volume rendering technique that accelerates ray casting by traversing a multi-dimensional hierarchy (mdh) to adaptively choose the resolution level for evaluating volume data [S2, unofficial]. It solves the problem of expensive ray traversal in large volumes by approximating homogeneous regions with least-squares trilinear coefficients, allowing the renderer to skip fine-grained sampling in unimportant or smooth areas [S2, unofficial]. Unlike sibling methods that rely on simple opacity thresholds or constant-value homogeneity, this method uses a user-defined error tolerance and data importance weighting to control refinement, providing a balance between image quality and performance [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin and direction; a hierarchical data structure (mdh tree) where each node stores trilinear coefficients, nodal error, and importance; a user-defined error tolerance; a data importance weighting function (transfer function).
*   **Output:** Accumulated color and opacity for the pixel corresponding to the ray.

**Data Structures:**
The hierarchy uses a BON tree structure [S2, unofficial]. Each node contains:
1.  **Data Model:** Trilinear coefficients representing a least-squares fit to the data in the region covered by the node. This provides $C^0$ continuity at shared faces [S2, unofficial].
2.  **Evaluation Information:**
    *   **Nodal Error:** The average deviation of the node's trilinear model from the actual data within the region [S2, unofficial].
    *   **Importance:** A user-defined value derived from a transfer function, indicating the significance of data values in that range [S2, unofficial].

**Algorithm Steps:**
1.  **Initialization:** For each pixel, cast a ray through the volume. Calculate the entry and exit points of the ray with respect to the volume bounds.
2.  **Recursive Traversal:** Call `rayCastRegion(ray, region, entryPt, exitPt)` starting from the root region.
3.  **Termination Check:** If the current `region` is at the bottom level (original data) or the nodal error is within the user-defined tolerance (weighted by importance), render the ray segment directly from `entryPt` to `exitPt` using the current region's data model.
4.  **Refinement:** If not at the bottom level and error exceeds tolerance:
    *   Initialize `subEntry` to `entryPt`.
    *   Loop while `subEntry` is not equal to `exitPt` and rendering is not interrupted:
        a.  **Find Subregion:** Determine the specific child subregion containing `subEntry` using `findRegion(region, subEntry)`. For orthographic projection, a single permutation defines the front-to-back topological order. For perspective, the permutation is fixed per ray but may vary between rays [S2, unofficial].
        b.  **Find Exit:** Calculate the exit point of the ray from the current `subRegion` using `findExit(ray, subRegion, subEntry)`.
        c.  **Recurse:** Call `rayCastRegion(ray, subRegion, subEntry, subExit)`.
        d.  **Advance:** Update `subEntry` to `subExit`.
5.  **Direct Rendering (Leaf or Tolerated Node):**
    *   If stopping at a node higher than the bottom level: Evaluate data, color, and opacity at the intersection points of the region's bounding box using the node's trilinear coefficients [S2, unofficial].
    *   If going down to the data level: Look up the eight corner data values of the cell/voxel and compute colors and opacity at the intersection points of the cell boundary [S2, unofficial].
    *   Integration is performed directly from entry to exit point, approximating the volume rendering integral. The method uses an approximation similar to coherent projection, evaluating the "emission" contribution at the thickest point of the cell's projection [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_cast_region(ray, region, entry_pt, exit_pt):
    """
    Recursively compute the contribution of a ray through a specified region.
    """
    # Check if we should stop refining
    if region.is_bottom_level or region.error_within_tolerance():
        render_ray_direct(ray, entry_pt, exit_pt, region)
        return

    sub_entry = entry_pt
    while sub_entry != exit_pt and not interrupted():
        # Find the child subregion containing the current ray position
        sub_region = find_region(region, sub_entry)
        
        # Calculate where the ray exits this subregion
        sub_exit = find_exit(ray, sub_region, sub_entry)
        
        # Recursively process the subregion
        ray_cast_region(ray, sub_region, sub_entry, sub_exit)
        
        # Advance to the next segment
        sub_entry = sub_exit

def find_exit(ray, sub_region, sub_entry):
    """
    Calculate the exit point of the ray from the subregion.
    Uses a generalization of the Snyder and Barr method.
    """
    # Maintain ray parameters tx, ty, tz for imminent intersections with grid planes
    # In the adaptive case, updates are nondestructive
    # Update amount is wx * dtx (if tx selected), where wx is the width in grid points
    # Return the point corresponding to the minimum intersection parameter
    pass

def render_ray_direct(ray, entry_pt, exit_pt, region):
    """
    Render the ray segment directly using the region's data model.
    """
    if region.is_bottom_level:
        # Use 8 corner data values of the cell/voxel
        data_values = get_corner_data(region, entry_pt, exit_pt)
        compute_color_opacity(data_values, ray, entry_pt, exit_pt)
    else:
        # Use trilinear coefficients of the node
        coefficients = region.get_trilinear_coefficients()
        compute_color_opacity_from_coefficients(coefficients, ray, entry_pt, exit_pt)
```

**Parameters:**
*   **Error Tolerance:** A user-defined parameter controlling the maximum allowed error in the image. Higher tolerance allows rendering at coarser levels, increasing speed [S2, unofficial].
*   **Data Importance Weighting:** A user-supplied function that weights the error metric. It allows small errors in critical data ranges to be considered serious, while substantial errors are acceptable in uninteresting ranges (e.g., air) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Gradient Discontinuity:** The gradient calculated using the derivative of the trilinear function may not be continuous at shared cell faces, introducing potential error [S2, unofficial].
*   **Perspective Permutations:** For perspective projection, rays may require varying permutations for subregion traversal, though the permutation is fixed within an individual ray [S2, unofficial].

## Subtopics

### [Adaptive Volume Ray Casting](8347-adaptive-volume-ray-casting.md)
Adaptive Volume Ray Casting is a technique that builds on Hierarchical Trilinear Ray Casting by using an octree structure to determine traversal depth based on camera-defined resolution limits. It produces better image quality than splatting by avoiding splatting artifacts but is more computationally intensive.

### Hierarchical Volume Rendering with GPU Ray-Casting
A single-pass volume rendering method compatible with a hierarchical data structure, extending Scharsach's ray-casting method [S3, unofficial]. It performs traversal and color accumulation in a fragment shader for each pixel using a quad covering the display surface [S3, unofficial]. The vertex shader calculates ray position and direction in data space, which are interpolated to the fragment shader [S3, unofficial]. Entry/exit points are calculated, and rays not intersecting the volume terminate early [S3, unofficial]. Tree traversal alternates horizontal (within node grid) and vertical (between hierarchy levels) passes, using a mini-stack or stack-less algorithm [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the error tolerance and data importance. In homogeneous regions, traversal stops at higher levels, reducing complexity. In complex regions, it approaches regular grid ray casting.
*   **Memory Complexity:** Requires storage for the hierarchical tree (mdh) with trilinear coefficients and error/importance metadata for each node.
*   **Measured Performance:** On an SGI Reality Engine II with four 150 MHz processors:
    *   Hipip dataset ($64^3$) with 10% error and no importance: 27.0 seconds (hierarchical) vs. 97.0 seconds (regular grid) [S2, unofficial].
    *   CT Head dataset ($256 \times 256 \times 113$) with 1% error and importance: 56.0 seconds (hierarchical) vs. 205.0 seconds (regular grid) [S2, unofficial].
*   **Strengths:** Provides error-controlled rendering; handles perspective and orthographic projections; easily parallelizable (each ray is nearly independent) [S2, unofficial].
*   **Limitations:** Noticeably slower than ray casting a non-hierarchical rectilinear volume; slower than coherent projection or hardware 3D textures [S2, unofficial].
*   **When to Use:** When image quality must be controlled by error tolerance and importance weighting, and when volumes have large homogeneous or unimportant regions [S2, unofficial].
*   **When Not to Use:** When maximum speed is required and error control is not critical; when hardware 3D textures or coherent projection are available and sufficient [S2, unofficial].

## Background
Hierarchical Trilinear Ray Casting was developed by Gelder, Kim, and Wilhelms at the University of California, Santa Cruz [S2, unofficial]. It was described in the technical report "Hierarchically Accelerated Ray Casting for Volume Rendering with Controlled Error" (UCSC-CRL-95-31) in March 1995 [S2, unofficial]. The method generalizes previous work on homogeneity (which relied on nearly constant values) to least-squares fits with trilinear basis functions [S2, unofficial]. It was implemented within a general system for multi-dimensional trees (mdh) [S2, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S2] [PDF] Hierarchically Accelerated Ray Casting for Volume Rendering with ..., https://tr.soe.ucsc.edu/sites/default/files/technical-reports/UCSC-CRL-95-31.pdf, fetched 2026-10-27 (unofficial)
[S3] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
