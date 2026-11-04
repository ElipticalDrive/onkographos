# Tile-Based Alpha Compositing

Also called: Tsplat compositing

## Overview
Tile-Based Alpha Compositing is a rendering technique that blends multiple semi-transparent primitives (such as 2D Gaussians or splats) into a final image by processing them in depth order from nearest to farthest within disjoint tiles [S1, unofficial]. It is distinguished from back-to-front compositing by its ability to use early termination: once the accumulated transmittance drops below a threshold, the pixel is considered fully opaque and remaining splats are skipped, providing a significant computational speedup [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of splats, each defined by a 2D center $(s_x, s_y)$, an inverse 2D covariance matrix $cov2d\_inv$, an opacity $\alpha_{splat}$, and a color $C_{splat}$.
*   **Output:** A framebuffer containing accumulated color $C_{accum}$ and accumulated opacity $\alpha_{accum}$ for each pixel.

**Data Structures:**
*   **Framebuffer:** A 2D array of structs or parallel arrays for $C_{accum}$ (RGB) and $\alpha_{accum}$ (float). Initialized to $(0, 0, 0)$ and $0$ respectively.
*   **Splat:** Struct containing:
    *   $s_x, s_y$: Float center coordinates in pixel space.
    *   $cov2d\_inv$: 2x2 matrix represented by elements $a, b, d$ where $cov2d\_inv = \begin{bmatrix} a & b \\ b & d \end{bmatrix}$. Note: The off-diagonal element is symmetric.
    *   $\alpha_{splat}$: Float opacity.
    *   $C_{splat}$: RGB color vector.

**Steps:**
1.  **Initialization:** Initialize the framebuffer's $C_{accum}$ to zero and $\alpha_{accum}$ to zero for all pixels.
2.  **Sorting:** Sort all splats by depth (distance from camera) in descending order (front-to-back).
3.  **Tile Processing:** Divide the image into disjoint tiles. For each tile:
    4.  **Splat Rasterization:** For each splat overlapping the tile:
        5.  **Bounding Box:** Compute the axis-aligned bounding box (AABB) of the splat's 2D Gaussian projection within the tile.
        6.  **Pixel Loop:** Iterate over each pixel $(p_x, p_y)$ within the AABB:
            7.  **Displacement:** Calculate $dx = p_x - s_x$ and $dy = p_y - s_y$.
            8.  **Optimization:** For a fixed row $p_y$, $dy$ is constant. Precompute coefficients for the quadratic form in $dx$ to optimize the inner loop [S1, unofficial].
            9.  **Gaussian Evaluation:** Compute the power term:
                $$ power = -\frac{1}{2} (a \cdot dx^2 + 2b \cdot dx \cdot dy + d \cdot dy^2) $$
                where $a = cov2d\_inv.x\_axis.x$, $b = cov2d\_inv.x\_axis.y$, $d = cov2d\_inv.y\_axis.y$ [S1, unofficial].
            10. **Alpha Calculation:** Compute the splat's alpha at this pixel:
                $$ \alpha = \alpha_{splat} \cdot \exp(power) $$
            11. **Clamping:** Clamp $\alpha$ to $[0, 1]$.
            12. **Compositing:** Update the framebuffer using front-to-back alpha blending:
                $$ T = 1 - \alpha_{accum} $$
                $$ contrib = T \cdot \alpha $$
                $$ C_{accum} += contrib \cdot C_{splat} $$
                $$ \alpha_{accum} += contrib $$
            13. **Early Termination:** If $\alpha_{accum} \ge 1.0$ (or a threshold close to 1.0, e.g., $0.99$), mark the pixel as opaque and skip further splats for this pixel [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tile_based_alpha_compositing(splats, width, height, tile_size=16):
    # Initialize framebuffer
    accum_color = [[(0.0, 0.0, 0.0) for _ in range(width)] for _ in range(height)]
    accum_alpha = [[0.0 for _ in range(width)] for _ in range(height)]
    
    # Sort splats front-to-back
    sorted_splats = sorted(splats, key=lambda s: s.depth, reverse=True)
    
    # Process tiles
    for ty in range(0, height, tile_size):
        for tx in range(0, width, tile_size):
            tile_end_y = min(ty + tile_size, height)
            tile_end_x = min(tx + tile_size, width)
            
            for splat in sorted_splats:
                # Check if splat overlaps tile
                if not splat_overlaps_tile(splat, tx, ty, tile_end_x, tile_end_y):
                    continue
                
                # Extract covariance elements
                a = splat.cov2d_inv[0][0]
                b = splat.cov2d_inv[0][1]
                d = splat.cov2d_inv[1][1]
                
                # Precompute row-constant terms for optimization [S1, unofficial]
                # For each row py in tile
                for py in range(ty, tile_end_y):
                    dy = py - splat.sy
                    # Coefficients for quadratic in dx: a*dx^2 + 2*b*dx*dy + d*dy^2
                    # Let term1 = d*dy^2, term2 = 2*b*dy
                    term1 = d * dy * dy
                    term2 = 2.0 * b * dy
                    
                    for px in range(tx, tile_end_x):
                        # Skip if pixel already opaque
                        if accum_alpha[py][px] >= 0.99:
                            continue
                            
                        dx = px - splat.sx
                        
                        # Compute power
                        power = -0.5 * (a * dx * dx + term2 * dx + term1)
                        
                        # Compute alpha
                        alpha = splat.opacity * math.exp(power)
                        alpha = max(0.0, min(1.0, alpha))
                        
                        if alpha < 1e-6:
                            continue
                            
                        # Compositing
                        T = 1.0 - accum_alpha[py][px]
                        contrib = T * alpha
                        
                        # Update color
                        r, g, b_val = accum_color[py][px]
                        cr, cg, cb = splat.color
                        accum_color[py][px] = (
                            r + contrib * cr,
                            g + contrib * cg,
                            b_val + contrib * cb
                        )
                        
                        # Update alpha
                        accum_alpha[py][px] += contrib
                        
                        # Early termination check
                        if accum_alpha[py][px] >= 0.99:
                            break # Break inner dx loop if pixel is opaque
```

## Subtopics

### [Front-to-Back Alpha Compositing](4213-front-to-back-alpha-compositing.md)
Front-to-Back Alpha Compositing is a rendering technique that blends multiple semi-transparent primitives (such as 2D Gaussians or splats) into a final image by processing them in depth order from nearest to farthest, ensuring correct occlusion where near primitives paint over far ones. It is distinguished from back-to-front compositing by its ability to use early termination: once the accumulated transmittance drops below a threshold, the pixel is considered fully opaque and remaining splats are skipped, providing a significant computational speedup.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K)$ where $N$ is the number of splats and $K$ is the average number of pixels per splat's bounding box. Early termination reduces the effective $K$ for opaque regions.
*   **Memory Complexity:** $O(W \cdot H)$ for the framebuffer, where $W, H$ are image dimensions.
*   **Strengths:** Parallelizable across tiles; no synchronization needed between tiles; early termination improves performance in opaque scenes.
*   **Limitations:** Requires sorting splats by depth; performance depends on splat density and overlap.
*   **When to use:** Real-time rendering of large numbers of semi-transparent primitives, especially on CPU or GPU where tile-based parallelism is beneficial.
*   **When not to use:** Scenes with very low opacity where early termination is ineffective; when strict back-to-front order is required for specific artistic effects.

## Background
Tile-Based Alpha Compositing is used in modern Gaussian Splatting renderers, such as Tsplat, to achieve real-time performance in terminal-based or lightweight environments [S1, unofficial]. The method leverages the spatial coherence of splats and the efficiency of tile-based processing to minimize overhead and maximize throughput.

## Sources
[S1] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26 (unofficial)
