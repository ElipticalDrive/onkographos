# Tile-Based Culling

Also called: StopThePop Tile-Based Culling

## Overview
Tile-Based Culling computes a precise mapping between 3D Gaussians and image tiles to eliminate irrelevant tiles during rasterization [S1]. It solves the problem of overestimating the number of tiles a Gaussian intersects, which occurs when using standard axis-aligned bounding boxes (AABBs) that are overly conservative [S1][S2, unofficial]. Unlike the parent method, which assigns a Gaussian to every tile its projected splat touches (potentially including tiles with negligible opacity), Tile-Based Culling iteratively examines candidate tiles to verify actual overlap, thereby reducing the workload for the subsequent radix sort and rendering phases [S1]. Tile-Based Culling iterates over all candidate tiles while AccuTile does not, resulting in a markedly higher runtime cost for Tile-Based Culling despite warp-level load balancing [S1].

## How it works
The method computes a tight, opacity-aware bounding box per Gaussian and then iteratively checks each tile within that box to determine if it should be included in the final Gaussian-to-Tile mapping [S1]. The implementation described in the primary source selects the approach by Radl et al. for its simpler control flow and load balancing [S2, unofficial].

**Inputs and Outputs:**
*   **Input:** A list of $N$ projected 2D Gaussians, each defined by center $(x_i, y_i)$, 2D covariance matrix $\Sigma'_i$, and opacity $\alpha_i$ [S2, unofficial].
*   **Output:** A list of (Gaussian, Tile) pairs, where each pair represents a verified overlap between a Gaussian and a tile [S1].

**Data Structures:**
*   **Bounding Box:** An axis-aligned bounding box (AABB) in screen space that tightly encloses the Gaussian's elliptical extent above a certain opacity threshold [S1].
*   **Tile List:** A dynamic list or buffer storing the indices of tiles that overlap with the Gaussian [S1].

**Parameters:**
*   **Padded Alpha Threshold:** A threshold value used to accurately compute the bounding box and prevent undercounting Gaussian-to-Tile mappings [S1]. The specific value is not provided in the ablation study [S1].

