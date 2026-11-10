# Tile-based Rasterization with Radix Sort

Also called: Tiled 2D Gaussian Splatting Rasterization, Tile-based Gaussian Rasterization

## Overview
This family implements the rendering pass for 3D Gaussian Splatting by dividing the image into a grid of tiles and using a global radix sort to order splats by depth within each tile. It solves the performance bottleneck of per-pixel sorting by ensuring that each tile is processed by a single GPU thread block walking a contiguous, depth-sorted list of splats. It sets itself apart from non-tiled methods by managing memory locality and enabling efficient early-termination culling, and from simple bounding-box assignment methods by providing a robust mechanism (radix sort) to handle splats straddling tile boundaries and to stabilize depth ordering for backpropagation.

## Shared principles
All methods in this family operate on a list of $N$ projected 2D Gaussian splats, where each splat $i$ is defined by its screen-space center $(x_i, y_i)$, 2D covariance matrix $\Sigma'_i$, color $c_i$, opacity $\alpha_i$, and view-space depth $Z_c$. The image is divided into a grid of tiles with dimensions $T_w \times T_h$ (typically $16 \times 16$ or $32 \times 32$ pixels).

The core mechanism consists of three phases:
1.  **Tile Assignment:** Each Gaussian is assigned to every tile its projected 2D splat touches. If a splat straddles tile boundaries, it is duplicated into each relevant tile, creating one work item per (Gaussian, tile) pair.
2.  **Key Packing:** For each (splat, tile) pair, a 64-bit integer sort key is constructed. The tile ID is packed into the high bits, and the splat's depth $Z_c$ is packed into the low bits. Specifically, the key is formed as:
    $$ \text{key} = (\text{tile\_id} \ll 32) \mid (\text{uint32\_t})Z_c $$
    where `tile_id` is the 1D tile index ($y \times \text{grid\_width} + x$) and $Z_c$ is cast to a 32-bit unsigned integer. The value associated with the key is the Gaussian index.
3.  **Global Radix Sort:** A single GPU radix sort is performed over all (splat, tile) pairs. This simultaneously bins splats by tile ID (ensuring contiguity) and sorts them by depth within each tile.

After sorting, each tile is processed by one GPU thread block that walks the sorted list. For each pixel in a tile, front-to-back compositing is performed using the tile's sorted list of splats, evaluating per-pixel alpha $\alpha_i(\mathbf{x})$ and breaking early when transmittance $T < \epsilon$. The same sorted lists are reused for backpropagation, allowing gradients to flow to position, shape, opacity, and color without additional sorting.

## Subtopics

### [Tile-Based Culling](2819-tile-based-culling.md)
Tile-Based Culling is a method for computing precise Gaussian-to-tile mappings in 3D Gaussian Splatting to cull irrelevant tiles during rasterization. It improves upon the standard conservative bounding box assignment by computing a tight, opacity-aware bounding box and iteratively examining candidate tiles to determine actual overlap. It sets itself apart from siblings like [Tile-based Rasteriser](2846-tile-based-rasteriser.md) by focusing on the pre-sorting culling phase to reduce the number of elements processed by the radix sort and subsequent rendering passes, though it incurs higher runtime cost in the preprocessing stage compared to non-iterative methods like AccuTile.

### [Tile-based Rasteriser](2846-tile-based-rasteriser.md)
The tile-based rasteriser is a GPU-optimized rendering component that sorts projected 2D Gaussian splats once per frame to serve both forward rendering and backpropagation. It solves the performance bottleneck of per-pixel sorting by dividing the image into tiles and using a single global radix sort to achieve per-tile depth ordering. Unlike sibling methods that may perform redundant depth sorting or evaluate splats against irrelevant pixels, this method ensures that each tile is processed by one GPU thread block walking a single sorted list, enabling real-time performance for millions of splats.

### [Tile-based Depth Sorting](8038-tile-based-depth-sorting.md)
Tile-based depth sorting mitigates "popping" artifacts in 3D Gaussian Splatting caused by inconsistent per-ray depth sorting when viewpoints rotate. It replaces per-ray sorting with tile-based sorting to ensure a consistent sorting order within a local region. This method builds on [Tile-based Rasterization with Radix Sort](7921-tile-based-rasterization-with-radix-sort.md) by applying its global sort mechanism specifically to stabilize view-dependent appearance.

### Gaussian Rasterization Key Generation
- **Definition:** Generates key-value pairs for all Gaussian-tile overlaps to enable sorting by tile ID and depth. Each Gaussian generates one key-value pair per tile it overlaps [S1, unofficial].
- **Steps:** For each Gaussian `idx`, if radius > 0, calculate the bounding rect `(rect_min, rect_max)` in tile coordinates based on image space center points_xy[idx] and radius radii[idx]. Iterate over tiles `y` in `[rect_min.y, rect_max.y)` and `x` in `[rect_min.x, rect_max.x)`. Construct key: `key = (y * grid.x + x) << 32 | (uint32_t)depths[idx]`. Store key at `gaussian_keys_unsorted[off]` and value (`idx`) at `gaussian_values_unsorted[off]`. Increment `off` [S1, unofficial].
- **Layout:** Key is a 64-bit integer (`uint64_t`). Upper 32 bits: 1D tile ID (`y * grid.x + x`). Lower 32 bits: camera coordinate depth (cast from float). Value is a 32-bit integer (`uint32_t`) representing the Gaussian index (`idx`) [S1, unofficial].

### Tile Range Identification
- **Definition:** Identifies the start and end indices in the sorted list of Gaussian instances for each tile. Runs once per instanced Gaussian ID after sorting [S1, unofficial].
- **Steps:** Read the tile ID from the current key: `currtile = key >> 32`. If `idx == 0`, set `ranges[currtile].x = 0`. If `idx > 0`, read `prevtile = point_list_keys[idx - 1] >> 32`. If `currtile != prevtile`, set `ranges[prevtile].y = idx` and `ranges[currtile].x = idx`. If `idx == L - 1`, set `ranges[currtile].y = L` [S1, unofficial].

## Background
The tile-based rasterization approach for 3D Gaussian Splatting was introduced as part of the original Gaussian Splatting paper by Kerbl et al. (2023) to enable real-time rendering of millions of splats on consumer GPUs. The method leverages the GPU's parallel sorting capabilities (e.g., via CUB or Thrust) to efficiently organize spatial data. It has become the standard rendering implementation for 3D Gaussian Splatting due to its balance of simplicity, performance, and differentiability.

## Sources
[S1] NorahCadeJS/GAI-GS, https://github.com/NorahCadeJS/GAI-GS, fetched 2026-10-27 (unofficial)
