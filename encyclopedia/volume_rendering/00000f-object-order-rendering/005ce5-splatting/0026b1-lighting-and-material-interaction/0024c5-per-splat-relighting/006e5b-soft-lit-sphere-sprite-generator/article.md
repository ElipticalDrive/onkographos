# Soft-Lit Sphere Sprite Generator

Also called: CPU Point Sprite Renderer, Rough-Plastic Sphere Sprite

## Overview
Generates a 2D RGBA sprite representing a softly lit sphere with a restrained rough-plastic highlight, optimized for CPU-based point-cloud rendering without external renderer dependencies [S1, unofficial]. It differs from GPU-accelerated splatting methods by providing a lightweight, self-contained procedural texture that approximates diffuse and rim lighting for individual points, serving as the visual primitive for the [Per-Splat Relighting](3371-per-splat-relighting.md) pipeline.

## How it works
**Inputs:**
- `radius`: Integer, sprite radius in pixels.
- `base_color`: Tuple of three integers `(red, green, blue)`, each in range `[0, 255]`.

**Outputs:**
- A 2D RGBA image (PIL Image or equivalent array) of size `(2 * radius + 1, 2 * radius + 1, 4)` with `uint8` data type.

**Data Structures:**
- The sprite is stored as a 2D grid of pixels. Each pixel contains four channels: Red, Green, Blue, and Alpha.
- Coordinates are normalized to the range `[-1, 1]` relative to the sprite center.

**Parameters:**
- `radius`: Clamped to a minimum of 2 (`radius = max(radius, 2)`) [S1, unofficial].
- Light direction: Fixed vector `[-0.48, -0.58, 0.66]` [S1, unofficial].
- Lighting weights: Ambient `0.64`, Diffuse `0.29`, Rim `0.07` [S1, unofficial].
- Highlight offset: `(0.32, 0.38)` [S1, unofficial].
- Highlight spread: `0.23` [S1, unofficial].
- Highlight intensity: `38.0` [S1, unofficial].

**Steps:**
1.  **Grid Generation**: Create a grid of coordinates `xx, yy` ranging from `-radius` to `radius`. Normalize these to `nx, ny` by dividing by `radius`.
2.  **Sphere Masking**: Compute `squared_radius = nx**2 + ny**2`. Determine if a pixel is inside the sphere: `inside = squared_radius <= 1.0`. For inside pixels, compute the z-component of the normal: `nz = sqrt(clip(1.0 - squared_radius, 0.0, 1.0))`. For outside pixels, `nz` is undefined but masked out later.
3.  **Lighting Calculation**:
    - Define light direction `light = [-0.48, -0.58, 0.66]` and normalize it to unit length.
    - Compute diffuse term: `diffuse = clip(nx*lx + ny*ly + nz*lz, 0.0, 1.0)`.
    - Compute rim term: `rim = clip(nz, 0.0, 1.0)`.
    - Combine: `illumination = 0.64 + 0.29 * diffuse + 0.07 * rim`.
4.  **Highlight Calculation**: Compute a 2D Gaussian highlight centered at `(-0.32, -0.38)` with standard deviation `0.23`:
    `highlight = exp(-(((nx + 0.32)/0.23)**2 + ((ny + 0.38)/0.23)**2))`.
5.  **Color Compositing**:
    - Scale base RGB by illumination: `rgb = base_color * illumination[..., None]`.
    - Add highlight: `rgb += 38.0 * highlight[..., None]`.
    - Clip RGB values to `[0, 255]`.
6.  **Alpha Generation**: Compute edge alpha: `edge = clip((1.03 - sqrt(squared_radius)) * radius, 0.0, 1.0)`. Set alpha channel to `edge * 255.0` for inside pixels, `0.0` for outside.
7.  **Output**: Concatenate RGB and Alpha channels, convert to `uint8`, and return as an image.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from PIL import Image

