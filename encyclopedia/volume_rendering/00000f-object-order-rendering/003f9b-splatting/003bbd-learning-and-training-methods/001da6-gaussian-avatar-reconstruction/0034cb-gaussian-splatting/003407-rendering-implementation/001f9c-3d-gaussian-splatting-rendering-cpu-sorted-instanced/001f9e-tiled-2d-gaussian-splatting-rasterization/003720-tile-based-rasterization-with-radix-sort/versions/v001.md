# Tile-based Rasterization with Radix Sort

Also called: Tiled 2D Gaussian Splatting Rasterization with Radix Sort, Global Radix Sort Rasterization

## Overview
Tile-based Rasterization with Radix Sort is a GPU-parallel rendering method for 3D Gaussian Splatting that assigns projected 2D splats to image tiles and sorts them by depth using a single global radix sort [S1, unofficial][S2]. It solves the performance bottleneck of per-pixel depth sorting for millions of splats by grouping work into tiles, enabling real-time rendering and efficient backpropagation [S1, unofficial][S2]. Unlike non-tiled approaches that sort per pixel or per tile independently, it uses a unified 64-bit key to bin and sort all (splat, tile) pairs in one pass [S2].

## Shared principles
The method operates on a list of $N$ projected 2D Gaussian splats, each defined by center $(x_i, y_i)$, 2D covariance matrix $\Sigma'_i$, color $c_i$, opacity $\alpha_i$, and depth $Z_c$ [S1, unofficial][S2]. The image is divided into a grid of tiles with dimensions $T_w \times T_h$ (typically $16 \times 16$ or $32 \times 32$ pixels) [S1, G, unofficial][S2].

The core mechanism involves three phases:
1.  **Tile Assignment:** Each Gaussian is assigned to every tile its projected 2D splat touches. If a splat straddles tile boundaries, it is duplicated into each relevant tile [S1, unofficial][S2].
2.  **Key Packing:** For each (splat, tile) pair, a 64-bit integer sort key is constructed. The tile ID is packed into the high bits, and the splat's depth $Z_c$ is packed into the low bits [S2]. Specifically, the key is formed as:
    $$ \text{key} = (\text{tile\_id} \ll 32) \mid (\text{uint32\_t})Z_c $$
    where `tile_id` is the 1D tile index ($y \times \text{grid\_width} + x$) and $Z_c$ is cast to a 32-bit unsigned integer [S1, unofficial][S2].
3.  **Global Radix Sort:** A single GPU radix sort is performed over all (splat, tile) pairs [S1, unofficial][S2]. This simultaneously bins splats by tile ID (ensuring contiguity) and sorts them by depth within each tile [S2].

After sorting, each tile is processed by one GPU thread block that walks the sorted list [S1, unofficial]. For each pixel in a tile, front-to-back compositing is performed using the tile's sorted list of splats, evaluating per-pixel alpha $\alpha_i(\mathbf{x})$ and breaking early when transmittance $T < \epsilon$ [S2]. The same sorted lists are reused for backpropagation, allowing gradients to flow to position, shape, opacity, and color without additional sorting [S1, unofficial].

## Subtopics

### [Tile-Based Culling](2819-tile-based-culling.md)
Tile-Based Culling computes a precise Gaussian-to-Tile mapping to cull irrelevant tiles during 3D Gaussian Splatting rasterization. It improves upon standard conservative bounding box assignment by computing tight, opacity-aware bounding boxes and iteratively verifying tile intersections. It is distinguished from [SnugBox](3378-snugbox.md) by its iterative tile examination process, which results in higher runtime costs for the culling phase compared to AccuTile, though it may show marginal downstream performance gains due to undercounting artifacts. See [Tile-Based Culling](2819-tile-based-culling.md).

### [Tile-based Rasteriser](2846-tile-based-rasteriser.md)
The Tile-based Rasteriser is the GPU-optimized sorting and compositing component of the 3D Gaussian Splatting pipeline that assigns projected 2D Gaussian splats to image tiles and sorts them by depth using a single global radix sort. It solves the performance bottleneck of per-pixel depth sorting for millions of splats by grouping work into tiles, enabling real-time rendering and efficient backpropagation. Unlike non-tiled approaches that sort per pixel or per tile independently, it uses a unified 64-bit key to bin and sort all (splat, tile) pairs in one pass. See [Tile-based Rasteriser](2846-tile-based-rasteriser.md).

### Gaussian Rasterization Key Generation
Generates key-value pairs for all Gaussian-tile overlaps to enable sorting by tile ID and depth. Each Gaussian generates one key-value pair per tile it overlaps [S1, unofficial]. For each Gaussian `idx`, if radius > 0, the bounding rect `(rect_min, rect_max)` in tile coordinates is calculated based on image space center points `points_xy[idx]` and radius `radii[idx]` [S1, unofficial]. The algorithm iterates over tiles `y` in `[rect_min.y, rect_max.y)` and `x` in `[rect_min.x, rect_max.x)` [S1, unofficial]. The key is constructed as `key = (y * grid.x + x) << 32 | (uint32_t)depths[idx]` and stored at `gaussian_keys_unsorted[off]`, while the value (Gaussian index `idx`) is stored at `gaussian_values_unsorted[off]` [S1, unofficial]. The key layout is a 64-bit integer (`uint64_t`) where the upper 32 bits are the 1D tile ID (`y * grid.x + x`) and the lower 32 bits are the camera coordinate depth (cast from float); the value is a 32-bit integer (`uint32_t`) representing the Gaussian index [S1, unofficial].

### Tile Range Identification
Identifies the start and end indices in the sorted list of Gaussian instances for each tile. This runs once per instanced Gaussian ID after sorting [S1, unofficial]. The algorithm reads the tile ID from the current key: `currtile = key >> 32` [S1, unofficial]. If `idx == 0`, it sets `ranges[currtile].x = 0` [S1, unofficial]. If `idx > 0`, it reads `prevtile = point_list_keys[idx - 1] >> 32`; if `currtile != prevtile`, it sets `ranges[prevtile].y = idx` and `ranges[currtile].x = idx` [S1, unofficial]. Finally, if `idx == L - 1`, it sets `ranges[currtile].y = L` [S1, unofficial].

### Tile-based Depth Sorting
A method to mitigate 'popping' artifacts caused by per-ray depth sorting inconsistencies when viewpoints rotate, by replacing per-ray sorting with tile-based sorting [S2]. It replaces the per-ray depth sorting mechanism with tile-based sorting to ensure consistent sorting order within a local region [S2].

## Background
The tile-based rasterization approach with global radix sort was introduced as part of the original 3D Gaussian Splatting implementation by Kerbl et al. (2023) [S1, unofficial]. It was designed to handle the massive number of splats (millions) efficiently on GPU hardware by leveraging parallel sorting primitives. The method has since been adopted as the standard rendering backend for 3DGS and its variants. The specific issue of "popping" artifacts due to per-ray sorting was later addressed by StopThePop [40], which proposed replacing per-ray sorting with tile-based sorting to ensure consistent sorting order at a local region [S2].

## Sources
[S1] NorahCadeJS/GAI-GS, https://github.com/NorahCadeJS/GAI-GS, fetched 2026-10-27 (unofficial)
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
