# Occlusion Clipping with Orthogonal Opacity Map

Also called: Orthogonal Opacity Map Occlusion Clipping, Orthogonal Opacity Map Occlusion Culling, Opacity Culling and Clipping, Occlusion Culling and Clipping, Empty Space Skipping and Occlusion Clipping

## Overview
Occlusion Clipping with Orthogonal Opacity Map is an acceleration technique for texture-based volume rendering that skips occluded voxels by creating a low-resolution opacity map on a plane orthogonal to a major dataset axis [S1]. It solves the problem of inefficient per-voxel occlusion testing in texture-based rendering by operating at the sub-volume level, culling fully hidden sub-volumes and clipping partially hidden ones [S1]. Unlike early ray termination which tests occlusion per-voxel, this method tests occlusion at the sub-volume level, which is more efficient for texture-based rendering where primitives contain thousands of voxels [S1]. It differs from hierarchical approaches like [Hidden Volume Removal using Hierarchical Occlusion Maps](2624-hidden-volume-removal-using-hierarchical-occlusion-maps.md) by using a fixed-resolution, multi-pass averaging strategy rather than a tree structure, and it complements [Acceleration](18-acceleration.md) by being more effective when there are many non-empty voxels and significant occlusion [S1].

## How it works
**Inputs:**
- 3D texture dataset partitioned into axis-aligned sub-volume bounding boxes [S1].
- Camera parameters (view direction, projection type) [S1].
- Opacity threshold (typically 95% of maximum opacity) [S1].
- Desired number of opacity map updates per frame, $k$ [S1].

**Outputs:**
- Rendered image [S1].
- Updated orthogonal opacity map (used for subsequent sub-volumes in the same frame) [S1].

**Data Structures:**
1. **Orthogonal Opacity Map:** A 2D array of floating-point values. Each pixel represents the minimum opacity of the corresponding region in the frame buffer. The map is orthogonal to the major axis of the dataset most parallel to the viewing direction [S1].
2. **Sub-volume Bounding Boxes:** Axis-aligned boxes defining the extent of each texture primitive [S1].
3. **CPU Hierarchical Map:** A structure built on the CPU from the GPU-obtained low-resolution opacity map to speed up culling and clipping operations [S1].

**Steps:**

1. **Initialization:**
   - Set the opacity map to "invalid".
   - Calculate $\Delta\text{Voxels} = \lfloor \text{Total Voxels in non-empty boxes} / (k + 1) \rfloor$ [S1].
   - Render textures as if clipping is not used until $\Delta\text{Voxels}$ voxels are rendered [S1].

2. **Opacity Map Update (performed every $\Delta\text{Voxels}$ rendered):**
   - Project the current frame buffer onto an object-aligned plane orthogonal to the major dataset axis [S1].
   - Down-sample the projected image to a low-resolution map to reduce GPU-to-CPU transfer cost and comparison overhead [S1].
   - Each pixel in the map stores the minimum opacity of the corresponding region in the frame buffer [S1].
   - Mark the map as "valid" [S1].

