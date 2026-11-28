# Bisection Isosurface Extraction

Also called: BEM Isosurface Computation Implementation, Isovalue Stepping, RayEvaluationSteppingIso

## Overview
Bisection Isosurface Extraction is a meshfree method for locating isosurfaces within scalar fields by performing a line search using the bisection algorithm [S1]. It sets itself apart from sibling methods by decoupling the field evaluation from the meshing process, using an octree to identify relevant regions and applying bisection only along specific search lines to find intersection points, thereby significantly reducing computational costs compared to volume-meshed approaches [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A scalar field function (e.g., electric potential) computable at arbitrary points; an octree structure defining the spatial domain (inner and outer octrees); a target isovalue.
*   **Output:** A set of points on the isosurface, which are subsequently meshed into triangular elements [S1].

**Data Structures**
*   **Octree:** A hierarchical structure dividing space into cubes. The implementation uses an inner octree and an outer octree. The outer octree adds one additional layer of cubes to the inner octree to account for field values outside the bounding box of the object [S1].
*   **Relevant Cubes:** A subset of octree cubes selected for detailed computation. In the reference implementation, 28 cubes from the inner octree and 16 from the outer octree were selected, representing 1.3% of the total cubes [S1].
*   **Search Line:** A line segment along which the bisection is performed to find the isovalue [S1].

**Parameters**
*   **Convergence Error:** The bisection stops when the interval width or function value difference is below a threshold. The source notes an average of 20 iterations are needed to find the point with a defined isovalue and relatively small error [S1].
*   **Solver Settings (for field computation):** GMRES with Jacobi preconditioner, convergence error < $10^{-6}$, achieved in 56 iteration steps [S1].
*   **Matrix Compression:** Fast Multipole Method (FMM) with $L = 12$ [S1].

**Steps**
1.  **Construct Octree:** Build an inner octree and an outer octree (with one additional layer) covering the domain [S1].
2.  **Determine Data Range:** Calculate the range of the scalar field for each cube in the octree [S1].
3.  **Select Relevant Cubes:** Identify cubes where the isovalue lies within the data range. Select these cubes for detailed computation [S1].
4.  **Define Search Lines:** Define search lines within or crossing the relevant cubes [S1].
5.  **Bisection Line Search:** For each search line, apply the bisection method to find the point where the scalar field equals the target isovalue.
    *   Let $f(x)$ be the scalar field value at point $x$.
    *   Let $v_{target}$ be the target isovalue.
    *   Initialize interval $[a, b]$ on the line such that $f(a) - v_{target}$ and $f(b) - v_{target}$ have opposite signs.
    *   Iterate:
        1.  $c = (a + b) / 2$
        2.  If $|f(c) - v_{target}| < \epsilon$ or $|b - a| < \delta$, stop.
        3.  If $(f(a) - v_{target}) \cdot (f(c) - v_{target}) < 0$, set $b = c$.
        4.  Else, set $a = c$.
    *   The source notes an average of 20 iterations are required [S1].
6.  **Collect Points:** Gather the intersection points found via bisection [S1].
7.  **Meshing:** Merge the isosurface segments into a common element container. Renumber element nodes [S1].
8.  **Output:** Transfer the element set to visualization software [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bisection_isosurface_extraction(field_function, target_isovalue, octree, search_lines):
    """
    Extracts isosurface points using bisection along search lines.
    
    Args:
        field_function: Callable that returns scalar value at (x, y, z).
        target_isovalue: The scalar value defining the isosurface.
        octree: Octree structure with inner and outer layers.
        search_lines: List of line segments (start, end) for bisection.
        
    Returns:
        List of 3D points on the isosurface.
    """
    isosurface_points = []
    
    # Step 1-3: Select relevant cubes based on octree data range
    # Note: In the source, this selection is done by determining data range per cube
    # and selecting 28 inner + 16 outer cubes.
    relevant_cubes = select_relevant_cubes(octree, target_isovalue)
    
    # Step 4-5: Perform bisection on search lines
    for line in search_lines:
        start_point = line.start
        end_point = line.end
        
        # Evaluate endpoints
        val_start = field_function(start_point)
        val_end = field_function(end_point)
        
        # Check if isovalue exists in this interval
        if (val_start - target_isovalue) * (val_end - target_isovalue) > 0:
            continue
            
        # Bisection loop
        a = start_point
        b = end_point
        val_a = val_start
        val_b = val_end
        
        # Average 20 iterations as per source [S1]
        for _ in range(20):
            c = (a + b) / 2.0
            val_c = field_function(c)
            
            if abs(val_c - target_isovalue) < 1e-6: # Convergence threshold [S1]
                isosurface_points.append(c)
                break
                
            if (val_a - target_isovalue) * (val_c - target_isovalue) < 0:
                b = c
                val_b = val_c
            else:
                a = c
                val_a = val_c
                
    # Step 6-8: Meshing and output
    # Merge segments, renumber nodes, and return points
    return isosurface_points

def select_relevant_cubes(octree, target_isovalue):
    """
    Selects cubes where the scalar field range includes the target isovalue.
    """
    relevant = []
    for cube in octree.all_cubes:
        min_val, max_val = cube.data_range
        if min_val <= target_isovalue <= max_val:
            relevant.append(cube)
    return relevant
```

**Edge Cases and Pitfalls**
*   **Boundary Conditions:** The outer octree layer is necessary to capture field values outside the bounding box, preventing errors in isosurface location near boundaries [S1].
*   **Precision:** The method relies on the accuracy of the scalar field function. In the source, the field was computed using a BEM solver with GMRES convergence error $< 10^{-6}$ [S1].
*   **Refinement:** Without refinement, the octree structure may be visible in the isosurface, leading to non-smooth results [S1].

## Subtopics

### Surface Location via Segment Endpoint Values
Determines surface position based on scalar values at segment endpoints without iterative bisection. Choose when computational resources are extremely limited and lower precision is acceptable. [Surface Location via Segment Endpoint Values](3756-surface-location-via-segment-endpoint-values.md)

### Isocontour Shading with Curvature
Focuses on shading techniques for isosurfaces, potentially using curvature information. Choose when visual quality and shading accuracy are prioritized over extraction speed. [Isocontour Shading with Curvature](3819-isocontour-shading-with-curvature.md)

### Multi-Stage Rasterization for Shaded Isosurfaces
Uses rasterization techniques for isosurface rendering. Choose when GPU acceleration is available and rasterization pipelines are preferred over ray-based methods. [Multi-Stage Rasterization for Shaded Isosurfaces](5394-multi-stage-rasterization-for-shaded-isosurfaces.md)

### IsoSurface Ray Marching
Advances along rays to find isosurface intersections. Choose when a simpler, step-based approach is sufficient and bisection overhead is undesirable. [IsoSurface Ray Marching](11315-isosurface-ray-marching.md)

## Operating characteristics
*   **Time Complexity:** The method reduces computational costs by performing detailed computations in only 1.3% of the cubes [S1]. In the reference example, 29,984 potential computations were required for a coarse isosurface, taking ~10 minutes on a single core [S1].
*   **Memory Complexity:** Uses an octree structure. The inner octree had 3,209 cubes and the outer had 65 cubes [S1].
*   **Performance:** The source reports 29,984 computations for a coarse isosurface (168 nodes, 80 elements) taking ~10 minutes on a single core Intel Xeon CPU E3-1275v5 [S1].
*   **Strengths:** Significant reduction in computational costs and storage requirements compared to volume-meshed post-processing [S1]. High flexibility due to meshfree nature [S1].
*   **Limitations:** Performance optimization was omitted in the source implementation, focusing on accuracy and resource testing [S1]. Coarse isosurfaces may lack smoothness without refinement [S1].
*   **When to Use:** When high-quality, meshfree isosurfaces are needed and computational resources allow for iterative field evaluations. Particularly suitable for post-processing of BEM results [S1].
*   **When Not to Use:** When real-time performance is critical and the overhead of field evaluation and bisection is prohibitive.

## Background
The method was presented by A. Buchau et al. in 2017 as an innovative approach for meshfree computation of isosurfaces [S1]. It combines octree-based isovalue search methods, FMM accelerated BEMs, and advancing front meshing algorithms [S1]. The implementation was done in C# using the .NET library for an in-house post-processing BEM software [S1].

## Sources
[S1] A meshfree isosurface computation method for boundary element methods, https://doi.org/10.2495/cmem-v5-n5-647-658, fetched 2026-10-26
