# Tile-based Rasterization with Radix Sort

Also called: Tiled Rasterization with Global Radix Sort, Tile-based Gaussian Rasterization

## Overview
Tile-based Rasterization with Radix Sort is a GPU-parallel rasterization method for 3D Gaussian Splatting that divides the image into tiles (typically $16 \times 16$ or $32 \times 32$ pixels) to manage memory and processing load, assigning Gaussians to tiles based on their projected 2D splats [S1, unofficial][S2, unofficial]. It solves the performance bottleneck of per-pixel sorting by using a single global radix sort to bin splats by tile and sort them by depth simultaneously [S2, unofficial]. This sets it apart from non-tiled methods by enabling thread-block-level parallelism per tile and from sibling sorting methods by relying on a global sort key rather than per-ray or local approximations.

## Shared principles
The method operates on a list of $N$ projected 2D Gaussian splats, each defined by center $(x_i, y_i)$, 2D covariance matrix $\Sigma'_i$, color $c_i$, opacity $\alpha_i$, and depth $Z_c$ [S1, unofficial][S2, unofficial]. The image is divided into a grid of tiles with dimensions $T_w \times T_h$ [S1, unofficial][S2, unofficial].

The core mechanism involves three phases:

1.  **Tile Assignment:** Each Gaussian is assigned to every tile its projected 2D splat touches. If a splat straddles tile boundaries, it is duplicated into each relevant tile [S1, unofficial][S2, unofficial]. This creates one work item per (Gaussian, tile) pair.
2.  **Key Packing:** For each (splat, tile) pair, a 64-bit integer sort key is constructed. The tile ID is packed into the high bits, and the splat's depth $Z_c$ is packed into the low bits [S2, unofficial]. Specifically, the key is formed as:
    $$ \text{key} = (\text{tile\_id} \ll 32) \mid (\text{uint32\_t})Z_c $$
    where `tile_id` is the 1D tile index ($y \times \text{grid\_width} + x$) and $Z_c$ is cast to a 32-bit unsigned integer [S1, unofficial][S2, unofficial].
3.  **Global Radix Sort:** A single GPU radix sort is performed over all (splat, tile) pairs [S1, unofficial][S2, unofficial]. This simultaneously bins splats by tile ID (ensuring contiguity) and sorts them by depth within each tile [S2, unofficial].

After sorting, each tile is processed by one GPU thread block that walks the sorted list [S1, unofficial]. For each pixel in a tile, front-to-back compositing is performed using the tile's sorted list of splats, evaluating per-pixel alpha $\alpha_i(\mathbf{x})$ and breaking early when transmittance $T < \epsilon$ [S2, unofficial]. The same sorted lists are reused for backpropagation, allowing gradients to flow to position, shape, opacity, and color without additional sorting [S1, unofficial].

## Subtopics

### [Opacity-Thresholded Tile Intersection Filtering](2109-opacity-thresholded-tile-intersection-filtering.md)
[Opacity-Thresholded Tile Intersection Filtering](2109-opacity-thresholded-tile-intersection-filtering.md) filters out 3D Gaussians during the tile intersection phase if their opacity falls below a specified threshold, treating their bounding box as effectively empty. It addresses the performance bottleneck caused by large, mostly transparent Gaussians that intersect many tiles but contribute negligible color. It sets itself apart from siblings by operating specifically on the opacity attribute during the initial culling stage rather than relying on depth, transmittance, or sorting optimizations.

### [Tile-Based Culling](2819-tile-based-culling.md)
[Tile-Based Culling](2819-tile-based-culling.md) computes a precise mapping between 3D Gaussians and image tiles to eliminate irrelevant tiles during rasterization. It solves the problem of overestimating the number of tiles a Gaussian intersects, which occurs when using standard axis-aligned bounding boxes (AABBs) that are overly conservative. Unlike the parent method, which assigns a Gaussian to every tile its projected splat touches (potentially including tiles with negligible opacity), Tile-Based Culling iteratively examines candidate tiles to verify actual overlap, thereby reducing the workload for the subsequent radix sort and rendering phases.

### [Tile-based Rasteriser](2846-tile-based-rasteriser.md)
[Tile-based Rasteriser](2846-tile-based-rasteriser.md) is a GPU-optimized rendering component that sorts projected 2D Gaussian splats once per frame to serve both forward rendering and backpropagation. It solves the performance bottleneck of per-pixel sorting by dividing the image into tiles and using a single global radix sort to achieve per-tile depth ordering. Unlike sibling methods that may perform redundant depth sorting or evaluate splats against irrelevant pixels, this method ensures that each tile is processed by one GPU thread block walking a single sorted list, enabling real-time performance for millions of splats.

### [Gaussian Rasterization Key Generation](5133-gaussian-rasterization-key-generation.md)
[Gaussian Rasterization Key Generation](5133-gaussian-rasterization-key-generation.md) generates the key-value pairs required for the tile-based rasterization of 3D Gaussians. It maps each 3D Gaussian to every tile it overlaps, creating one work item per (Gaussian, tile) pair. The keys pack the tile ID and depth to enable a single global radix sort that simultaneously bins splats by tile and sorts them by depth. It differs from [Tile-based Culling](2819-tile-based-culling.md) by generating explicit overlap entries rather than filtering, and from [Tile-based Depth Sorting](8038-tile-based-depth-sorting.md) by handling the key construction logic explicitly before the sort phase.

### [Tile-based Depth Sorting](8038-tile-based-depth-sorting.md)
[Tile-based Depth Sorting](8038-tile-based-depth-sorting.md) is a rasterization strategy designed to mitigate "popping" artifacts in surface-based volumetric representations (such as 2DGS and QGS) where Gaussian distributions are concentrated on surfaces. It addresses the sensitivity of alpha-blending order by replacing inconsistent per-ray depth sorting with a two-stage approach: coarse global ordering via per-tile depth approximation and fine local ordering via per-pixel resorting. This method sets itself apart from standard 3D Gaussian Splatting by enforcing strict consistency between forward rendering and backward gradient computation traversal orders, thereby eliminating streak-like and blocky artifacts introduced by single-ray approximations.

### Tile Range Identification
- [S1, unofficial] Identifies the start and end indices in the sorted list of Gaussian instances for each tile. Runs once per instanced Gaussian ID after sorting.
- [S1, unofficial] Read the tile ID from the current key: `currtile = key >> 32`.
- [S1, unofficial] If `idx == 0`, set `ranges[currtile].x = 0`.
- [S1, unofficial] If `idx > 0`, read `prevtile = point_list_keys[idx - 1] >> 32`. If `currtile != prevtile`, set `ranges[prevtile].y = idx` and `ranges[currtile].x = idx`.
- [S1, unofficial] If `idx == L - 1`, set `ranges[currtile].y = L`.

### FlashGS
- [S2, unofficial] A method focusing on accelerating the forward process by partitioning the screen into finer tiles and scheduling them on a multi-GPU pipeline.

## Background
The method builds on [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md), which introduced the tile-based memory management for 3D Gaussian Splatting. The specific use of global radix sort for tile-based depth ordering is described in unofficial implementations and surveys [S1, unofficial][S2, unofficial]. Variants like FlashGS extend this by optimizing tile scheduling for multi-GPU pipelines [S2, unofficial].

## Sources
[S1] NorahCadeJS/GAI-GS, https://github.com/NorahCadeJS/GAI-GS, fetched 2026-10-27 (unofficial)
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