**Steps:**
1.  **Compute Tight Bounding Box:** Calculate a tight, opacity-aware bounding box for each Gaussian [S1]. This box is tighter than the standard AABB used in basic splatting but may not be perfectly tight if thresholds are not handled correctly [S1].
2.  **Iterative Tile Examination:** For each Gaussian, iterate over all tiles that touch its computed bounding box [S1].
3.  **Overlap Verification:** For each candidate tile, determine if the Gaussian actually overlaps with it [S1]. This is done by computing the point where the value of the Gaussian is maximal for each tile inside the bounding rectangle and checking if this value exceeds the opacity threshold [S2, unofficial].
4.  **Warp-Level Load Balancing:** Use warp-level load balancing to accelerate the tile examination process, ensuring that threads within a warp process tiles efficiently even if the number of candidate tiles varies [S1][S2, unofficial].
5.  **Generate Mapping:** Add the (Gaussian, Tile) pair to the output list if an overlap is verified [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tile_based_culling(gaussians, tile_grid_width, tile_grid_height, alpha_threshold, tile_size=16):
    """
    Computes precise Gaussian-to-Tile mappings using iterative examination.
    """
    mappings = []
    for i, g in enumerate(gaussians):
        # Step 1: Compute tight, opacity-aware bounding box
        # [G] The exact bounding box derivation is not provided in the sources.
        # It is computed from the Gaussian's elliptical extent and alpha threshold.
        bbox_min_x, bbox_min_y, bbox_max_x, bbox_max_y = compute_tight_bbox(g, alpha_threshold)
        
        # Determine the range of tiles to examine
        tile_min_x = max(0, int(bbox_min_x // tile_size))
        tile_min_y = max(0, int(bbox_min_y // tile_size))
        tile_max_x = min(tile_grid_width - 1, int(bbox_max_x // tile_size))
        tile_max_y = min(tile_grid_height - 1, int(bbox_max_y // tile_size))
        
        # Step 2 & 3: Iteratively examine each tile and verify overlap
        for ty in range(tile_min_y, tile_max_y + 1):
            for tx in range(tile_min_x, tile_max_x + 1):
                tile_id = ty * tile_grid_width + tx
                # Compute the point of maximal Gaussian value within this tile
                # [G] The specific algorithm for finding the peak location inside a tile
                # is detailed in Radl et al. [S2, unofficial].
                peak_x = tx * tile_size + tile_size / 2.0
                peak_y = ty * tile_size + tile_size / 2.0
                dx = peak_x - g.center.x
                dy = peak_y - g.center.y
                # Standard 2D Gaussian density evaluation
                max_value = g.opacity * math.exp(-0.5 * (dx**2 / g.covariance[0][0] + dy**2 / g.covariance[1][1]))
                
                # Check if the Gaussian overlaps with the tile
                if max_value > alpha_threshold:
                    mappings.append((i, tile_id))
    return mappings

def compute_tight_bbox(gaussian, alpha_threshold):
    # [G] Standard implementation solves for the ellipse intersection at the threshold level.
    # Returns (min_x, min_y, max_x, max_y) in screen coordinates.
    return 0.0, 0.0, 0.0, 0.0
```

**Edge Cases and Pitfalls:**
*   **Threshold Usage:** The use of thresholds in the original code can cause not all bounding boxes to be tight [S1].
*   **Undercounting:** Without a padded alpha threshold, the method undercounts Gaussian-to-Tile mappings [S1]. This artificially improves downstream performance (Radix Sort, Identify Tile Ranges, Render) due to fewer elements to process, but leads to incorrect rendering [S1].
*   **Comparison with AccuTile:** Tile-Based Culling iterates over all candidate tiles while AccuTile does not, resulting in a markedly higher runtime cost for Tile-Based Culling despite warp-level load balancing [S1].

## Subtopics

### [SnugBox](3378-snugbox.md)
SnugBox is a precise algorithm for computing the axis-aligned bounding box (AABB) of a 3D Gaussian in screen space, replacing the overly conservative tile assignment used in standard 3D Gaussian Splatting (3D-GS). It solves the problem of overestimating the number of tiles a Gaussian intersects, thereby reducing the number of pixels processed per Gaussian and accelerating rendering. Unlike the standard 3D-GS method which inscribes a circle within a square tile assignment, SnugBox derives a tight bounding box directly from the Gaussian's elliptical extent using its conic equation, providing numerical stability even under edge-on tilts. It serves as a plug-and-play optimization that yields an average inference speed-up of 1.82× without altering the rendered image.

### [Warp-Level Culling for Efficient Blending in 3D Gaussian Splatting](12278-warp-level-culling-for-efficient-blending-in-3d-gaussian-spl.md)
An optimization for the 3D Gaussian Splatting (3DGS) alpha blending stage that uses warp-level collaboration to subdivide image tiles and cull splats with no geometric overlap, avoiding expensive alpha evaluation. It improves rendering speed without changing the rendered output, implemented in the existing CUDA differentiable pipeline. Unlike [SnugBox](3378-snugbox.md), which focuses on tighter bounding boxes, this method focuses on geometric culling within the blending kernel to reduce the number of blending operations.

### GSAcc
A method introducing a depth-speculation mechanism that predicts per-tile depth ranges, enabling early culling of occluded Gaussians before sorting [S4, unofficial].

### Dynamic Cutouts for Gaussian Filtering
Allows runtime editing of visible Gaussians using cutout components that target all, parent, or specific renderers [S5, unofficial]. It evaluates cutout refresh rate and trigger bounds/visibility updates based on SortMode [S5, unofficial]. The parameter `CutoutsUpdateBounds` controls whether cutouts recalculate the renderer's world bounds, which is costly for moving cutouts [S5, unofficial].

### Complete tile culling using scan-line intersection
A method to compute exact intersections between screen-space Gaussian ellipses and tiles, eliminating false-positive and false-negative intersections found in prior methods (distance-based, bounding-box, or compact box approaches) [S6, unofficial]. It computes the number of intersecting tiles per Gaussian in the projection-forward pass and uses a separate pass to fill the buffer with tile-depth pairs [S6, unofficial]. For a Gaussian represented as an ellipse containing opacity above threshold and an interval for one axis, it computes the closed-form interval on the other axis that intersects the ellipse [S6, unofficial]. It picks the shorter dimension of the screen-space Gaussian ellipse (e.g., vertical) and, for each row (or column) of tiles, finds the ellipse's range of coordinates within the row, bounded by either the tile boundary or global extreme points of the ellipse [S6, unofficial]. The closed-form solution is implemented as a highly optimized branchless function, rounding down the minimum coordinate and rounding up the maximum coordinate toward the tile boundary to efficiently count or iterate through tiles overlapped by the ellipse [S6, unofficial].

### STP with Optimal Projection Adaptation
An adaptation of STP's culling logic to work with Optimal Projection planes, where tiles form general quadrilaterals rather than rectangles [S7]. It projects the tiles touched by each Gaussian onto its optimal plane, yielding non-rectangular quadrilaterals [S7]. For a splat on its optimal plane with mean $\mu \in \mathbb{R}^2$, it identifies tile edges visible at $\mu$, denoting a tile edge as $p + td$ [S7]. The point of highest density on a tile edge is found by taking the derivative of Gaussian density with respect to $t$ and setting it to zero: $p + \left(\frac{d^\top \Sigma^{-1} (\mu - p)}{d^\top \Sigma^{-1} d}\right)d$, where $\Sigma$ is the 2D covariance matrix [S7]. At most two tile edges are visible at any given mean $\mu$ [S7].

## Operating characteristics
*   **Time Complexity:** The method involves iterating over all candidate tiles within the bounding box for each Gaussian [S1]. This results in higher runtime cost for the preprocessing and key generation phases compared to methods that do not iterate over all candidates (like AccuTile) [S1]. However, it reduces the number of elements in the downstream Radix Sort and Render phases if the culling is effective [S1].
*   **Memory Complexity:** The output is a list of (Gaussian, Tile) pairs [S1]. The size of this list depends on the number of verified overlaps, which is generally smaller than the number of overlaps in standard splatting but larger than in methods that do not iterate over all candidates [S1].
*   **Performance:** In ablation studies, Tile-Based Culling showed significant speedup in Preprocess and Duplicate with Keys compared to baseline 3D-GS, but was slower than AccuTile in these phases due to the iterative examination [S1]. It slightly outperformed AccuTile in downstream functions (Radix Sort, Identify Tile Ranges, Render) in the ablation, but this was attributed to undercounting of mappings due to the lack of a padded alpha threshold [S1].
*   **Strengths:** Provides a more precise mapping than standard AABB-based assignment, reducing unnecessary computations in downstream phases [S1]. Warp-level load balancing helps with performance [S1][S2, unofficial].
*   **Limitations:** Iterating over all candidate tiles can be computationally expensive [S1]. The accuracy of the bounding box and the resulting mapping is sensitive to the alpha threshold [S1].

## Background
Tile-Based Culling was introduced in the "StopThePop" paper [S1]. It is designed to address the "popping" artifacts and inefficiencies in 3D Gaussian Splatting rendering by providing a more accurate culling mechanism [S1]. The method builds on the principles of tile-based rasterization and radix sorting [S1][S2, unofficial].

## Sources
[S1] Speedy-Splat: Fast 3D Gaussian Splatting with Sparse Pixels and Sparse Primitives, https://doi.org/10.1109/cvpr52734.2025.02006, fetched 2026-10-26
[S2] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
[S3] 3D Gaussian Splatting - Paper Explained, Training NeRFStudio, https://learnopencv.com/3d-gaussian-splatting/, fetched 2026-10-28 (unofficial)
[S4] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S5] wuyize25/gsplat-unity, https://github.com/wuyize25/gsplat-unity, fetched 2026-10-27 (unofficial)
[S6] VkSplat: High-Performance 3DGS Training in Vulkan Compute, https://doi.org/10.2312/egs.20261024, fetched 2026-10-29 (unofficial)
[S7] Fast and Robust 3D Gaussian Splatting for Virtual Reality, https://doi.org/10.1145/3681756.3697947, fetched 2026-10-30
