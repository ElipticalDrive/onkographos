# Tile-based Rasteriser

Also called: 3DGS GPU Rasterization, Radix Sort Binning, Tile-Based Rasterization, Tile-based Rasterization with Radix Sort, Tile-based Rasterizer, Tile-based sorting rasterizer

## Overview
The Tile-based Rasteriser is the GPU-optimized sorting and compositing component of the 3D Gaussian Splatting pipeline that assigns projected 2D Gaussian splats to image tiles and sorts them by depth using a single global radix sort [S1, S2, unofficial]. It solves the performance bottleneck of per-pixel depth sorting for millions of splats by grouping work into tiles, enabling real-time rendering and efficient backpropagation [S1, S2, unofficial]. Unlike non-tiled approaches that sort per pixel or per tile independently, it uses a unified 64-bit key to bin and sort all (splat, tile) pairs in one pass [S2, unofficial].

## How it works
**Inputs:**
- List of $N$ projected 2D Gaussian splats, each defined by center $(x_i, y_i)$, 2D covariance matrix $\Sigma'_i$, color $c_i$, opacity $\alpha_i$, and depth $Z_c$ [S1, S2, unofficial].
- Image resolution $(W, H)$ and tile dimensions $16 \times 16$ pixels [S1, S2, unofficial].

**Outputs:**
- A sorted list of (splat, tile) pairs, ordered by tile ID and then by depth $Z_c$ within each tile [S1, S2, unofficial].
- The final rendered image and gradients for training [S1, unofficial].

**Data Structures:**
- **Splat List:** Array of $N$ splats. Each splat stores its projected 2D parameters: position $(x_i, y_i)$, covariance $\Sigma'_i$, color $c_i$, opacity $\alpha_i$, and depth $Z_c$ [S1, S2, unofficial].
- **Tile Grid:** Implicit grid of tiles covering the image, with tile dimensions $T_w \times T_h = 16 \times 16$ pixels [S1, S2, unofficial].
- **Sort Key:** A 64-bit integer key for each (splat, tile) pair, packing the tile ID into the high bits and the splat's depth $Z_c$ into the low bits [S2, unofficial].
- **Sorted Pairs:** Array of (splat_index, tile_id, depth) tuples, ordered by the Sort Key [S2, unofficial].

