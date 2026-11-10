# Tile-based Rasterization with Radix Sort

Also called: Tiled 2D Gaussian Splatting Rasterization, Tile-based Gaussian Rasterization

## Overview
Tile-based rasterization with radix sort is a GPU-parallel rendering method for 3D Gaussian Splatting that divides the image into tiles to manage memory and processing load, assigning Gaussians to tiles based on their projected 2D splats and sorting them by depth within each tile using a global radix sort [Parent](2061-tiled-2d-gaussian-splatting-rasterization.md). It sets itself apart from non-tiled approaches by ensuring that each tile is processed by a single GPU thread block walking a sorted list, which enables real-time performance for millions of splats and consistent depth ordering for backpropagation [S1, unofficial][S2, unofficial].

## Shared principles
The method relies on three core mechanisms common to all implementations in this family:

1.  **Tile Assignment:** The image is divided into a grid of tiles with dimensions $T_w \times T_h$ (typically $16 \times 16$ or $32 \times 32$ pixels). Each projected 2D Gaussian splat is assigned to every tile its bounding box or ellipse intersects. If a splat straddles tile boundaries, it is duplicated into each relevant tile, creating one work item per (Gaussian, tile) pair [S1, unofficial][S2, unofficial].
2.  **Key Packing:** For each (splat, tile) pair, a 64-bit integer sort key is constructed. The tile ID is packed into the high 32 bits, and the splat's view-space depth $Z_c$ is packed into the low 32 bits. The key is formed as:
    $$ \text{key} = (\text{tile\_id} \ll 32) \mid (\text{uint32\_t})Z_c $$
    where `tile_id` is the 1D tile index ($y \times \text{grid\_width} + x$) and $Z_c$ is cast to a 32-bit unsigned integer [S1, unofficial][S2, unofficial]. The value associated with the key is the Gaussian index [S1, unofficial].
3.  **Global Radix Sort:** A single GPU radix sort is performed over all (splat, tile) pairs. This simultaneously bins splats by tile ID (ensuring contiguity) and sorts them by depth within each tile [S1, unofficial][S2, unofficial]. After sorting, each tile is processed by one GPU thread block that walks the sorted list, performing front-to-back compositing. The same sorted lists are reused for backpropagation, allowing gradients to flow to position, shape, opacity, and color without additional sorting [S1, unofficial][S2, unofficial].

## Subtopics

### [Tile-Based Culling](2819-tile-based-culling.md)
Tile-Based Culling is a method for computing precise Gaussian-to-tile mappings in 3D Gaussian Splatting to cull irrelevant tiles during rasterization. It improves upon the standard conservative bounding box assignment by computing a tight, opacity-aware bounding box and iteratively examining candidate tiles to determine actual overlap. It sets itself apart from siblings like [Tile-based Rasteriser](2846-tile-based-rasteriser.md) by focusing on the pre-sorting culling phase to reduce the number of elements processed by the radix sort and subsequent rendering passes, though it incurs higher runtime cost in the preprocessing stage compared to non-iterative methods like AccuTile.

### [Tile-based Rasteriser](2846-tile-based-rasteriser.md)
The tile-based rasteriser is a GPU-optimized rendering component that sorts projected 2D Gaussian splats once per frame to serve both forward rendering and backpropagation. It solves the performance bottleneck of per-pixel sorting by dividing the image into tiles and using a single global radix sort to achieve per-tile depth ordering. Unlike sibling methods that may perform redundant depth sorting or evaluate splats against irrelevant pixels, this method ensures that each tile is processed by one GPU thread block walking a single sorted list, enabling real-time performance for millions of splats.

### [Tile-based Depth Sorting](8038-tile-based-depth-sorting.md)
Tile-based depth sorting is a rasterization strategy designed to mitigate "popping" artifacts in surface-based volumetric representations (such as 2DGS and QGS) where Gaussian distributions are concentrated on surfaces. It addresses the sensitivity of alpha-blending order by replacing inconsistent per-ray depth sorting with a two-stage approach: coarse global ordering via per-tile depth approximation and fine local ordering via per-pixel resorting. This method sets itself apart from standard 3D Gaussian Splatting by enforcing strict consistency between forward rendering and backward gradient computation traversal orders, thereby eliminating streak-like and blocky artifacts introduced by single-ray approximations.

### Gaussian Rasterization Key Generation
Generates key-value pairs for all Gaussian-tile overlaps to enable sorting by tile ID and depth. Each Gaussian generates one key-value pair per tile it overlaps [S1, unofficial]. For each Gaussian `idx`, if radius > 0, calculate the bounding rect `(rect_min, rect_max)` in tile coordinates based on image space center points_xy[idx] and radius radii[idx] [S1, unofficial]. Iterate over tiles `y` in `[rect_min.y, rect_max.y)` and `x` in `[rect_min.x, rect_max.x)` [S1, unofficial]. Construct key: `key = (y * grid.x + x) << 32 | (uint32_t)depths[idx]` [S1, unofficial]. Store key at `gaussian_keys_unsorted[off]` and value (`idx`) at `gaussian_values_unsorted[off]` [S1, unofficial]. Increment `off` [S1, unofficial]. The key is a 64-bit integer (`uint64_t`) where upper 32 bits are the 1D tile ID (`y * grid.x + x`) and lower 32 bits are the camera coordinate depth (cast from float) [S1, unofficial]. The value is a 32-bit integer (`uint32_t`) representing the Gaussian index (`idx`) [S1, unofficial].

### Tile Range Identification
Identifies the start and end indices in the sorted list of Gaussian instances for each tile. Runs once per instanced Gaussian ID after sorting [S1, unofficial]. Read the tile ID from the current key: `currtile = key >> 32` [S1, unofficial]. If `idx == 0`, set `ranges[currtile].x = 0` [S1, unofficial]. If `idx > 0`, read `prevtile = point_list_keys[idx - 1] >> 32` [S1, unofficial]. If `currtile != prevtile`, set `ranges[prevtile].y = idx` and `ranges[currtile].x = idx` [S1, unofficial]. If `idx == L - 1`, set `ranges[currtile].y = L` [S1, unofficial].

## Background
The method originates from the 3D Gaussian Splatting framework by Kerbl et al. (2023), which introduced the concept of rasterizing 3D Gaussians as 2D splats [S1, unofficial]. The tile-based approach with radix sort was adopted to optimize the sorting step, which is a bottleneck in naive per-pixel sorting implementations. The specific implementation details, such as key packing and tile range identification, are derived from the open-source reference implementation and its variants [S1, unofficial]. The method is widely used in real-time 3D Gaussian Splatting renderers and neural avatar reconstruction systems.

## Sources
[S1] NorahCadeJS/GAI-GS, https://github.com/NorahCadeJS/GAI-GS, fetched 2026-10-27 (unofficial)
