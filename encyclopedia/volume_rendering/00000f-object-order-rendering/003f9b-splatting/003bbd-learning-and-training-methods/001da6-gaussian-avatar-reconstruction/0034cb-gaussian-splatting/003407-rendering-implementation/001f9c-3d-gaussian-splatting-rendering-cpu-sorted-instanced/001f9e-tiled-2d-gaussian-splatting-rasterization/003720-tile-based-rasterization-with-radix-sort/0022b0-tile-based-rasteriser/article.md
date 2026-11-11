# Tile-based Rasteriser

Also called: 3DGS GPU Rasterization, Radix Sort Binning, Tile-based Rasterizer, Tile-based sorting rasterizer

## Overview
The tile-based rasteriser is a GPU-optimized rendering component that sorts projected 2D Gaussian splats once per frame to serve both forward rendering and backpropagation [S1, unofficial][S2, unofficial]. It solves the performance bottleneck of per-pixel sorting by dividing the image into tiles and using a single global radix sort to achieve per-tile depth ordering [S2, unofficial]. Unlike sibling methods that may perform redundant depth sorting or evaluate splats against irrelevant pixels, this method ensures that each tile is processed by one GPU thread block walking a single sorted list, enabling real-time performance for millions of splats [S1, unofficial].

## How it works
The method operates on a list of $N$ projected 2D Gaussian splats, each defined by center $(x_i, y_i)$, 2D covariance matrix $\Sigma'_i$, color $c_i$, opacity $\alpha_i$, and depth $Z_c$ [S1, unofficial][S2, unofficial]. The image is divided into a grid of tiles with dimensions $T_w \times T_h$ (typically $16 \times 16$ pixels) [S1, unofficial][S2, unofficial].

The core mechanism involves three phases:

1.  **Tile Assignment:** Each Gaussian is assigned to every tile its projected 2D splat touches. If a splat straddles tile boundaries, it is duplicated into each relevant tile [S1, unofficial][S2, unofficial]. This creates one work item per (Gaussian, tile) pair [S1, unofficial].
2.  **Key Packing:** For each (splat, tile) pair, a 64-bit integer sort key is constructed. The tile ID is packed into the high bits, and the splat's depth $Z_c$ is packed into the low bits [S2, unofficial]. Specifically, the key is formed as:
    $$ \text{key} = (\text{tile\_id} \ll 32) \mid (\text{uint32\_t})Z_c $$
    where `tile_id` is the 1D tile index ($y \times \text{grid\_width} + x$) and $Z_c$ is cast to a 32-bit unsigned integer [S1, unofficial][S2, unofficial].
3.  **Global Radix Sort:** A single GPU radix sort is performed over all (splat, tile) pairs [S1, unofficial][S2, unofficial]. This simultaneously bins splats by tile ID (ensuring contiguity) and sorts them by depth within each tile [S2, unofficial].

After sorting, each tile is processed by one GPU thread block that walks the sorted list [S1, unofficial]. For each pixel in a tile, front-to-back compositing is performed using the tile's sorted list of splats, evaluating per-pixel alpha $\alpha_i(\mathbf{x})$ and breaking early when transmittance $T < \epsilon$ [S2, unofficial]. The same sorted lists are reused for backpropagation, allowing gradients to flow to position, shape, opacity, and color without additional sorting [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def tile_based_rasterization(splats, image_width, image_height, tile_size=16):
    """
    Performs tile-based rasterization for 3D Gaussian Splatting.
    
    Args:
        splats: List of N splats, each with attributes:
            - center: (x, y) in screen space
            - cov2d: 2x2 covariance matrix [[a, b], [b, c]]
            - color: (r, g, b) tuple
            - opacity: scalar alpha
            - depth: Z_c (view space depth)
        image_width: Width of the image in pixels
        image_height: Height of the image in pixels
        tile_size: Size of tiles in pixels (default 16)
        
    Returns:
        rendered_image: 2D list of (r, g, b) tuples
    """
    # 1. Tile Assignment
    tile_width = (image_width + tile_size - 1) // tile_size
    tile_height = (image_height + tile_size - 1) // tile_size
    
    work_items = []
    
    for i, splat in enumerate(splats):
        covered_tiles = get_tiles_covered_by_splat(splat, image_width, image_height, tile_size)
        for tile_id in covered_tiles:
            work_items.append({
                'splat_index': i,
                'tile_id': tile_id,
                'depth': splat['depth']
            })
            
    if not work_items:
        return [[(0.0, 0.0, 0.0) for _ in range(image_width)] for _ in range(image_height)]

    # 2. Key Packing
    keys = []
    for item in work_items:
        key = ((item['tile_id'] << 32) | (item['depth'] & 0xFFFFFFFF))
        keys.append(key)
        
    # 3. Global Radix Sort
    # Sort indices by keys. In GPU implementations this is a parallel radix sort.
    sorted_indices = sorted(range(len(keys)), key=lambda i: keys[i])
    sorted_work_items = [work_items[i] for i in sorted_indices]
    
    # 4. Process Tiles
    rendered_image = [[(0.0, 0.0, 0.0) for _ in range(image_width)] for _ in range(image_height)]
    
    current_tile_id = -1
    current_tile_items = []
    
    for item in sorted_work_items:
        if item['tile_id'] != current_tile_id:
            if current_tile_id != -1:
                render_tile(rendered_image, current_tile_id, current_tile_items, 
                            splats, tile_size, image_width, image_height)
            current_tile_id = item['tile_id']
            current_tile_items = []
        current_tile_items.append(item)
        
    if current_tile_id != -1:
        render_tile(rendered_image, current_tile_id, current_tile_items, 
                    splats, tile_size, image_width, image_height)
                    
    return rendered_image

