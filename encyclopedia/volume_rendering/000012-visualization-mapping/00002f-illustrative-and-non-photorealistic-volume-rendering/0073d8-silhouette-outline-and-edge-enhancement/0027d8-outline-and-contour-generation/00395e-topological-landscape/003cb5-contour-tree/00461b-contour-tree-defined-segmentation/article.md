# Contour-Tree-Defined Segmentation
Also called: Topological Zone Segmentation

## Overview
Contour-Tree-Defined Segmentation partitions a scalar volume into topological zones, where each zone corresponds to an arc in the contour tree of the underlying trilinear interpolant [S1]. It solves the problem of assigning independent transfer functions to distinct topological structures, allowing for the isolation of specific features based on their creation and destruction isovalue ranges [S1]. It builds on the concept of topological zones defined by Cox et al. [S1] and extends prior work by allowing independent transfer functions per zone and using trilinear interpolation over hexahedral cells [S1].

## How it works
**Inputs:** A scalar field defined on a regular grid of hexahedral cells, typically with trilinear interpolation [S1].
**Outputs:** A segmentation of the volume into topological zones, each associated with a specific transfer function, and the final rendered image [S1].

**Data Structures:**
1. **Grid:** Hexahedral cells with scalar values at vertices [S1].
2. **Widget:** A local graph structure within each cube [S1]. Edges include the cube's mesh edges, plus augmented edges connecting body saddles, face saddles, and vertices [S1].
3. **Contour Tree:** A graph where nodes represent critical points (minima, maxima, saddles) and arcs represent intervals between critical values [S1].
4. **Simplified Tree:** The contour tree after simplification to remove noise (symbolic perturbation) [S1].
5. **Branch Decomposition:** A hierarchical representation of the simplified contour tree to manage complexity and avoid oversegmentation [S1].
6. **Transfer Functions:** A mapping from each topological zone (arc in the simplified tree) to opacity/color parameters [S1].

**Steps:**

1. **Compute the Contour Tree:** Compute the contour tree of the trilinear interpolant using a variation of the method by Pascucci and Cole-McLaughlin [S1]. Construct the widget in each cube as described above [S1]. Apply the graph-theoretic algorithm to the union of cellwise join and split trees derived from these widgets [S1]. Any graph accurately representing the topology of the trilinear interpolant may be substituted [S1].
2. **Simplify the Contour Tree:** Apply topological simplification to the computed contour tree to remove symbolic perturbation (noise) [S1]. This step merges insignificant arcs, reducing the number of topological zones [S1].
3. **Assign Transfer Functions:** Assign a specific transfer function to each remaining arc (topological zone) in the simplified contour tree [S1]. The hierarchy is stored using branch decomposition to allow coarse interactions and propagation of transfer function changes [S1]. Branch decomposition concatenates sequences of arcs into single units if their regions do not overlap in value range, preventing oversegmentation [S1].
4. **Volume Rendering with Zone-Specific Opacity:** For each ray cast through the volume, determine which topological zone the current sample point belongs to [S1]. Look up the opacity directly from the transfer function assigned to that specific zone [S1]. Apply the trilinear interpolant over the hexahedral cells to determine the scalar value at the sample point [S1]. For cells spanning multiple topological zones, the method calculates the correct opacity by identifying the specific zone active at the sample point's scalar value, rather than interpolating opacities across tetrahedra [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def contour_tree_segmentation_render(grid, simplified_tree, transfer_functions_map, ray_origin, ray_direction, step_size, max_steps):
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_opacity = 0.0
    t = 0.0
    while t < max_steps * step_size:
        pos = [ray_origin[i] + t * ray_direction[i] for i in range(3)]
        scalar_value = trilinear_interpolate(grid, pos) # [G] Standard trilinear interpolation over the containing hexahedral cell
        zone_id = get_zone_id_for_scalar(simplified_tree, scalar_value)
        tf = transfer_functions_map[zone_id]
        opacity = tf.get_opacity(scalar_value)
        color = tf.get_color(scalar_value)
        alpha_contribution = opacity * (1.0 - accumulated_opacity)
        accumulated_color = [accumulated_color[i] + color[i] * alpha_contribution for i in range(3)]
        accumulated_opacity += alpha_contribution
        if accumulated_opacity >= 1.0:
            break
        t += step_size
    return accumulated_color

def get_zone_id_for_scalar(tree, scalar_value):
    # Each arc spans from a creation critical value to a destruction critical value [S1]
    for arc in tree.arcs:
        if arc.min_critical_value <= scalar_value <= arc.max_critical_value:
            return arc.zone_id
    return tree.default_zone_id # [G] Fallback for out-of-bounds values
```

**Edge Cases and Pitfalls:**
*   **Symbolic Perturbation:** Without simplification, noise creates excessive topological zones, leading to oversegmentation and performance issues [S1].
*   **Cells Spanning Multiple Zones:** If a hexahedral cell contains critical points, it spans multiple topological zones [S1]. The method must correctly identify which zone is active at the specific sample point's scalar value [S1].
*   **Trilinear Interpolation:** The method relies on the topology of the trilinear interpolant, not a tetrahedral decomposition [S1]. Using tetrahedral interpolation for opacity lookup is explicitly avoided as it causes artifacts [S1].

## Subtopics

### [Contour-Based Segmentation for FT Specification](11723-contour-based-segmentation-for-ft-specification.md)
Uses user-drawn 2D contours to isolate volume portions before transfer function specification. It differs by replacing topological tree traversal with direct geometric separation via sketches.

## Operating characteristics
*   **Complexity:** The complexity is dominated by the contour tree construction and simplification [S1]. For large datasets, branch decomposition is used to manage the hierarchy, allowing coarse interactions [S1].
*   **Performance:** The method avoids the artifacts associated with tetrahedral interpolation of opacities [S1]. It allows independent transfer functions for individual zones, which can increase the complexity of the rendering pipeline but provides finer control [S1].
*   **Strengths:** Topologically distinct regions are segmented correctly [S1]. It handles the trilinear interpolant directly, avoiding tetrahedral artifacts [S1]. Branch decomposition helps manage complexity and prevents oversegmentation [S1].
*   **Limitations:** Requires computation of the contour tree, which can be expensive for very large datasets without simplification [S1].
*   **When to use:** When isolating specific topological features based on their creation and destruction isovalue ranges is required [S1].
*   **When not to use:** Not given in the sources. [G]

## Background
The method was introduced in "Topology-Controlled Volume Rendering" [S1]. It builds on the concept of topological zones defined by Cox et al. [S1] and extends the work of Takeshima et al. [S1] by allowing independent transfer functions for individual zones and using trilinear interpolation over hexahedral cells [S1]. The contour tree computation uses variations of methods by Pascucci and Cole-McLaughlin [S1], and the hierarchical representation uses branch decomposition by Pascucci et al. [S1].

## Sources
[S1] Topology-Controlled Volume Rendering, https://doi.org/10.1109/tvcg.2007.47, fetched 2026-10-27
