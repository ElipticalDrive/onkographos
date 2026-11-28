# Adaptive Tile Sizing

Also called: Dynamic Tile Sizing

## Overview
Adaptive Tile Sizing is a strategy for selecting tile dimensions in tiled parallel compositing based on the total number of pixels to render, aiming to optimize performance by balancing memory access patterns and computational load [S1]. It differs from sibling methods like [Tile-Based Alpha Compositing](1215-tile-based-alpha-compositing.md) by treating tile dimensions as tunable parameters subject to empirical thresholds rather than fixed constants, addressing the trade-off where larger tiles reduce duplicated primitive processing but increase sequential load per tile [S1].

## How it works
**Inputs and Outputs:**
- Input: Total number of pixels to render ($N_{pix}$) [S1].
- Input: Memory indexing scheme (column-major or row-major) [S1].
- Output: Selected tile dimensions ($W \times H$) [G].

**Data Structures:**
- Tile dimensions are represented as a pair of integers $(W, H)$ [G].
- The method evaluates two candidate configurations:
  - Small tile: $16 \times 16$ pixels [S1].
  - Large tile: $16 \times 32$ pixels [S1].

**Parameters:**
- Threshold ($T$): 1,500,000 pixels [S1]. This value is determined experimentally [S1].

**Steps:**
1. Calculate the total number of pixels $N_{pix}$ to be rendered in the current frame or batch [G].
2. Compare $N_{pix}$ against the threshold $T$ [S1].
3. If $N_{pix} < T$, select the $16 \times 32$ tile configuration [S1].
4. If $N_{pix} \ge T$, select the $16 \times 16$ tile configuration [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_tile_dimensions(num_pixels: int, indexing: str) -> tuple[int, int]:
    """
    Selects tile dimensions based on pixel count and memory indexing.
    
    Args:
        num_pixels: Total number of pixels to render.
        indexing: Memory indexing scheme, either 'column-major' or 'row-major'.
        
    Returns:
        A tuple (width, height) representing the selected tile dimensions.
    """
    threshold = 1500000
    
    # Base selection based on pixel count threshold
    if num_pixels < threshold:
        width, height = 16, 32
    else:
        width, height = 16, 16
        
    # Adjust orientation based on memory indexing
    # Column-major: 16x32 is efficient (contiguous memory access along height)
    # Row-major: 32x16 is efficient (contiguous memory access along width)
    if indexing == 'row-major':
        # Swap dimensions to align the larger dimension with the row stride
        width, height = height, width
        
    return width, height
```

**Edge Cases and Pitfalls:**
- **Memory Indexing Dependency:** The performance advantage of $16 \times 32$ over $32 \times 16$ (or vice versa) is strictly dependent on memory indexing [S1]. In column-major indexing, $16 \times 32$ outperforms $32 \times 16$ because the larger dimension (32) aligns with the contiguous memory stride [S1]. In row-major indexing, this relationship is reversed [S1]. Implementers must swap dimensions if the rendering backend uses row-major storage [S1].
- **Resolution Sensitivity:** The threshold of 1.5 million pixels is specific to the hardware and implementation context described in the source [S1]. Blindly using $16 \times 16$ tiles for all resolutions can lead to suboptimal performance, as larger tiles are not always faster [S1].
- **Performance Degradation:** Using larger tiles (e.g., $16 \times 32$) on low-resolution scenes can increase render time [S1]. For example, on the Bicycle scene at low resolution, using $16 \times 32$ tiles increased average render time to 9.77 ms, demonstrating that larger tiles are not universally superior [S1].

## Subtopics

### [Image-Space Partitioning (Tile Size Optimization)](16582-image-space-partitioning-tile-size-optimization.md)
Image-Space Partitioning divides the output image into rectangular tiles to distribute ray-casting work among parallel threads, balancing the trade-off between mutex contention (small tiles) and load imbalance (large tiles). It differs from sibling methods by explicitly treating tile dimensions as tunable parameters subject to empirical sweep, rather than fixed constants, to identify performance "sweet spots" on specific hardware.

### Saliency-Based Adaptive Tiling
An adaptive tiling method based on hybrid saliency detection that divides each Group of Frames (GoF) into fine-grained uniform tiles, extracts static and dynamic saliency metrics, fuses them into a distribution map, and clusters tiles to group salient regions into larger tiles for higher quality representation [S2]. It divides each GoF into fine-grained uniform tiles [S2]. It extracts static saliency consisting of: 1) Intra-tile dispersion ($S_I$) capturing geometric features within tiles. 2) Inter-tile variability ($S_E$) capturing geometric features between tiles [S2]. It extracts dynamic saliency ($S_D$) by comparing matching tiles between the first and last frames to represent motion changes [S2]. It computes a weighted fusion of $S_I$, $S_E$, and $S_D$ to yield a comprehensive saliency distribution map for all tiles within a single GoF [S2]. It employs a clustering algorithm to group tiles with similar saliency values. Tiles with high saliency and proximal distances are prioritized and merged into larger tiles representing regions of greater user interest [S2].

## Operating characteristics
- **Time Complexity:** The selection logic is $O(1)$, involving a single comparison and potential dimension swap [G]. The impact on overall render time depends on the chosen tile size's efficiency in reducing duplicated Gaussian processing versus increasing sequential load per tile [S1].
- **Memory Complexity:** $O(1)$ for the selection logic [G]. The tile size choice affects the granularity of shared memory usage and cache efficiency, but no additional data structures are required for the selection itself [G].
- **Measured Performance:** On the Bicycle scene, using $16 \times 32$ tiles at low resolution resulted in an average render time of 9.77 ms, which was slower than the alternative, indicating that larger tiles are not always faster [S1]. The threshold of 1.5 million pixels was determined experimentally to optimize performance across varying resolutions [S1].
- **Strengths:** Adapts to resolution changes without manual tuning; accounts for memory indexing architecture; reduces duplicated Gaussian processing at higher resolutions [S1].
- **Limitations:** The threshold is hardware-specific and may not generalize across all architectures; requires knowledge of memory indexing scheme; may not account for scene complexity or content distribution [S1].
- **When to use:** When rendering 3D Gaussian Splatting (3DGS) or similar tile-based compositing workloads where resolution varies significantly and memory access patterns are critical [S1].

## Background
Adaptive Tile Sizing was introduced in the context of optimizing 3D Gaussian Splatting (3DGS) rendering performance [S1]. The method addresses the inefficiency of fixed tile sizes by empirically determining a pixel-count threshold that balances the benefits of larger tiles (reduced duplicate processing) against their costs (increased sequential load) [S1]. The specific threshold of 1.5 million pixels and the $16 \times 16$ vs. $16 \times 32$ configurations were derived from experiments on the Bicycle scene using a Julia-based implementation [S1]. The importance of memory indexing (column-major vs. row-major) was also highlighted as a critical factor in tile orientation performance [S1].

## Sources
[S1] Fast 3D Gaussian Splatting Rendering via Easily Integrable Improvements, https://doi.org/10.1109/lsp.2024.3521379, fetched 2026-10-26
[S2] Adaptive 3-D Gaussian Splatting Video Streaming, https://doi.org/10.1109/MNET.2025.3638049, fetched 2026-11-03
