# Tile-based Rasterization with Radix Sort

Also called: Tiled 2D Gaussian Splatting Rasterization with Global Radix Sort

## Overview
Tile-based Rasterization with Radix Sort is a GPU-parallel rasterization method for 3D Gaussian Splatting that divides the image into tiles to manage memory and processing load, assigning Gaussians to tiles based on their projected 2D splats [Parent]. It solves the performance bottleneck of per-pixel sorting by using a single global radix sort to achieve per-tile depth ordering, enabling real-time performance for millions of splats [Sibling: Tile-based Rasteriser]. Unlike sibling methods that may perform redundant depth sorting or evaluate splats against irrelevant pixels, this method ensures that each tile is processed by one GPU thread block walking a single sorted list [Sibling: Tile-based Rasteriser].

## Shared principles
The method operates on a list of $N$ projected 2D Gaussian splats, each defined by center $(x_i, y_i)$, 2D covariance matrix $\Sigma'_i$, color $c_i$, opacity $\alpha_i$, and depth $Z_c$ [Sibling: Tile-based Rasteriser]. The image is divided into a grid of tiles with dimensions $T_w \times T_h$ (typically $16 \times 16$ pixels) [Sibling: Tile-based Rasteriser].

The core mechanism involves three phases:

1.  **Tile Assignment:** Each Gaussian is assigned to every tile its projected 2D splat touches. If a splat straddles tile boundaries, it is duplicated into each relevant tile [Sibling: Tile-based Rasteriser]. This creates one work item per (Gaussian, tile) pair [Sibling: Tile-based Rasteriser].
2.  **Key Packing:** For each (splat, tile) pair, a 64-bit integer sort key is constructed. The tile ID is packed into the high bits, and the splat's depth $Z_c$ is packed into the low bits [Sibling: Tile-based Rasteriser]. Specifically, the key is formed as:
    $$ \text{key} = (\text{tile\_id} \ll 32) \mid (\text{uint32\_t})Z_c $$
    where `tile_id` is the 1D tile index ($y \times \text{grid\_width} + x$) and $Z_c$ is cast to a 32-bit unsigned integer [Sibling: Tile-based Rasteriser].
3.  **Global Radix Sort:** A single GPU radix sort is performed over all (splat, tile) pairs [Sibling: Tile-based Rasteriser]. This simultaneously bins splats by tile ID (ensuring contiguity) and sorts them by depth within each tile [Sibling: Tile-based Rasteriser].

After sorting, each tile is processed by one GPU thread block that walks the sorted list [Sibling: Tile-based Rasteriser]. For each pixel in a tile, front-to-back compositing is performed using the tile's sorted list of splats, evaluating per-pixel alpha $\alpha_i(\mathbf{x})$ and breaking early when transmittance $T < \epsilon$ [Sibling: Tile-based Rasteriser]. The same sorted lists are reused for backpropagation, allowing gradients to flow to position, shape, opacity, and color without additional sorting [Sibling: Tile-based Rasteriser].

## Subtopics

### [Tile-Based Culling](2819-tile-based-culling.md)
Computes a precise mapping between 3D Gaussians and image tiles to eliminate irrelevant tiles during rasterization. It solves the problem of overestimating the number of tiles a Gaussian intersects, which occurs when using standard axis-aligned bounding boxes (AABBs) that are overly conservative. Unlike the parent method, which assigns a Gaussian to every tile its projected splat touches (potentially including tiles with negligible opacity), Tile-Based Culling iteratively examines candidate tiles to verify actual overlap, thereby reducing the workload for the subsequent radix sort and rendering phases. Tile-Based Culling iterates over all candidate tiles while AccuTile does not, resulting in a markedly higher runtime cost for Tile-Based Culling despite warp-level load balancing.

