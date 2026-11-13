# Tile-based Rasterization with Radix Sort

Also called: Tiled Radix Sort Rasterization, Global Radix Sort for GS

## Overview
Tile-based rasterization with radix sort is a GPU-parallel rendering method for 3D Gaussian Splatting that replaces per-pixel sorting with a single global radix sort to establish depth ordering per tile [S1][S2, unofficial]. It solves the performance bottleneck of evaluating millions of splats against every pixel by dividing the image into tiles (typically $16 \times 16$ or $32 \times 32$ pixels) and assigning each Gaussian to the tiles it overlaps, creating one work item per (Gaussian, tile) pair [S1][S2, unofficial]. Unlike [Non-Tiled 2D Gaussian Splatting with Transmittance Culling](2067-non-tiled-2d-gaussian-splatting-with-transmittance-culling.md) which processes pixels individually, or [Gaussian Splat Projection and Sorting](3369-gaussian-splat-projection-and-sorting.md) which may use less efficient sorting strategies, this method bins splats by tile ID and sorts them by depth in a single pass, enabling real-time performance for millions of splats by allowing one GPU thread block to process each tile's sorted list [S1][S2, unofficial].

## Shared principles
The method relies on the following mechanism, terms, and math common to all implementations in this family:

1.  **Tile Grid:** The image of width $W$ and height $H$ is divided into a grid of tiles of size $T_w \times T_h$ pixels (commonly $16 \times 16$ or $32 \times 32$) [S1][S2, unofficial]. The number of tiles in x and y directions is $W_{tiles} = \lceil W / T_w \rceil$ and $H_{tiles} = \lceil H / T_h \rceil$ [S1].
2.  **Tile Assignment:** Each projected 2D Gaussian $i$ with screen-space center $(x_i, y_i)$ and radius $r_i$ (derived from its 2D covariance matrix $\Sigma'_i$) is assigned to every tile it intersects [S1][S2, unofficial]. If a splat straddles tile boundaries, it is duplicated into each relevant tile, creating one work item per (Gaussian, tile) pair [S1][S2, unofficial].
3.  **Key Packing:** For each (splat, tile) pair, a 64-bit integer sort key is constructed [S1][S2, unofficial]. The key packs the 1D tile ID into the high 32 bits and the camera-space depth $Z_c$ into the low 32 bits:
    $$ \text{key} = (\text{tile\_id} \ll 32) \mid (\text{uint32\_t})Z_c $$
    where $\text{tile\_id} = y_{tile} \times W_{tiles} + x_{tile}$ and $Z_c$ is cast to a 32-bit unsigned integer [S1][S2, unofficial]. The value associated with the key is the original Gaussian index [S1].
4.  **Global Radix Sort:** A single parallel radix sort is performed over all generated keys [S1][S2, unofficial]. This simultaneously bins the work items by tile ID (ensuring contiguity) and sorts them by depth within each tile [S2, unofficial].
5.  **Tile Processing:** After sorting, each tile is processed by one GPU thread block that walks the contiguous sorted list of splats for that tile [S1]. For each pixel in the tile, front-to-back compositing is performed using the sorted splats, evaluating per-pixel alpha $\alpha_i(\mathbf{x})$ and breaking early when transmittance $T < \epsilon$ [S2, unofficial].
6.  **Reusability:** The same sorted lists are reused for backpropagation, allowing gradients to flow to position, shape, opacity, and color without additional sorting [S1].

## Subtopics

### [Tile-Based Culling](2819-tile-based-culling.md)
Computes a tight, opacity-aware bounding box and iteratively verifies overlap to reduce the number of (Gaussian, tile) pairs, solving the overestimation problem of standard AABBs.

### [Tile-based Rasteriser](2846-tile-based-rasteriser.md)
The core GPU-optimized component that performs the tile assignment, key packing, global radix sort, and per-tile rendering, solving the per-pixel sorting bottleneck.

### [Gaussian Rasterization Key Generation](5133-gaussian-rasterization-key-generation.md)
Generates the explicit key-value pairs for the sort, mapping Gaussians to overlapping tiles and packing the tile ID and depth into 64-bit integers.

### [Tile-based Depth Sorting](8038-tile-based-depth-sorting.md)
Mitigates "popping" artifacts in surface-based representations by enforcing strict consistency between forward rendering and backward gradient computation traversal orders via tile-based depth approximation.

### Opacity-Thresholded Tile Intersection Filtering
Filters out Gaussians whose opacity $\tau$ falls below an alpha-threshold $\epsilon$ immediately during tile intersection, treating their axis-aligned bounding box as empty [S1]. This reduces the number of duplicated Gaussians and subsequent $\alpha$-computations, particularly benefiting higher resolutions where large, transparent Gaussians would otherwise affect many tiles [S1]. Combining this with other modifications decreased rendering time on the Bicycle scene at native resolution by a factor of 3.77 and GPU memory usage by a factor of 2.11 compared to the reference [S1].

### Tile Range Identification
Identifies the start and end indices in the sorted list of Gaussian instances for each tile, running once per instanced Gaussian ID after sorting [S2, unofficial]. The algorithm reads the tile ID from the current key (`currtile = key >> 32`). If `idx == 0`, it sets `ranges[currtile].x = 0`. If `idx > 0`, it reads `prevtile = point_list_keys[idx - 1] >> 32`; if `currtile != prevtile`, it sets `ranges[prevtile].y = idx` and `ranges[currtile].x = idx`. Finally, if `idx == L - 1`, it sets `ranges[currtile].y = L` [S2, unofficial].

### FlashGS
A method focusing on accelerating the forward process by partitioning the screen into finer tiles and scheduling them on a multi-GPU pipeline [S3, unofficial].

## Background
The tile-based rasterization with radix sort approach is a foundational optimization in modern 3D Gaussian Splatting implementations, enabling real-time rendering of millions of splats. It was popularized by the original 3D Gaussian Splatting paper and subsequent high-performance implementations like gsplat [S3, unofficial]. The method addresses the $O(N \cdot P)$ complexity of naive splatting (where $N$ is the number of Gaussians and $P$ is the number of pixels) by reducing it to $O(N \log N)$ for sorting and $O(M)$ for rendering, where $M$ is the number of (Gaussian, tile) pairs [S1][S2, unofficial]. It is widely used in real-time viewers, training pipelines, and mobile deployments (e.g., Mobile-GS) due to its efficiency and compatibility with GPU architectures [S3, unofficial].

## Sources
[S1] Fast 3D Gaussian Splatting Rendering via Easily Integrable Improvements, https://doi.org/10.1109/lsp.2024.3521379, fetched 2026-10-26
[S2] NorahCadeJS/GAI-GS, https://github.com/NorahCadeJS/GAI-GS, fetched 2026-10-27 (unofficial)
[S3] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
