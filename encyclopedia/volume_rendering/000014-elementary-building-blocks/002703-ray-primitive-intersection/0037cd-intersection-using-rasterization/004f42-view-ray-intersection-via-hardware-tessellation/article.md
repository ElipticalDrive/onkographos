# View-Ray Intersection via Hardware Tessellation

Also called: Hardware Tessellation Ray-Surface Intersection

## Overview
This method computes the intersection points between view-rays and the boundary surfaces of volume blocks by reformulating the zero-finding problem as a surface approximation problem, utilizing GPU hardware tessellation for parallel efficiency [S1]. It differs from [Fragment Shader Volume Ray Casting](9618-fragment-shader-volume-ray-casting.md) by deferring the ray-marching computation to a later stage; instead, it focuses on generating precise entry and exit points for the volume blocks using rasterization primitives [S1].

## How it works
**Inputs:**
*   Volume blocks $G_\alpha$, each defined by spline surfaces.
*   Camera parameters defining the view-rays.

**Outputs:**
*   A texture buffer containing an unordered list of intersections per pixel, structured for subsequent depth-sorting.

**Data Structures:**
*   **Intersection List Entry:** A linked list node stored in a texture buffer (populated via atomic operations). Each entry contains:
    *   Parameter value (position in the parameter domain).
    *   Depth value (distance to the screen).
    *   Block number ($\alpha$).
    *   Orientation flag (front-facing or back-facing).

**Steps:**
1.  **Render Boundary Surfaces:** Render all boundary surfaces of all volume blocks $G_\alpha$. Each block has 6 boundary surfaces. This is performed using a single-pass algorithm based on Hjelmervik [S1].
    *   *Note:* The algorithm uses hardware tessellation to create a view-dependent triangulation of the spline surfaces. This approach guarantees water-tightness and satisfies the approximation error requirements for pixel-accurate rendering [S1].
2.  **Store Intersections:** Store the resulting intersection data into a texture buffer. The rasterization process effectively finds all ray-surface intersections of the triangulation in parallel [S1].
3.  **Populate Linked Lists:** For each pixel, populate a linked list of intersections. Modern GPUs allow atomic operations to build these lists in a single pass, a technique adapted from order-independent transparency methods [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Hardware Tessellation Rendering
# This step is handled by the GPU driver/hardware tessellator.
# The vertex shader and tessellation shader generate triangles
# that approximate the spline surfaces of volume blocks G_alpha.
# Fragment shader outputs intersection data to a texture buffer.

def render_volume_blocks(blocks, camera):
    for block in blocks:
        # Generate view-dependent triangulation of the 6 boundary surfaces
        # using hardware tessellation (Hjelmervik's method)
        triangulation = hardware_tessellate(block.spline_surfaces, camera.view_direction)
        
        # Rasterize the triangulation
        # For each pixel, if a ray intersects a triangle, record the hit
        rasterize(triangulation, camera, output_texture_buffer)

# Step 2: Post-processing for Depth Sorting (Algorithm 2)
# This is typically implemented in a subsequent shader pass or CPU-side
# if the list is small, but the source implies GPU atomic operations
# for list population. The sorting logic is described below.

def blockwise_depth_sort(intersection_lists):
    for pixel in pixels:
        list = intersection_lists[pixel]
        
        while list is not empty:
            # 1) Find pOut
            pOut = None
            min_depth = infinity
            for entry in list:
                if not entry.used and not entry.front_facing:
                    if entry.depth < min_depth:
                        min_depth = entry.depth
                        pOut = entry
            
            # 2) Exit if list is empty
            if pOut is None:
                break
            
            # 3) Find pIn
            pIn = None
            min_depth = infinity
            for entry in list:
                if not entry.used and entry.front_facing and entry.block_number == pOut.block_number:
                    if entry.depth < min_depth:
                        min_depth = entry.depth
                        pIn = entry
            
            # 4) Render volume between pIn and pOut
            # (This step is conceptual; the actual compositing happens later)
            render_volume_segment(pIn, pOut)
            
            # 5) Mark pOut and pIn as used
            pOut.used = True
            pIn.used = True
```

**Edge Cases and Pitfalls:**
*   **Neighboring Blocks:** In Isogeometric Analysis (IGA), surfaces of neighboring blocks match exactly. This produces entries in the intersection list with approximately the same depth value [S1]. The depth-sorting algorithm (Algorithm 2) must ensure that paired entry and exit points correspond to the same block number to avoid incorrect pairing [S1].
*   **Non-Convex Objects:** View-rays may intersect the geometry multiple times per block, leading to multiple entry and exit points [S1]. The linked list approach handles this by iterating through all entries.
*   **Multi-Block Objects:** Each volume block adds further intersections, requiring the block number check in the pairing logic [S1].

## Subtopics

### Blockwise Depth-Sorting of Intersections
A method to pair entry and exit points from an unordered list of intersections per view-ray, handling non-convex objects and multi-block structures using atomic operations for linked lists [S1]. It differs from simple depth-sorting by explicitly checking block numbers to ensure pairs correspond to the same volume block, which is critical when neighboring blocks produce nearly identical depth values [S1]. Choose this method when dealing with multi-block volumes or non-convex geometries where simple front-to-back sorting is insufficient [S1].

## Operating characteristics
*   **Time Complexity:** The intersection finding is performed in a single rasterization pass, leveraging parallel GPU hardware tessellation [S1]. The depth-sorting step requires iterating through the intersection list per pixel, which depends on the number of intersections (dependent on view-angle and geometry complexity) [S1].
*   **Memory Complexity:** Requires a texture buffer to store the intersection lists. The size of the buffer depends on the maximum number of intersections per pixel [S1].
*   **Strengths:** Highly parallel and efficient due to hardware tessellation. Guarantees water-tightness and pixel-accurate approximation error [S1]. Single-pass intersection finding [S1].
*   **Limitations:** Requires post-processing to sort and pair intersections. Precision issues may arise with neighboring blocks having identical surfaces, requiring careful block-number matching [S1].
*   **When to use:** When pixel-accurate geometry is required for volume rendering, especially with spline-based volumes (IGA) [S1].
*   **When not to use:** If the volume is convex and single-block, simpler ray-marching might be more efficient. If hardware tessellation is not available or inefficient for the specific geometry, other methods may be preferred.

## Background
The method originates from Interactive Isogeometric Volume Visualization with Pixel-Accurate Geometry [S1]. It builds on the work of Hjelmervik [11] for single-pass hardware tessellation and Yeo et al. [26] for view-dependent triangulation. The linked list approach for handling multiple intersections is adapted from order-independent transparency techniques by Yang et al. [25], Everitt [10], Carpenter and Ii [1], and Myers and Bavoil [20] [S1].

## Sources
[S1] Interactive Isogeometric Volume Visualization with Pixel-Accurate Geometry, https://doi.org/10.1109/tvcg.2015.2430337, fetched 2026-10-28