### [Tile-based Rasteriser](2846-tile-based-rasteriser.md)
A GPU-optimized rendering component that sorts projected 2D Gaussian splats once per frame to serve both forward rendering and backpropagation. It solves the performance bottleneck of per-pixel sorting by dividing the image into tiles and using a single global radix sort to achieve per-tile depth ordering. Unlike sibling methods that may perform redundant depth sorting or evaluate splats against irrelevant pixels, this method ensures that each tile is processed by one GPU thread block walking a single sorted list, enabling real-time performance for millions of splats.

### [Gaussian Rasterization Key Generation](5133-gaussian-rasterization-key-generation.md)
Generates the key-value pairs required for tile-based depth sorting in 3D Gaussian Splatting. It identifies which tiles each projected 2D Gaussian overlaps and constructs a 64-bit sort key combining the tile ID and the Gaussian's depth. This output serves as the direct input for the global radix sort performed by the parent method. It differs from sibling methods like [Tile-Based Culling](2819-tile-based-culling.md) by explicitly generating the full set of overlaps for sorting rather than just filtering, and from [Tile-based Depth Sorting](8038-tile-based-depth-sorting.md) by focusing on the key construction phase rather than the sorting algorithm itself.

### [FlashGS](7331-flashgs.md)
A multi-GPU rendering and training method for 3D Gaussian Splatting that accelerates the forward pass by partitioning the screen into finer tiles and scheduling them across a multi-GPU pipeline. It differs from single-GPU rasterizers by distributing the tile workload to enable scalability for large scenes, and from primitive-sharding approaches by focusing on screen-space partitioning rather than model-space partitioning.

### [Tile-based Depth Sorting](8038-tile-based-depth-sorting.md)
A rasterization strategy designed to mitigate "popping" artifacts in surface-based volumetric representations (such as 2DGS and QGS) where Gaussian distributions are concentrated on surfaces. It addresses the sensitivity of alpha-blending order by replacing inconsistent per-ray depth sorting with a two-stage approach: coarse global ordering via per-tile depth approximation and fine local ordering via per-pixel resorting. This method sets itself apart from standard 3D Gaussian Splatting by enforcing strict consistency between forward rendering and backward gradient computation traversal orders, thereby eliminating streak-like and blocky artifacts introduced by single-ray approximations.

### Opacity-Thresholded Tile Intersection Filtering
- **Step**: Filter out Gaussians whose opacity $\tau$ falls below the alpha-threshold $\epsilon$ immediately during tile intersection, as their axis-aligned bounding box is considered empty [S1].
- **Parameter**: Alpha-threshold: $\epsilon$. Gaussians with opacity $\tau < \epsilon$ are filtered [S1].
- **Number**: Combining all modifications decreased rendering time on the Bicycle scene at native resolution by a factor of 3.77 and GPU memory usage by a factor of 2.11 compared to the reference [S1].

### Tile Range Identification
- **Definition**: Identifies the start and end indices in the sorted list of Gaussian instances for each tile. Runs once per instanced Gaussian ID after sorting [S2, unofficial].
- **Step**: Read the tile ID from the current key: `currtile = key >> 32`. If `idx == 0`, set `ranges[currtile].x = 0`. If `idx > 0`, read `prevtile = point_list_keys[idx - 1] >> 32`. If `currtile != prevtile`, set `ranges[prevtile].y = idx` and `ranges[currtile].x = idx`. If `idx == L - 1`, set `ranges[currtile].y = L` [S2, unofficial].

## Background
The method builds on [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md). The specific optimizations regarding opacity-thresholded filtering and performance gains are detailed in [S1]. The implementation details and unofficial variations are documented in [S2, unofficial].

## Sources
[S1] Fast 3D Gaussian Splatting Rendering via Easily Integrable Improvements, https://doi.org/10.1109/lsp.2024.3521379, fetched 2026-10-26
[S2] NorahCadeJS/GAI-GS, https://github.com/NorahCadeJS/GAI-GS, fetched 2026-10-27 (unofficial)