def render_tile(image, tile_id, tile_items, splats, tile_size, img_w, img_h):
    tile_x = tile_id % ((img_w + tile_size - 1) // tile_size)
    tile_y = tile_id // ((img_w + tile_size - 1) // tile_size)
    
    x_start = tile_x * tile_size
    y_start = tile_y * tile_size
    x_end = min(x_start + tile_size, img_w)
    y_end = min(y_start + tile_size, img_h)
    
    transmittance = [[1.0 for _ in range(x_end - x_start)] for _ in range(y_end - y_start)]
    color_buffer = [[(0.0, 0.0, 0.0) for _ in range(x_end - x_start)] for _ in range(y_end - y_start)]
    
    for item in tile_items:
        splat = splats[item['splat_index']]
        for dy in range(y_end - y_start):
            for dx in range(x_end - x_start):
                T = transmittance[dy][dx]
                if T < 1e-4:
                    continue
                    
                alpha = evaluate_gaussian_alpha(splat, x_start + dx, y_start + dy)
                if alpha > 0.0:
                    r, g, b = splat['color']
                    color_buffer[dy][dx] = (
                        color_buffer[dy][dx][0] + alpha * r * T,
                        color_buffer[dy][dx][1] + alpha * g * T,
                        color_buffer[dy][dx][2] + alpha * b * T
                    )
                    transmittance[dy][dx] *= (1.0 - alpha)
                    
    for dy in range(y_end - y_start):
        for dx in range(x_end - x_start):
            image[y_start + dy][x_start + dx] = color_buffer[dy][dx]

def get_tiles_covered_by_splat(splat, img_w, img_h, tile_size):
    x_min, y_min, x_max, y_max = get_splat_bbox(splat)
    tile_w = (img_w + tile_size - 1) // tile_size
    tile_h = (img_h + tile_size - 1) // tile_size
    
    tile_x_start = max(0, int(math.floor(x_min / tile_size)))
    tile_x_end = min(tile_w, int(math.ceil(x_max / tile_size)))
    tile_y_start = max(0, int(math.floor(y_min / tile_size)))
    tile_y_end = min(tile_h, int(math.ceil(y_max / tile_size)))
    
    tiles = []
    for ty in range(tile_y_start, tile_y_end):
        for tx in range(tile_x_start, tile_x_end):
            tiles.append(ty * tile_w + tx)
    return tiles

def get_splat_bbox(splat):
    # Approximate 2D Gaussian bounds using 3-sigma ellipse axes
    # For simplicity, returns a conservative AABB around the splat center
    # In practice, compute eigenvalues of Sigma' to get axis lengths
    center_x, center_y = splat['center']
    # Placeholder for exact 3-sigma ellipse bounding box calculation [G]
    return (center_x - 3.0, center_y - 3.0, center_x + 3.0, center_y + 3.0)

def evaluate_gaussian_alpha(splat, px, py):
    dx = px - splat['center'][0]
    dy = py - splat['center'][1]
    quad = quadratic_form(dx, dy, splat['cov2d'])
    return splat['opacity'] * math.exp(-0.5 * quad)

def quadratic_form(dx, dy, cov2d):
    a, b, c = cov2d[0][0], cov2d[0][1], cov2d[1][1]
    det = a * c - b * b
    if det == 0:
        return 0.0
    inv_det = 1.0 / det
    # [dx, dy] * inv(Sigma') * [dx, dy]^T
    return (c * dx * dx - 2 * b * dx * dy + a * dy * dy) * inv_det
```

## Operating characteristics
- **Time Complexity:** One $O(n)$ radix sort over all (splat, tile) pairs replaces thousands of per-tile sorts [S2, unofficial].
- **Memory Complexity:** Requires storage for the work items and sorted keys. The number of work items is proportional to the number of splats times the average number of tiles each splat overlaps (typically small, e.g., ~4 for a $30 \times 30$ pixel splat) [S2, unofficial].
- **Measured Performance:** On the Mip-NeRF360 benchmark, the method reaches 27.21 dB PSNR and 0.214 LPIPS at 134 fps, surpassing previous quality leaders in perceptual similarity while rendering roughly three orders of magnitude faster [S1, unofficial].
- **Strengths:** Real-time performance for millions of splats; single sort serves both rendering and training; efficient use of GPU parallelism via tile-based processing [S1, unofficial].
- **Limitations:** Sorting per pixel is too slow for millions of splats; tile-based sorting is required for real-time performance [S1, unofficial].
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
The tile-based rasteriser was introduced as part of the 3D Gaussian Splatting pipeline, which includes anisotropic 3D Gaussians projected to 2D elliptical splats on the image plane (EWA projection), sorted by depth and alpha-blended front to back [S1, unofficial]. The method was developed to enable real-time rendering and training of 3D Gaussian Splatting, addressing the bottleneck of per-pixel sorting [S1, unofficial]. It is used in novel-view synthesis for virtual and augmented reality, digital twins, robotics simulation, and film production [S1, unofficial].

## Sources
[S1] An Architectural Review of the 3D Gaussian Splatting Rendering Pipeline, https://doi.org/10.5281/zenodo.21739653, fetched 2026-10-26 (unofficial)
[S2] 3DGS-101/textbook/05-alpha-compositing.md at main - GitHub, https://github.com/Chamud/3DGS-101/blob/main/textbook/05-alpha-compositing.md, fetched 2026-10-28 (unofficial)
