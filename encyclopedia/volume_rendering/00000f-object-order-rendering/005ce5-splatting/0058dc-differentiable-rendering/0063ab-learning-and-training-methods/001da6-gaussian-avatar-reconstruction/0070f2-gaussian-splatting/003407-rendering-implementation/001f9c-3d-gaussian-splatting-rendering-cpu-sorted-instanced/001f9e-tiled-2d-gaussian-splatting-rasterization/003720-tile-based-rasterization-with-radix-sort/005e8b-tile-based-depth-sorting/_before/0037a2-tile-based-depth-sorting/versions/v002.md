# Tile-based Depth Sorting

Also called: StopThePop sorting, consistent depth sorting

## Overview
Tile-based depth sorting is a rasterization strategy designed to mitigate "popping" artifacts in surface-based volumetric representations (such as 2DGS and QGS) where Gaussian distributions are concentrated on surfaces. It addresses the sensitivity of alpha-blending order by replacing inconsistent per-ray depth sorting with a two-stage approach: coarse global ordering via per-tile depth approximation and fine local ordering via per-pixel resorting. This method sets itself apart from standard 3D Gaussian Splatting by enforcing strict consistency between forward rendering and backward gradient computation traversal orders, thereby eliminating streak-like and blocky artifacts introduced by single-ray approximations.

## How it works
The method replaces the standard per-ray depth sorting mechanism with tile-based sorting to ensure consistent sorting order within a local region [S1].

**Inputs and Outputs:**
*   **Input:** A list of $N$ projected 2D Gaussian splats, where each splat $i$ is defined by its screen-space center $(x_i, y_i)$, 2D covariance matrix $\Sigma'_i$, color $c_i$, opacity $\alpha_i$, and view-space depth $Z_c$.
*   **Output:** A sorted list of splats per tile, ordered by depth, suitable for front-to-back compositing.

**Data Structures:**
The image is divided into a grid of tiles with dimensions $T_w \times T_h$ (typically $16 \times 16$ or $32 \times 32$ pixels). For each (splat, tile) pair, a 64-bit integer sort key is constructed. The tile ID is packed into the high bits, and the splat's depth $Z_c$ is packed into the low bits. Specifically, the key is formed as:
$$ \text{key} = (\text{tile\_id} \ll 32) \mid (\text{uint32\_t})Z_c $$
where `tile_id` is the 1D tile index ($y \times \text{grid\_width} + x$) and $Z_c$ is cast to a 32-bit unsigned integer. The value associated with the key is the Gaussian index.

**Steps:**
1.  **Tile Assignment:** Each Gaussian is assigned to every tile its projected 2D splat touches. If a splat straddles tile boundaries, it is duplicated into each relevant tile, creating one work item per (Gaussian, tile) pair.
2.  **Key Packing:** For each (splat, tile) pair, construct the 64-bit sort key as defined above.
3.  **Global Radix Sort:** A single GPU radix sort is performed over all (splat, tile) pairs. This simultaneously bins splats by tile ID (ensuring contiguity) and sorts them by depth within each tile.
4.  **Rendering:** Each tile is processed by one GPU thread block that walks the sorted list. For each pixel in a tile, front-to-back compositing is performed using the tile's sorted list of splats, evaluating per-pixel alpha $\alpha_i(\mathbf{x})$ and breaking early when transmittance $T < \epsilon$.
5.  **Backpropagation:** The same sorted lists are reused for backpropagation, allowing gradients to flow to position, shape, opacity, and color without additional sorting.

**Edge Cases and Pitfalls:**
*   **Popping Artifacts:** Standard per-ray sorting can lead to inconsistent depth orders between adjacent pixels when viewpoints rotate, causing visible popping. Tile-based sorting mitigates this by enforcing local consistency [S1].
*   **Precision:** The casting of $Z_c$ to `uint32_t` assumes sufficient depth precision. If $Z_c$ exceeds the range representable by 32 bits, the sort order will be incorrect.

## Subtopics

### [Per-tile Sorting and Per-pixel Resorting](9984-per-tile-sorting-and-per-pixel-resorting.md)
A variant that further refines the sorting order to address popping artifacts in surface-based representations by combining coarse global ordering with fine local per-pixel resorting.

## Operating characteristics
*   **Time Complexity:** Dominated by the global radix sort, which is $O(N \log N)$ or $O(N)$ depending on the radix sort implementation, where $N$ is the number of splats.
*   **Memory Complexity:** Requires storage for the sorted list of (splat, tile) pairs.
*   **Strengths:** Eliminates popping artifacts caused by per-ray depth sorting inconsistencies. Ensures consistent sorting order within a local region.
*   **Limitations:** More complex than standard per-ray sorting. Requires careful handling of tile boundaries and splat duplication.
*   **When to Use:** When rendering surface-based volumetric representations (e.g., 2DGS, QGS) where popping artifacts are noticeable.
*   **When Not to Use:** For simple volumetric rendering where popping artifacts are not a concern, or when computational resources are severely limited.

## Background
The method was introduced in the context of 3D Gaussian Splatting to address the issue of "popping" 3D Gaussians, which occurs due to per-ray depth sorting inconsistencies when viewpoints rotate [S1]. The approach is part of the "StopThePop" technique, which aims to provide more faithful results by ensuring consistent sorting order at a local region [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