def generate_soft_lit_sphere_sprite(radius: int, base_color: tuple) -> Image.Image:
    # Clamp radius
    radius = max(radius, 2)
    
    # 1. Grid Generation
    # Create coordinates from -radius to radius
    x = np.arange(-radius, radius + 1)
    y = np.arange(-radius, radius + 1)
    xx, yy = np.meshgrid(x, y)
    
    # Normalize to [-1, 1]
    nx = xx / radius
    ny = yy / radius
    
    # 2. Sphere Masking
    squared_radius = nx**2 + ny**2
    inside = squared_radius <= 1.0
    
    # Compute nz for inside pixels
    nz = np.zeros_like(nx)
    nz[inside] = np.sqrt(np.clip(1.0 - squared_radius[inside], 0.0, 1.0))
    
    # 3. Lighting Calculation
    light = np.array([-0.48, -0.58, 0.66])
    light = light / np.linalg.norm(light)
    lx, ly, lz = light
    
    # Diffuse: dot product of normal (nx, ny, nz) and light
    diffuse = np.clip(nx * lx + ny * ly + nz * lz, 0.0, 1.0)
    
    # Rim: based on nz
    rim = np.clip(nz, 0.0, 1.0)
    
    # Combine lighting
    illumination = 0.64 + 0.29 * diffuse + 0.07 * rim
    
    # 4. Highlight Calculation
    # Gaussian highlight centered at (-0.32, -0.38)
    highlight = np.exp(-(((nx + 0.32) / 0.23)**2 + ((ny + 0.38) / 0.23)**2))
    
    # 5. Color Compositing
    # Base color is (R, G, B)
    base_r, base_g, base_b = base_color
    
    # Apply illumination to base color
    rgb_r = base_r * illumination
    rgb_g = base_g * illumination
    rgb_b = base_b * illumination
    
    # Add highlight
    rgb_r += 38.0 * highlight
    rgb_g += 38.0 * highlight
    rgb_b += 38.0 * highlight
    
    # Clip to [0, 255]
    rgb_r = np.clip(rgb_r, 0, 255)
    rgb_g = np.clip(rgb_g, 0, 255)
    rgb_b = np.clip(rgb_b, 0, 255)
    
    # 6. Alpha Generation
    # Edge alpha
    edge = np.clip((1.03 - np.sqrt(squared_radius)) * radius, 0.0, 1.0)
    alpha = edge * 255.0
    
    # Set alpha to 0 for outside pixels
    alpha[~inside] = 0.0
    
    # 7. Output
    # Convert to uint8
    rgb_r = rgb_r.astype(np.uint8)
    rgb_g = rgb_g.astype(np.uint8)
    rgb_b = rgb_b.astype(np.uint8)
    alpha = alpha.astype(np.uint8)
    
    # Stack channels: H x W x 4
    sprite = np.stack([rgb_r, rgb_g, rgb_b, alpha], axis=-1)
    
    return Image.fromarray(sprite, mode='RGBA')
```

**Edge Cases and Pitfalls:**
- **Radius < 2**: The algorithm clamps `radius` to 2 to ensure sufficient resolution for the highlight and edge effects [S1, unofficial].
- **Precision**: Floating-point operations should be performed in `float32` or higher to avoid quantization artifacts in the highlight and alpha gradients.
- **Boundary Conditions**: The `clip` function is critical for `nz` and `diffuse` to prevent NaNs or negative values when `squared_radius` slightly exceeds 1.0 due to floating-point error.

## Operating characteristics
- **Time Complexity**: $O(R^2)$ where $R$ is the sprite radius. The operations are element-wise and highly parallelizable, but the method is designed for CPU execution.
- **Memory Complexity**: $O(R^2)$ for the output sprite. No auxiliary data structures are required.
- **Performance**: Optimized for CPU-based rendering, avoiding the overhead of GPU kernel launches for simple point sprites. Suitable for real-time applications where GPU resources are constrained or for preprocessing assets.
- **Strengths**: No external renderer dependencies; lightweight; produces visually pleasing "rough-plastic" highlights that approximate complex lighting.
- **Limitations**: Fixed light direction; does not support dynamic lighting changes per-splat without regenerating the sprite; limited to spherical primitives.

## Background
The Soft-Lit Sphere Sprite Generator is part of the Gaussian-JEPA project, specifically used in the context of CPU-based point-cloud rendering [S1, unofficial]. It provides a procedural alternative to pre-baked textures or complex GPU-based shading for individual points in a scene.

## Sources
[S1] Amazingren/Gaussian-JEPA, https://github.com/Amazingren/Gaussian-JEPA, fetched 2026-10-27 (unofficial)