**Steps:**
1. **Tile Assignment:** Assign each Gaussian to every tile its projected 2D splat touches [S1, unofficial]. If a splat straddles tile boundaries, it is duplicated into each relevant tile [S1, S2, unofficial].
2. **Key Packing:** For each (splat, tile) pair, pack the tile ID into the high bits and the splat's depth $Z_c$ into the low bits of a 64-bit integer key [S2, unofficial].
3. **Global Radix Sort:** Perform one GPU radix sort over all (splat, tile) pairs [S1, S2, unofficial]. This simultaneously bins splats by tile ID (contiguity) and sorts them by depth within each tile [S2, unofficial].
4. **Tile Processing:** Each tile is processed by one GPU thread block that walks the sorted list [S1, unofficial].
5. **Compositing:** For each pixel in a tile, run the front-to-back compositing loop using the tile's sorted list of splats, evaluating per-pixel alpha $\alpha_i(\mathbf{x})$ and breaking early when transmittance $T < \epsilon$ [S2, unofficial].
6. **Backpropagation:** Reuse the same sorted lists for training, allowing gradients to flow to position, shape, opacity, and color without additional sorting [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def tile_based_rasterization(splats, image_width, image_height, tile_width=16, tile_height=16, epsilon=1e-3):
    num_tiles_x = (image_width + tile_width - 1) // tile_width
    num_tiles_y = (image_height + tile_height - 1) // tile_height
    
    # Step 1 & 2: Tile Assignment and Pair Generation
    pairs = []
    for i, splat in enumerate(splats):
        # Determine tile coordinates overlapping the splat's 2D projection [G]
        tile_min_x = max(0, int((splat['x'] - 2 * splat['sigma_x']) / tile_width))
        tile_max_x = min(num_tiles_x - 1, int((splat['x'] + 2 * splat['sigma_x']) / tile_width))
        tile_min_y = max(0, int((splat['y'] - 2 * splat['sigma_y']) / tile_height))
        tile_max_y = min(num_tiles_y - 1, int((splat['y'] + 2 * splat['sigma_y']) / tile_height))
        
        for ty in range(tile_min_y, tile_max_y + 1):
            for tx in range(tile_min_x, tile_max_x + 1):
                tile_id = ty * num_tiles_x + tx
                pairs.append((i, tile_id, splat['Zc']))
                
    # Step 3: Key Packing and Global Radix Sort
    # Pack tile_id into high bits and depth into low bits [S2, unofficial]
    keys = [(tile_id << 32) | depth for i, tile_id, depth in pairs]
    sorted_indices = sorted(range(len(keys)), key=lambda k: keys[k])
    sorted_pairs = [pairs[i] for i in sorted_indices]
    
    # Step 4 & 5: Tile Processing and Compositing
    image = [[0.0, 0.0, 0.0] for _ in range(image_height)]
    for tile_id in range(num_tiles_x * num_tiles_y):
        ty = tile_id // num_tiles_x
        tx = tile_id % num_tiles_x
        y_start, y_end = ty * tile_height, min((ty + 1) * tile_height, image_height)
        x_start, x_end = tx * tile_width, min((tx + 1) * tile_width, image_width)
        
        # Extract contiguous segment for this tile from sorted list
        tile_segment = []
        for idx, (splat_idx, t_id, depth) in enumerate(sorted_pairs):
            if t_id == tile_id:
                tile_segment.append((splat_idx, depth))
            elif t_id > tile_id:
                break
                
        for y in range(y_start, y_end):
            for x in range(x_start, x_end):
                transmittance = 1.0
                color = [0.0, 0.0, 0.0]
                for splat_idx, _ in tile_segment:
                    splat = splats[splat_idx]
                    alpha = compute_gaussian_alpha(splat, x, y)
                    if alpha > 0:
                        color[0] += splat['color'][0] * alpha * transmittance
                        color[1] += splat['color'][1] * alpha * transmittance
                        color[2] += splat['color'][2] * alpha * transmittance
                        transmittance *= (1.0 - alpha)
                    if transmittance < epsilon:
                        break
                image[y][x] = color
    return image

def compute_gaussian_alpha(splat, x, y):
    # Standard 2D Gaussian evaluation at pixel coordinates [G]
    dx = x - splat['x']
    dy = y - splat['y']
    inv_det = 1.0 / (splat['sigma_xx'] * splat['sigma_yy'] - splat['sigma_xy']**2)
    exponent = -0.5 * inv_det * (splat['sigma_yy'] * dx**2 - 2 * splat['sigma_xy'] * dx * dy + splat['sigma_xx'] * dy**2)
    return splat['opacity'] * max(0.0, math.exp(exponent))
```

**Parameters:**
- Tile size: $16 \times 16$ pixels [S1, S2, unofficial].
- Early termination threshold: $\epsilon$ (implementation-defined, typically small) [S2, unofficial].

**Edge Cases and Pitfalls:**
- Sorting per pixel is too slow for millions of splats; tile-based sorting with a single global radix sort is required for real-time performance [S1, unofficial].
- A splat straddling tile boundaries must be duplicated into each relevant tile [S1, S2, unofficial].
- Precision issues in depth sorting may affect compositing order; ensure depth values are packed correctly into the sort key [S2, unofficial].

## Operating characteristics
- **Time Complexity:** One O(n) radix sort over all (splat, tile) pairs, where n is the number of (splat, tile) pairs [S2, unofficial]. This replaces thousands of per-tile sorts [S2, unofficial].
- **Memory Complexity:** O(n) for storing the (splat, tile) pairs and sorted indices [S2, unofficial].
- **Measured Performance:** On the Mip-NeRF360 benchmark, the method reaches 27.21 dB PSNR and 0.214 LPIPS at 134 fps, rendering roughly three orders of magnitude faster than previous quality leaders [S1, unofficial].
- **Scaling:** The method scales well with the number of splats due to the single global sort [S1, S2, unofficial].
- **Strengths:** Real-time performance, efficient use of GPU parallelism, shared sorting for rendering and training [S1, S2, unofficial].
- **Limitations:** Memory overhead for duplicating splats across tiles [S1, S2, unofficial].
- **When to Use:** Real-time rendering and training of 3D Gaussian Splatting [S1, unofficial].
- **When Not to Use:** Scenes with very few splats where the overhead of tile management outweighs the benefits [S2, unofficial].

## Background
The Tile-based Rasteriser was introduced as part of the 3D Gaussian Splatting pipeline by Kerbl et al. in 2023 [S1, unofficial]. It was designed to address the performance bottleneck of sorting millions of splats for real-time rendering and training [S1, unofficial]. The method leverages GPU-optimized radix sorting to achieve efficient per-tile depth ordering [S2, unofficial]. It is used in the 3D Gaussian Splatting rendering pipeline for novel-view synthesis [S1, unofficial].

## Sources
[S1] An Architectural Review of the 3D Gaussian Splatting Rendering Pipeline, https://doi.org/10.5281/zenodo.21739653, fetched 2026-10-26 (unofficial)
[S2] 3DGS-101/textbook/05-alpha-compositing.md at main - GitHub, https://github.com/Chamud/3DGS-101/blob/main/textbook/05-alpha-compositing.md, fetched 2026-10-28 (unofficial)
