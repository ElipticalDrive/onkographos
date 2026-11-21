# Tile-based Rasterization with Radix Sort

Also called: Tiled 2D Gaussian Splatting Rasterization, Tile-based Gaussian Rasterization

## Overview
Tile-based rasterization with radix sort is a GPU-parallel rendering method for 3D Gaussian Splatting that divides the image into tiles to manage memory and processing load, assigning Gaussians to tiles based on their axis-aligned bounding boxes [Parent]. It sets itself apart from non-tiled alternatives by using a single global radix sort to bin splats by tile ID and sort them by depth simultaneously, enabling efficient per-tile processing by GPU thread blocks [S1, unofficial][S2, unofficial].

## Shared principles
The method operates on a list of $N$ projected 2D Gaussian splats, each defined by center $(x_i, y_i)$, 2D covariance matrix $\Sigma'_i$, color $c_i$, opacity $\alpha_i$, and view-space depth $Z_c$ [S1, unofficial][S2, unofficial]. The image is divided into a grid of tiles with dimensions $T_w \times T_h$ (typically $16 \times 16$ or $32 \times 32$ pixels) [S1, G, unofficial][S2, unofficial].

The core mechanism involves three phases:

1.  **Tile Assignment:** Each Gaussian is assigned to every tile its projected 2D splat touches. If a splat straddles tile boundaries, it is duplicated into each relevant tile [S1, unofficial][S2, unofficial]. This creates one work item per (Gaussian, tile) pair.
2.  **Key Packing:** For each (splat, tile) pair, a 64-bit integer sort key is constructed. The tile ID is packed into the high bits, and the splat's depth $Z_c$ is packed into the low bits [S2, unofficial]. Specifically, the key is formed as:
    $$ \text{key} = (\text{tile\_id} \ll 32) \mid (\text{uint32\_t})Z_c $$
    where `tile_id` is the 1D tile index ($y \times \text{grid\_width} + x$) and $Z_c$ is cast to a 32-bit unsigned integer [S1, unofficial][S2, unofficial].
3.  **Global Radix Sort:** A single GPU radix sort is performed over all (splat, tile) pairs [S1, unofficial][S2, unofficial]. This simultaneously bins splats by tile ID (ensuring contiguity) and sorts them by depth within each tile [S2, unofficial].

After sorting, each tile is processed by one GPU thread block that walks the sorted list [S1, unofficial]. For each pixel in a tile, front-to-back compositing is performed using the tile's sorted list of splats, evaluating per-pixel alpha $\alpha_i(\mathbf{x})$ and breaking early when transmittance $T < \epsilon$ [S2, unofficial]. The same sorted lists are reused for backpropagation, allowing gradients to flow to position, shape, opacity, and color without additional sorting [S1, unofficial].

## Subtopics

### [Tile-Based Culling](2819-tile-based-culling.md)
Tile-Based Culling is a method for computing precise Gaussian-to-tile mappings in 3D Gaussian Splatting to cull irrelevant tiles during rasterization. It improves upon the standard conservative bounding box assignment by computing a tight, opacity-aware bounding box and iteratively examining candidate tiles to determine actual overlap. It sets itself apart from siblings like [Tile-based Rasteriser](2846-tile-based-rasteriser.md) by focusing on the pre-sorting culling phase to reduce the number of elements processed by the radix sort and subsequent rendering passes, though it incurs higher runtime cost in the preprocessing stage compared to non-iterative methods like AccuTile.

### [Tile-based Rasteriser](2846-tile-based-rasteriser.md)
The tile-based rasteriser is a GPU-optimized rendering component that sorts projected 2D Gaussian splats once per frame to serve both forward rendering and backpropagation. It solves the performance bottleneck of per-pixel sorting by dividing the image into tiles and using a single global radix sort to achieve per-tile depth ordering. Unlike sibling methods that may perform redundant depth sorting or evaluate splats against irrelevant pixels, this method ensures that each tile is processed by one GPU thread block walking a single sorted list, enabling real-time performance for millions of splats.

### [Gaussian Rasterization Key Generation](5133-gaussian-rasterization-key-generation.md)
Generates key-value pairs for all Gaussian-tile overlaps to enable sorting by tile ID and depth. Each Gaussian generates one key-value pair per tile it overlaps.

### [Tile-based Depth Sorting](8038-tile-based-depth-sorting.md)
Tile-based depth sorting is a rasterization strategy designed to mitigate "popping" artifacts in surface-based volumetric representations (such as 2DGS and QGS) where Gaussian distributions are concentrated on surfaces. It addresses the sensitivity of alpha-blending order by replacing inconsistent per-ray depth sorting with a two-stage approach: coarse global ordering via per-tile depth approximation and fine local ordering via per-pixel resorting. This method sets itself apart from standard 3D Gaussian Splatting by enforcing strict consistency between forward rendering and backward gradient computation traversal orders, thereby eliminating streak-like and blocky artifacts introduced by single-ray approximations.

### Tile Range Identification
#### definition
- [S1, unofficial] Identifies the start and end indices in the sorted list of Gaussian instances for each tile. Runs once per instanced Gaussian ID after sorting.
#### step
- [S1, unofficial] Read the tile ID from the current key: currtile = key >> 32.
- [S1, unofficial] If idx == 0, set ranges[currtile].x = 0.
- [S1, unofficial] If idx > 0, read prevtile = point_list_keys[idx - 1] >> 32. If currtile != prevtile, set ranges[prevtile].y = idx and ranges[currtile].x = idx.
- [S1, unofficial] If idx == L - 1, set ranges[currtile].y = L.

## Background
The method is part of the 3D Gaussian Splatting rendering pipeline, specifically the CPU-sorted, instanced variant [Parent]. It builds on [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md).

## Sources
[S1] NorahCadeJS/GAI-GS, https://github.com/NorahCadeJS/GAI-GS, fetched 2026-10-27 (unofficial)