3. **Culling and Clipping for each Sub-volume:**
   - **Project:** Project the front and back faces of the sub-volume's bounding box onto the orthogonal opacity map. Since the box is axis-aligned, projections are rectangles. For perspective projection, apply scaling and translation; for parallel projection, apply only translation [S1].
     - The distance between adjacent voxels $d$ projects to $d' = d(l_1 + l_2)/l_1$, where $l_1$ is the distance from the camera to the slice and $l_2$ is the slice thickness [S1].
   - **Combine:** Combine the two projections to determine the total projected area [S1].
   - **Cull Check:** If the projected area is fully covered by pixels with opacity $>$ threshold, cull the sub-volume (skip entirely) [S1].
   - **Clip Check (if not culled):**
     - Search for all transparent pixels in the map covering the projected bounding rectangle $R_p$ [S1].
     - Include any part of $R_p$ falling outside the map border as visible regions [S1].
     - Compute the bounding rectangle $R_v$ of all visible regions [S1].
     - Back-project $R_v$ to the front and back faces of the sub-volume (parallel to the opacity map) to get $R_{vpf}$ and $R_{vpb}$ [S1].
     - Compute the union $R_{vp} = R_{vpf} \cup R_{vpb}$ [S1].
     - If $R_{vp}$ is empty, cull the sub-volume [S1].
     - Otherwise, use $R_{vp}$ as the front/back faces to create a new bounding box for the clipped sub-volume [S1].
   - **Render:** Render the (possibly clipped) sub-volume [S1].
   - **Update:** Update the opacity map immediately before rendering the next sub-volume to track accumulated opacity [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volume(dataset, camera, k, threshold):
    # 1. Initialization
    opacity_map = None
    is_map_valid = False
    voxels_rendered = 0
    total_non_empty_voxels = sum(sv.voxel_count for sv in dataset.sub_volumes)
    delta_voxels = floor(total_non_empty_voxels / (k + 1))
    
    # Helper: Project a rectangular face onto the orthogonal opacity map
    def project_face(face_rect, cam, op_map):
        # Standard perspective/parallel projection matrix transform.
        # Returns (x_min, y_min, x_max, y_max) in map coordinates.
        pass # Standard graphics operation
    
    # Helper: Check if projected area is fully occluded
    def is_fully_occluded(proj_rect, op_map, thresh):
        for y in range(proj_rect.y_min, proj_rect.y_max):
            for x in range(proj_rect.x_min, proj_rect.x_max):
                if op_map[y, x] <= thresh:
                    return False
        return True
        
    # Helper: Find transparent pixels covering a rectangle
    def find_transparent_pixels(proj_rect, op_map):
        visible = []
        for y in range(proj_rect.y_min, proj_rect.y_max):
            for x in range(proj_rect.x_min, proj_rect.x_max):
                if op_map[y, x] <= thresh:
                    visible.append((x, y))
        return visible
        
    # Helper: Compute bounding rectangle of a list of pixels
    def compute_bounding_rect(pixels):
        if not pixels: return None
        xs = [p[0] for p in pixels]
        ys = [p[1] for p in pixels]
        return (min(xs), min(ys), max(xs), max(ys))
        
    # Helper: Back-project rect to sub-volume front/back faces
    def back_project(rect, bbox, cam):
        # Standard inverse projection to front and back planes.
        pass # Standard graphics operation
        
    # Helper: Union of two rectangles
    def union_rects(r1, r2):
        if r1 is None: return r2
        if r2 is None: return r1
        return (min(r1.x_min, r2.x_min), min(r1.y_min, r2.y_min),
                max(r1.x_max, r2.x_max), max(r1.y_max, r2.y_max))

    rendered_image = []
    
    for sub_volume in dataset.sub_volumes:
        # Update map if invalid or voxel budget exceeded
        if not is_map_valid or voxels_rendered >= delta_voxels:
            opacity_map = project_framebuffer_to_orthogonal_map(camera)
            is_map_valid = True
            voxels_rendered = 0
            
        proj_front = project_face(sub_volume.bbox.front_face, camera, opacity_map)
        proj_back = project_face(sub_volume.bbox.back_face, camera, opacity_map)
        combined_rect = union_rects(proj_front, proj_back)
        
        # Cull check
        if is_fully_occluded(combined_rect, opacity_map, threshold):
            continue
            
        # Clip check
        visible_pixels = find_transparent_pixels(combined_rect, opacity_map)
        # Handle border: add area outside map border to visible regions
        if combined_rect.x_min < 0 or combined_rect.y_min < 0 or \
           combined_rect.x_max > opacity_map.width or combined_rect.y_max > opacity_map.height:
            visible_pixels.append("border")
            
        if not visible_pixels:
            continue
            
        rect_visible = compute_bounding_rect(visible_pixels)
        bp_front, bp_back = back_project(rect_visible, sub_volume.bbox, camera)
        clipped_union = union_rects(bp_front, bp_back)
        
        if clipped_union is None:
            continue
            
        render_sub_volume(sub_volume, clipped_union)
        voxels_rendered += sub_volume.voxel_count
        
    return rendered_image
```

**Edge Cases and Pitfalls:**
- **Precision:** The opacity value of each pixel increases monotonically during the rendering of the same frame [S1].
- **Boundaries:** Part or even the entire projected rectangle may fall out of the border of the opacity map, which is then added to the visible regions [S1].
- **Performance:** Current implementations require CPU involvement and data transfer across the AGP bus, which limits performance [S1].
- **Accuracy vs Efficiency:** Low-resolution opacity map trades off accuracy of occlusion detection for efficiency on the GPU [S1].

## Subtopics

### [Orthogonal Opacity Map Creation](991-orthogonal-opacity-map-creation.md)
This method creates a 2D opacity map on a plane orthogonal to a major axis (typically the view direction) by projecting the rendered frame buffer onto that plane and downsampling the result. It solves the problem of efficiently generating a low-resolution occlusion mask for volume rendering culling, minimizing data transfer costs between GPU and CPU while preserving sufficient occlusion information. It differs from hierarchical approaches by using a fixed-resolution, multi-pass averaging strategy rather than a tree structure. See [Orthogonal Opacity Map Creation](991-orthogonal-opacity-map-creation.md).

### [Hidden Volume Removal using Hierarchical Occlusion Maps](2624-hidden-volume-removal-using-hierarchical-occlusion-maps.md)
An occlusion culling method using Hierarchical Occlusion Maps (HOMs) to skip hidden nodes in the min-max octree, compensating for the lack of early ray termination in object-order rendering. See [Hidden Volume Removal using Hierarchical Occlusion Maps](2624-hidden-volume-removal-using-hierarchical-occlusion-maps.md).

### Adaptive Number of Updates
An adaptive system to adjust the number of opacity map updates ($k$) per frame based on performance history to balance rendering speed and occlusion detection overhead [S1]. The system starts with an empirically determined $k$ (typically 9) and maintains a performance history window [S1]. Every $n$-th frame (e.g., $n=10$), it polls the performance using the number in range $[k-hw, k+hw]$ that has not been updated for the longest [S1]. The performance function is $p = v - c \cdot k$, where $v$ is the percentage of voxels culled and clipped, and $c$ is a constant (0.02 for tested datasets) [S1]. When all slots are valid, the slot with the best performance becomes the new center [S1].

### Occlusion Culling for Point Splatting
A technique to save time by culling points that are occluded by opaque volume material in front of their footprint [S2, unofficial]. It requires front-to-back rendering to maintain an opacity image [S2, unofficial]. Mueller '00 method: After compositing, convolve the opacity image with a box filter sized to the projected footprint [S2, unofficial]. If a pixel value in the convolved map is $>$ threshold, the entire footprint neighborhood is $>$ threshold, allowing fast culling by checking only the pixel nearest to the projected point center [S2, unofficial]. Lee '00 method: Keep points in an octree and maintain a visibility map as a quadtree [S2, unofficial]. Check the projection of an octree node against the corresponding level of the visibility map quadtree [S2, unofficial]. Cull occluded octree nodes; subdivide if not occluded; rasterize points that fail the occlusion test [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of sub-volumes and the cost of projecting and checking against the opacity map. The adaptive update strategy aims to minimize the number of map updates ($k$) to balance overhead and culling benefit [S1].
- **Memory Complexity:** Requires storage for the orthogonal opacity map (low-resolution 2D array) and the CPU hierarchical map [S1].
- **Measured Performance:** On average, 50-60% of non-empty voxels are excluded from rendering [S1]. The average speedup factor for ESS+CC vs conventional is 3.3 [S1]. For the 'Knee' dataset, ESS+CC achieves a speedup of 4.7x over Basic, with 63.8% of voxels skipped [S1]. With occlusion clipping, about 60% of non-empty voxels can be skipped, gaining an additional 80% speedup on average for iso-surface-like rendering [S1].
- **Scaling:** Performance varies with dataset, transfer function, and viewing angle [S1].
- **Strengths:** Efficient for texture-based rendering with many non-empty voxels and significant occlusion [S1].
- **Limitations:** Requires CPU involvement and data transfer across the AGP bus, which limits performance [S1]. Low-resolution map trades off accuracy for efficiency [S1].
- **When to use:** When there are many non-empty voxels and significant occlusion [S1].
- **When not to use:** When there is a large amount of empty voxels, where [Acceleration](18-acceleration.md) is more efficient [S1].

## Background
Occlusion Clipping with Orthogonal Opacity Map was introduced by Mueller et al. in the paper "Empty Space Skipping and Occlusion Clipping for Texture-based Volume Rendering" [S1]. It is related to hardware occlusion queries used for visibility culling in opaque surfaces, as described by Govindraju et al. 2003 [S1]. The method extends the approach in [Li and Kaufman 2003] to 3D textures by converting the box sets bounding the sub-volumes to an orthogonal BSP tree, developing an incremental slicing algorithm to render the axis-aligned boxes, introducing the concept of orthogonal opacity map to cull and clip sub-volumes efficiently, and presenting methods to create the orthogonal opacity map efficiently on the GPU, along with an algorithm for adaptive update [S1].

## Sources
[S1] Empty Space Skipping and Occlusion Clipping for Texture-based Volume Rendering, https://www3.cs.stonybrook.edu/~mueller/papers/OcclusionClipping.pdf, fetched 2026-10-26
[S2] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
