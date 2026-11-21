# Occlusion Clipping with Orthogonal Opacity Map

Also called: Orthogonal Opacity Map Occlusion Clipping, Orthogonal Opacity Map Occlusion Culling, Opacity Culling and Clipping, Occlusion Culling and Clipping, Empty Space Skipping and Occlusion Clipping

## Overview
Occlusion Clipping with Orthogonal Opacity Map is an acceleration technique for texture-based volume rendering that skips occluded voxels by maintaining a low-resolution 2D opacity map on a plane orthogonal to a major dataset axis [S1]. It differs from per-voxel early ray termination by testing occlusion at the sub-volume level, allowing it to cull entire bounding boxes or clip their geometry against the opacity map to avoid rendering thousands of voxels hidden behind opaque surfaces [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D texture dataset, transfer function, viewing parameters, sub-volume bounding boxes (axis-aligned).
*   **Output:** Final image, updated opacity map.

**Data Structures:**
1.  **Orthogonal Opacity Map:** A 2D array of floating-point values. Each pixel represents the minimum opacity accumulated in the corresponding region of the frame buffer. The map is orthogonal to the major axis of the dataset most parallel to the viewing direction [S1].
2.  **Hierarchical CPU Map:** A CPU-side structure (e.g., min-max octree or hierarchical grid) built from the GPU-obtained low-resolution opacity map to accelerate culling queries [S1].
3.  **Sub-volume Bounding Boxes:** Axis-aligned boxes defining regions of the volume to be rendered [S1].

**Parameters:**
*   **Opacity Threshold ($T_{op}$):** Typically 95% of the maximum opacity [S1].
*   **Desired Updates ($k$):** Initial empirically determined number of map updates per frame (typically 9) [S1].
*   **DeltaVoxels ($D_v$):** Number of voxels rendered before an opacity map update. Calculated as $D_v = \lfloor \text{Total Voxels in non-empty boxes} / (k + 1) \rfloor$ [S1].
*   **Constant ($c$):** Used in performance function, typically 0.02 [S1].

**Steps:**

1.  **Initialization:**
    *   Set opacity map to invalid.
    *   Render textures as if clipping is not used.
    *   Initialize voxel counter $V_{count} = 0$.

2.  **Rendering Loop (per sub-volume):**
    *   If opacity map is invalid, render sub-volume normally.
    *   Increment $V_{count}$ by the number of voxels in the sub-volume.
    *   If $V_{count} \ge D_v$:
        *   Update Opacity Map (see Step 3).
        *   Reset $V_{count} = 0$.
    *   If opacity map is valid:
        *   **Culling/Clipping:**
            1.  Project the front and back faces of the sub-volume's bounding box onto the orthogonal opacity map. Since the box is axis-aligned, projections are rectangles. For perspective projection, apply scaling and translation; for parallel, only translation [S1].
            2.  Combine the two projections to determine the total projected area.
            3.  Search for transparent pixels in the map covering the projected rectangle $R_p$. Pixels with opacity $< T_{op}$ are transparent.
            4.  If $R_p$ falls outside the map border, add that area to visible regions.
            5.  Compute the bounding rectangle $R_v$ of all visible regions.
            6.  **Back-project:** Project $R_v$ back to the front and back faces of the sub-volume (parallel to the opacity map) to get $R_{vpf}$ and $R_{vpb}$.
            7.  Compute union $R_{vp} = R_{vpf} \cup R_{vpb}$.
            8.  If $R_{vp}$ is empty, cull the sub-volume (skip rendering).
            9.  Otherwise, use $R_{vp}$ as the new front/back faces to create a clipped bounding box and render only the visible portion [S1].

3.  **Opacity Map Update:**
    *   Project the current frame buffer onto the object-aligned plane orthogonal to the major axis [S1].
    *   Down-sample the result to a low-resolution image to reduce GPU-to-CPU transfer costs and comparison overhead [S1].
    *   Each pixel in the map stores the minimum opacity of the corresponding frame buffer region [S1].
    *   Note: Opacity values in the map increase monotonically during the rendering of the same frame [S1].

**Formula:**
*   **Projection Distance:** When projecting a slice parallel to the opacity map, the distance between adjacent voxels $d$ becomes $d'$:
    $$ d' = d \frac{l_1 + l_2}{l_1} $$
    where $l_1$ is the distance from the camera to the slice, and $l_2$ is the slice thickness [S1].

**Pitfalls:**
*   **Resolution Trade-off:** Low-resolution maps reduce accuracy of occlusion detection but improve efficiency [S1].
*   **CPU Bottleneck:** Current implementations require CPU involvement and data transfer across the AGP/PCIe bus, limiting performance [S1].
*   **Degenerate Input:** If the projected area is fully covered by pixels with opacity $> T_{op}$, the object is safely skipped [S1].

## Subtopics

### [Orthogonal Opacity Map Creation](991-orthogonal-opacity-map-creation.md)
This method creates a 2D opacity map on a plane orthogonal to a major axis by projecting the rendered frame buffer onto that plane and downsampling the result. It solves the problem of efficiently generating a low-resolution occlusion mask for volume rendering culling, minimizing data transfer costs between graphics and main memory while preserving sufficient occlusion information. It differs from hierarchical approaches by using a fixed-resolution, multi-pass averaging strategy rather than a tree structure. See [Orthogonal Opacity Map Creation](991-orthogonal-opacity-map-creation.md).

### [Hidden Volume Removal using Hierarchical Occlusion Maps](2624-hidden-volume-removal-using-hierarchical-occlusion-maps.md)
Hidden Volume Removal (HVR) is an occlusion culling optimization for object-order volume rendering that eliminates a large fraction of cells and octree nodes within the opacity range, particularly effective for isosurface visualization. It compensates for the lack of early ray termination inherent in object-order rendering by using Hierarchical Occlusion Maps (HOMs) to skip hidden nodes in the min-max octree. Unlike its sibling [Orthogonal Opacity Map Creation](991-orthogonal-opacity-map-creation.md), which relies on orthogonal projections and neighbor finding, HVR uses a hierarchical map structure to approximate node visibility, providing superior skipping efficiency at the cost of some artifacts due to approximation. See [Hidden Volume Removal using Hierarchical Occlusion Maps](2624-hidden-volume-removal-using-hierarchical-occlusion-maps.md).

### Adaptive Number of Updates
An adaptive system to adjust the number of opacity map updates ($k$) per frame based on performance history to balance rendering speed and occlusion detection overhead [S1].
*   **Step:** Polling: Every $n$-th frame (e.g., $n=10$), poll performance using the number in range $[k-h_w, k+h_w]$ that has not been updated for the longest. Increment age of all slots by 1 except the polled one.
*   **Step:** Update $k$: When all slots are valid, pick the slot with best performance as the new center. Change $k$ to this new center and shift performance statistics accordingly.
*   **Formula:** Performance function $p = v - c \cdot k$, where $v$ is the percentage of voxels culled and clipped, $k$ is the actual number of map updates, and $c$ is a constant (0.02 for tested datasets) [S1].
*   **Parameter:** $k$: Initial empirically determined number of map updates (typically 9) [S1].

### Occlusion Culling for Point Splatting
A technique to save time by culling points that are occluded by opaque volume material in front of their footprint [S2, unofficial].
*   **Step:** Require front-to-back rendering to maintain an opacity image (occlusion map).
*   **Step:** Check if the volume material in front of the point's footprint is opaque; if so, the point is not visible.
*   **Step:** Mueller '00 method: After compositing, convolve the opacity image with a box filter sized to the projected footprint. If a pixel value in the convolved map is $> threshold$, the entire footprint neighborhood is $> threshold$, allowing fast culling by checking only the pixel nearest to the projected point center [S2, unofficial].
*   **Step:** Lee '00 method: Keep points in an octree and maintain a visibility map as a quadtree. Check the projection of an octree node against the corresponding level of the visibility map quadtree. Cull occluded octree nodes; subdivide if not occluded; rasterize points that fail the occlusion test [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of sub-volumes and the cost of projecting bounding boxes onto the opacity map. The adaptive update mechanism aims to minimize the number of expensive frame buffer reads [S1].
*   **Memory Complexity:** Requires storage for the low-resolution opacity map and the CPU-side hierarchical map [S1].
*   **Measured Performance:** On average, 50-60% of non-empty voxels are excluded from rendering. The average speedup factor for Empty Space Skipping + Occlusion Clipping (ESS+CC) vs conventional is 3.3x [S1]. For the 'Knee' dataset, ESS+CC achieves a speedup of 4.7x over Basic, with 63.8% of voxels skipped [S1].
*   **Strengths:** Highly effective when there are many non-empty voxels and significant occlusion, complementing Empty Space Skipping which is better for large amounts of empty voxels [S1].
*   **Limitations:** Requires CPU involvement and data transfer across the bus, which limits performance [S1]. Low-resolution maps trade off accuracy for efficiency [S1].

## Background
The technique was introduced by Mueller et al. in 2004 [S1]. It extends earlier work on empty space skipping and is related to hardware occlusion queries used for visibility culling in opaque surfaces, as described by Govindraju et al. in 2003 [S1]. It is designed specifically for texture-based volume rendering, where primitives contain thousands of voxels, making per-voxel tests inefficient [S1].

## Sources
[S1] Empty Space Skipping and Occlusion Clipping for Texture-based Volume Rendering, https://www3.cs.stonybrook.edu/~mueller/papers/OcclusionClipping.pdf, fetched 2026-10-26
[S2] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
