# Blue-Red Colormap

Also called: Depth Normalization & Blue-Red Colormap, Depth Visualization, Depth Visualization via Min-Max Normalization, Jet-like Colormap

## Overview
The Blue-Red Colormap normalizes scalar depth values to the range $[0, 1]$ and maps them to a piecewise linear color gradient ranging from blue at minimum depth to red at maximum depth [S1, unofficial]. It solves the problem of interpreting 3D spatial structure from rendered depth buffers by providing a high-contrast continuous representation [S1, unofficial]. Unlike the parent Cosine Palette Color Mapping which generates smooth gradients via trigonometric functions, this method uses explicit linear piecewise functions optimized for depth visualization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A depth buffer containing raw depth values $d_{i,j}$ for each pixel $(i,j)$, and the global minimum ($d_{min}$) and maximum ($d_{max}$) depth values across the entire image [S1, unofficial][S2, unofficial].
*   **Output:** An RGB image where each pixel is colored according to its normalized depth value [S1, unofficial].

**Data Structures:**
*   `depth`: A 2D array or flattened vector of shape $(H, W)$ or $(H \times W)$ containing raw depth values [S2, unofficial].
*   `render_buffer`: A 2D array of shape $(H, W, 3)$ storing the final RGB output [S2, unofficial].
*   `min_v`, `max_v`: Scalar values representing the global minimum and maximum depth [S2, unofficial].

**Steps:**

1.  **Compute Global Extents:** Determine the minimum and maximum depth values across all pixels. Parallel implementations compute these using atomic operations across all rays [S2, unofficial].
    $$ d_{min} = \min_{i,j} (d_{i,j}) $$
    $$ d_{max} = \max_{i,j} (d_{i,j}) $$

2.  **Normalize Depth:** For each pixel, normalize the raw depth value to the range $[0, 1]$ using min-max normalization. A small epsilon ($\epsilon = 10^{-8}$) is added to the denominator to prevent division by zero in degenerate cases [S1, unofficial].
    $$ x = \frac{d_{i,j} - d_{min}}{d_{max} - d_{min} + \epsilon} $$

3.  **Compute Color Channels:** Calculate the Red ($R$), Green ($G$), and Blue ($B$) channels using piecewise linear functions of the normalized depth $x$ [S1, unofficial]:
    $$ R = \text{clip}(2x - 1, 0, 1) $$
    $$ G = \text{clip}(1 - |2x - 1|, 0, 1) $$
    $$ B = \text{clip}(1 - 2x, 0, 1) $$
    *   When $x=0$ (near), $R=0, G=0, B=1$ (Blue) [S1, unofficial].
    *   When $x=0.5$ (mid), $R=0, G=1, B=0$ (Green) [S1, unofficial].
    *   When $x=1$ (far), $R=1, G=0, B=0$ (Red) [S1, unofficial].

4.  **Finalize Output:** Stack the channels into an RGB vector, scale to $[0, 255]$, and cast to `uint8` for storage or display [S1, unofficial].
    $$ \text{Color}_{i,j} = \text{uint8}([R, G, B] \times 255) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def blue_red_colormap(depth_buffer):
    """
    Applies Blue-Red colormap to a depth buffer.
    
    Args:
        depth_buffer: 2D array of shape (height, width) with raw depth values.
        
    Returns:
        rgb_image: 2D array of shape (height, width, 3) with uint8 RGB values.
    """
    def clip(val, lo, hi):
        return max(lo, min(hi, val))

    # Step 1: Compute global min and max
    min_v = min(row for row in depth_buffer)
    max_v = max(row for row in depth_buffer)
    
    # Step 2: Normalize depth to [0, 1]
    epsilon = 1e-8
    normalized_depth = [[(depth_buffer[r][c] - min_v) / (max_v - min_v + epsilon) 
                         for c in range(len(depth_buffer[0]))] 
                        for r in range(len(depth_buffer))]
    
    # Step 3: Compute RGB channels
    height = len(normalized_depth)
    width = len(normalized_depth[0])
    rgb_image = [[[0.0, 0.0, 0.0] for _ in range(width)] for _ in range(height)]
    
    for r in range(height):
        for c in range(width):
            x = normalized_depth[r][c]
            # Red channel: increases linearly from 0.5 to 1.0
            red = clip(2.0 * x - 1.0, 0.0, 1.0)
            # Green channel: triangular shape, peak at 0.5
            green = clip(1.0 - abs(2.0 * x - 1.0), 0.0, 1.0)
            # Blue channel: decreases linearly from 1.0 to 0.0
            blue = clip(1.0 - 2.0 * x, 0.0, 1.0)
            rgb_image[r][c] = [red, green, blue]
            
    # Step 4: Stack and convert to uint8
    final_image = [[tuple(int(ch * 255) for ch in rgb_image[r][c]) 
                    for c in range(width)] for r in range(height)]
    
    return final_image
```

**Parameters:**
*   `epsilon`: Small constant for numerical stability, typically $10^{-8}$ [S1, unofficial].
*   `T_threshold`: Transparency threshold for ray marching termination, default $10^{-2}$ [S2, unofficial].
*   `max_samples_for_rendering`: Maximum samples per ray, default 100 [S2, unofficial].
*   `dist_to_focus`: Distance to focus for Depth of Field, default 1.2 [S2, unofficial].
*   `len_dis`: Lens size for Depth of Field, default 0.04 [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Variance:** If $d_{max} = d_{min}$, the denominator becomes zero. The epsilon term prevents division by zero [S1, unofficial].
*   **Out-of-Bounds Depth:** Depth values outside the $[d_{min}, d_{max}]$ range calculated from the buffer will be clipped to 0 or 1 by the `clip` function in the channel calculations, ensuring valid RGB output [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** High contrast between near and far objects; simple piecewise linear implementation [S1, unofficial].
*   **Limitations:** Not perceptually uniform; the green band in the middle may obscure subtle depth variations in the mid-range [S1, unofficial].
*   **When to use:** Visualizing depth maps where spatial structure needs to be quickly assessed [S1, unofficial].
*   **When not to use:** Not given in the sources.

## Background
The method is implemented in open-source repositories for 3D vision and depth visualization, such as `nerf-3d-vision` and `taichi-ngp-renderer` [S1, unofficial][S2, unofficial].

## Sources
[S1] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S2] Linyou/taichi-ngp-renderer, https://github.com/Linyou/taichi-ngp-renderer, fetched 2026-10-27 (unofficial)
