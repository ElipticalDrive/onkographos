# Neubauer Iso-surface Ray Casting
Also called: Kd-Jump, Neubauer method

## Overview
Neubauer Iso-surface Ray Casting is a single-kernel iso-surface ray casting method that uses a min/max octree built from macro-cells to accelerate ray traversal [S2, S1, unofficial]. It sets itself apart from its siblings by projecting macro-cell boundaries onto the image plane to create a hexagonal footprint, allowing local rays to skip redundant octree traversal steps [S1, unofficial]. Unlike [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) or [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md), it relies on spatial subdivision rather than iterative root-finding for the initial traversal, though it may use Newton-Raphson for precise intersection within the cell [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A 3D scalar volume dataset, an iso-value threshold, and camera parameters (image plane resolution, ray origin, direction).
*   Output: A 2D image where pixels are colored based on the first intersection with the iso-surface.

**Data Structures:**
1.  **Macro-Cell Grid:** The volume is subdivided into cubic macro-cells of size $m \times m \times m$ voxels.
2.  **Min/Max Octree:** A hierarchical structure built from the macro-cells. Each node stores the minimum and maximum scalar values found within its bounding volume. This allows for early rejection: if the iso-value is not within $[min, max]$ of a node, the node is skipped [S1, unofficial].

**Parameters:**
*   $m$: Macro-cell size dimension. Typical values are between 4 and 10 [S1, unofficial].

**Steps:**
1.  **Preprocessing:**
    *   Subdivide the complete data set into macro-cells of size $m^3$.
    *   Build a min/max octree from these macro-cells. For each node, compute the min and max scalar values of all voxels in its subtree [S1, unofficial].
2.  **Ray Traversal (Per Pixel):**
    *   For each pixel on the image plane, cast a primary ray.
    *   Traverse the octree from the root. At each step, check if the ray intersects the node's bounding box.
    *   If the ray intersects the bounding box, check the node's min/max values. If the iso-value is outside $[min, max]$, prune the subtree (do not traverse children) [S1, unofficial].
    *   Continue traversal until a leaf node is reached.
3.  **Leaf Processing (Hexagonal Footprint Optimization):**
    *   At a leaf node (macro-cell), project the boundaries of the macro-cell onto the image plane.
    *   This projection yields a hexagonal footprint on the image plane [S1, unofficial].
    *   Identify all pixels that fall within this hexagonal footprint.
    *   For these pixels, perform local ray traversal within the macro-cell grid. This avoids re-traversing the upper levels of the octree for rays that are spatially close and likely to traverse the same macro-cell [S1, unofficial].
4.  **Iso-surface Intersection:**
    *   Within the macro-cell, find the intersection of the ray with the iso-surface. The source notes mention Neubauer's approach generally but do not specify the exact root-finding algorithm used *inside* the cell in the provided text snippets, though it is implied to be part of the "Neubauer method" [S1, unofficial]. [G: Typically, this involves evaluating the scalar field along the ray and using Newton-Raphson or similar methods to find the root, given the context of iso-surface ray casting.]
5.  **Shading:**
    *   Compute surface normal and apply shading model at the intersection point [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neubauer_ray_casting(volume, iso_value, m, image_width, image_height):
    # Preprocessing
    macro_cells = build_macro_cells(volume, m)
    octree = build_min_max_octree(macro_cells)
    
    image = empty_image(image_width, image_height)
    
    # Process pixels
    for y in range(image_height):
        for x in range(image_width):
            ray = create_ray_from_pixel(x, y, camera_params)
            
            # Traverse octree
            leaf_node = traverse_octree(octree, ray, iso_value)
            
            if leaf_node is None:
                continue
                
            # Get hexagonal footprint for this leaf node
            hexagon = project_macro_cell_boundaries(leaf_node, camera_params)
            
            # Identify pixels in footprint to optimize traversal
            pixels_in_footprint = get_pixels_in_hexagon(hexagon, image_width, image_height)
            
            for px, py in pixels_in_footprint:
                # Re-cast ray for this pixel (or reuse if parallel)
                local_ray = create_ray_from_pixel(px, py, camera_params)
                
                # Traverse within macro-cell (local grid)
                intersection = find_iso_surface_intersection(local_ray, leaf_node.data, iso_value)
                
                if intersection is not None:
                    normal = compute_normal(intersection, leaf_node.data)
                    color = shade(intersection, normal, lighting_model)
                    image[py, px] = color

def traverse_octree(node, ray, iso_value):
    if node is None:
        return None
    
    # Check bounding box intersection
    if not ray_intersects_bbox(ray, node.bbox):
        return None
    
    # Min/Max test
    if not (node.min_val <= iso_value <= node.max_val):
        return None
    
    # If leaf, return it
    if node.is_leaf:
        return node
    
    # Recurse into children
    for child in node.children:
        result = traverse_octree(child, ray, iso_value)
        if result is not None:
            return result
            
    return None

def find_iso_surface_intersection(ray, cell_data, iso_value):
    # [G] Implementation detail not fully specified in sources.
    # Typically involves sampling the scalar field along the ray
    # and solving for the root where f(t) = iso_value.
    # Newton-Raphson is a common choice for smooth fields.
    t = newton_raphson_solve(ray, cell_data, iso_value)
    if t is valid:
        return ray.origin + t * ray.direction
    return None
```

## Subtopics

### [Bézier Clipping for Metaball Ray Casting](9481-b-zier-clipping-for-metaball-ray-casting.md)
This method computes the ray-isosurface intersection for metaballs by representing the scalar density field along a ray as a polynomial in Bézier form and applying Bézier clipping to find the root. It sets itself apart from standard Newton-Raphson approaches by providing robust root isolation and handling multiple intersections within a single interval without derivative evaluation, building on [Neubauer Iso-surface Ray Casting](7024-neubauer-iso-surface-ray-casting.md).

## Operating characteristics
*   **Time Complexity:** Depends on the octree depth and the number of macro-cells intersected. The hexagonal footprint optimization reduces redundant traversal for neighboring rays [S1, unofficial].
*   **Memory Complexity:** Requires storage for the min/max octree. The source notes that the octree approach consumes much less additional memory for acceleration structures compared to some other methods [S1, unofficial].
*   **Measured Performance:**
    *   On a single Pentium IV 1.9 GHz processor, an average frame rate of 0.8 fps was reported for a 512x512 image resolution [S1, unofficial].
    *   In a cluster setting for the "Visible Female" dataset (428 MB), performance scales linearly with the number of render nodes: ~1 fps with 1 node, up to 22 fps with 32 nodes [S1, unofficial].
*   **Strengths:** Efficient space-leaping via min/max octree; reduced traversal overhead for neighboring rays via hexagonal footprint projection; lower memory overhead for acceleration structures compared to some alternatives [S1, unofficial].
*   **Limitations:** Slower than Wald's kd-tree approach for certain datasets (e.g., Visible Female: ~1-22 fps on cluster vs ~8.1 fps on single PC for Wald) [S1, unofficial]. Performance is sensitive to the choice of macro-cell size $m$ [S1, unofficial].
*   **When to use:** When memory is constrained for acceleration structures, or when rendering on clusters where linear scaling is beneficial. Not ideal when single-PC performance for complex datasets is the primary goal compared to kd-tree based methods [S1, unofficial].

## Background
The method was introduced by Neubauer et al. in 2002 [S1, unofficial]. It is used as a baseline for comparison in single-kernel iso-surface ray casting research [S2, unofficial]. The approach builds on the geometric definition of rays and the parallel execution model defined in [Ray-primitive intersection](769-ray-primitive-intersection.md).

## Sources
[S1] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
[S2] InKernel Stream Compaction and Its Application to ..., https://www.sci.utah.edu/~knolla/hughes13.pdf, fetched 2026-10-29 (unofficial)
