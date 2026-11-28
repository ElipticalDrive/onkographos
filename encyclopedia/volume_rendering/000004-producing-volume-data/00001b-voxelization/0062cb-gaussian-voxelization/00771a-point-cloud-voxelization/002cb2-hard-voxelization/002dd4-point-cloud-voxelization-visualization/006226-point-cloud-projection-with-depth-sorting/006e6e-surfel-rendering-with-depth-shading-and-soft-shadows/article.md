# Surfel Rendering with Depth Shading and Soft Shadows

Also called: CPU Surfel Renderer, Point Cloud Visualization Pipeline

## Overview
Renders a 3D point cloud as 2D surfels using per-point sprites, applies depth-based color shading, and composites a soft projected floor shadow to provide visual grounding without altering point locations or semantic colors [S1, unofficial]. It builds on [Point Cloud Projection with Depth Sorting](18548-point-cloud-projection-with-depth-sorting.md) by adding explicit shadow compositing and depth-aware sprite scaling, distinguishing itself from [Orthographic Voxelization and Projection](18550-orthographic-voxelization-and-projection.md) through its use of perspective-scaled soft shadows and continuous depth shading rather than hard voxel boundaries.

## How it works
**Inputs and Outputs:**
*   **Inputs:** `points` array of shape `[N, 3]` (float32), `labels` array of shape `[N]` (int64), `resolution` (int, default 1000), `point_radius` (float, default 16.0), `azimuth` (float, degrees), `elevation` (float, degrees), `transparent` (bool, default False).
*   **Outputs:** RGBA image array of shape `[size, size, 4]` (uint8), where `size` is the resolution.

**Data Structures:**
*   `camera`: `[N, 3]` float32, rotated point coordinates.
*   `xy`: `[N, 2]` float32, projected 2D coordinates.
*   `depth`: `[N]` float32, depth values for sorting.
*   `depth_norm`: `[N]` float32, normalized depth values in range [0, 1].
*   `order`: `[N]` int64, indices sorting points by depth.

**Steps:**

1.  **Normalization and Rotation:**
    Compute the bounding box span of the input points: `span = points.max(axis=0) - points.min(axis=0)`. Clamp `span` to a minimum of `1e-6` to prevent division by zero [S1, unofficial]. Calculate the scaling factor: `image_scale = 0.82 * size / float(span.max())`. Apply a rotation matrix constructed from `azimuth` and `elevation` angles to the points to obtain `camera` coordinates [S1, unofficial].

2.  **Projection and Centering:**
    Project the rotated points to 2D by taking the first two coordinates of `camera` to form `xy`. Center the points by subtracting the mean of `xy`. Scale the centered points by `image_scale` [S1, unofficial].

3.  **Depth Sorting:**
    Extract the depth values from the third coordinate of `camera` into `depth`. Compute `depth_norm` by normalizing `depth` to the range [0, 1]. Use the formula: `depth_norm = (depth - depth.min()) / max(depth.max() - depth.min(), 1e-6)` [S1, unofficial]. Generate `order` by sorting indices based on `depth` in ascending order [S1, unofficial].

4.  **Shadow Layer Initialization:**
    Create a blank RGBA canvas for the shadow layer. Determine the ground plane y-coordinate: `ground_y = min(size * 0.95, float(y.max()) + 0.025 * size)` [S1, unofficial]. Draw a base floor shadow ellipse at `(ground_y, 0)` with a radius proportional to the scene size [S1, unofficial].

5.  **Shadow Compositing:**
    Iterate through points in the order defined by `order`. For each point at index `i`:
    *   Compute shadow offset: `shadow_x = float(xy[i, 0]) + 0.11 * height` and `shadow_y = ground_y + 0.018 * height` [S1, unofficial].
    *   Compute perspective-scaled shadow radius: `shadow_radius_x = point_radius * (0.78 + 0.08 * float(depth_norm[i]))` and `shadow_radius_y = 0.30 * shadow_radius_x` [S1, unofficial].
    *   Draw a semi-transparent ellipse at `(shadow_x, shadow_y)` with axes `(shadow_radius_x, shadow_radius_y)` onto the shadow layer [S1, unofficial].
    *   After all points are processed, apply a Gaussian blur to the shadow layer with radius `max(2.0, 0.55 * point_radius)` [S1, unofficial].

6.  **Surfel Rendering:**
    Create the final RGBA canvas. Composite the blurred shadow layer onto the canvas. Iterate through points in `order`. For each point at index `i`:
    *   Compute shaded color: `color = np.clip(base_color * (0.84 + 0.16 * float(depth_norm[i])), 0.0, 255.0).astype(np.uint8)` [S1, unofficial].
    *   Compute sprite radius: `point_r = int(round(point_radius * (0.94 + 0.08 * float(depth_norm[i]))))` [S1, unofficial].
    *   Generate a circular sprite (surfels) with the shaded color and radius `point_r` [S1, unofficial].
    *   Alpha composite the sprite at the 2D position `(xy[i, 0], xy[i, 1])` onto the canvas [S1, unofficial].

7.  **Finalization:**
    Resize the canvas to `(resolution, resolution)` using LANCZOS resampling [S1, unofficial]. If `transparent` is False, composite the result onto a white background and save as RGB [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from PIL import Image, ImageDraw, ImageFilter

def render_surfels(points, labels, resolution=1000, point_radius=16.0, azimuth=0.0, elevation=0.0, transparent=False):
    """
    Render a point cloud as surfels with depth shading and soft shadows.
    
    Args:
        points: [N, 3] float32 array of 3D coordinates.
        labels: [N] int64 array of labels (used for base color, assumed uniform here for simplicity).
        resolution: int, output image size.
        point_radius: float, base sprite radius.
        azimuth: float, camera azimuth in degrees.
        elevation: float, camera elevation in degrees.
        transparent: bool, if False, composite on white background.
        
    Returns:
        RGBA image array [resolution, resolution, 4] uint8.
    """
    N = points.shape[0]
    size = resolution
    
    # 1. Normalization and Rotation
    span = points.max(axis=0) - points.min(axis=0)
    span = np.maximum(span, 1e-6) # Prevent division by zero [S1, unofficial]
    image_scale = 0.82 * size / span.max()
    
    # Rotation matrix from azimuth and elevation [S1, unofficial]
    # Note: Exact rotation matrix construction depends on convention. 
    # Assuming standard rotation around Y (azimuth) and X (elevation).
    az_rad = np.radians(azimuth)
    el_rad = np.radians(elevation)
    
    ry = np.array([
        [np.cos(az_rad), 0, np.sin(az_rad)],
        [0, 1, 0],
        [-np.sin(az_rad), 0, np.cos(az_rad)]
    ])
    rx = np.array([
        [1, 0, 0],
        [0, np.cos(el_rad), -np.sin(el_rad)],
        [0, np.sin(el_rad), np.cos(el_rad)]
    ])
    rot_matrix = rx @ ry
    
    camera = (points @ rot_matrix.T)
    
    # 2. Projection and Centering
    xy = camera[:, :2]
    xy_mean = xy.mean(axis=0)
    xy_centered = xy - xy_mean
    xy_scaled = xy_centered * image_scale
    
    # 3. Depth Sorting
    depth = camera[:, 2]
    depth_min = depth.min()
    depth_max = depth.max()
    depth_range = max(depth_max - depth_min, 1e-6) # Prevent division by zero [S1, unofficial]
    depth_norm = (depth - depth_min) / depth_range
    
    order = np.argsort(depth)
    
    # 4. Shadow Layer Initialization
    shadow_img = Image.new('RGBA', (size, size), (0, 0, 0, 0))
    draw_shadow = ImageDraw.Draw(shadow_img)
    
    # Ground y position [S1, unofficial]
    # Note: y.max() refers to the scaled y coordinates
    y_max = xy_scaled[:, 1].max()
    ground_y = min(size * 0.95, float(y_max) + 0.025 * size)
    
    # Base floor shadow ellipse [S1, unofficial]
    # Assuming a base ellipse centered at (0, ground_y) with some radius
    base_shadow_radius = size * 0.4 # Placeholder for base size, not explicitly given but implied by "base floor shadow"
    draw_shadow.ellipse([(-base_shadow_radius, ground_y - base_shadow_radius * 0.3), 
                         (base_shadow_radius, ground_y + base_shadow_radius * 0.3)], 
                        fill=(0, 0, 0, 50)) # Semi-transparent black
    
    # 5. Shadow Compositing
    height = size # Assuming height equals width for square canvas
    for i in order:
        x, y = xy_scaled[i]
        d_norm = depth_norm[i]
        
        # Shadow offset [S1, unofficial]
        shadow_x = float(x) + 0.11 * height
        shadow_y = ground_y + 0.018 * height
        
        # Perspective-scaled shadow radius [S1, unofficial]
        shadow_radius_x = point_radius * (0.78 + 0.08 * float(d_norm))
        shadow_radius_y = 0.30 * shadow_radius_x
        
        # Draw shadow ellipse [S1, unofficial]
        # Coordinates for ellipse: (left, top, right, bottom)
        left = shadow_x - shadow_radius_x
        top = shadow_y - shadow_radius_y
        right = shadow_x + shadow_radius_x
        bottom = shadow_y + shadow_radius_y
        
        draw_shadow.ellipse([left, top, right, bottom], fill=(0, 0, 0, 80)) # Semi-transparent black
        
    # Apply Gaussian blur [S1, unofficial]
    blur_radius = max(2.0, 0.55 * point_radius)
    shadow_img = shadow_img.filter(ImageFilter.GaussianBlur(radius=blur_radius))
    
    # 6. Surfel Rendering
    final_img = Image.new('RGBA', (size, size), (0, 0, 0, 0))
    draw_final = ImageDraw.Draw(final_img)
    
    # Composite shadow layer
    final_img = Image.alpha_composite(final_img, shadow_img)
    
    for i in order:
        x, y = xy_scaled[i]
        d_norm = depth_norm[i]
        
        # Shaded color [S1, unofficial]
        # Assuming base color is white (255, 255, 255) for simplicity, or derived from labels
        base_color = np.array([255, 255, 255], dtype=np.uint8)
        shade_factor = 0.84 + 0.16 * float(d_norm)
        color = np.clip(base_color * shade_factor, 0, 255).astype(np.uint8)
        
        # Sprite radius [S1, unofficial]
        point_r = int(round(point_radius * (0.94 + 0.08 * float(d_norm))))
        
        # Generate sprite (circle) [S1, unofficial]
        left = x - point_r
        top = y - point_r
        right = x + point_r
        bottom = y + point_r
        
        # Alpha composite circle
        # PIL Draw doesn't support alpha composite directly for circles, so we draw on a temp image
        sprite_img = Image.new('RGBA', (size, size), (0, 0, 0, 0))
        draw_sprite = ImageDraw.Draw(sprite_img)
        draw_sprite.ellipse([left, top, right, bottom], fill=tuple(color) + (255,))
        
        final_img = Image.alpha_composite(final_img, sprite_img)
        
    # 7. Finalization
    final_img = final_img.resize((size, size), Image.Resampling.LANCZOS)
    
    if not transparent:
        bg = Image.new('RGB', (size, size), (255, 255, 255))
        final_img_rgb = Image.new('RGB', (size, size), (0, 0, 0))
        final_img_rgb.paste(bg, mask=final_img.split()[3]) # Use alpha channel as mask
        final_img = final_img_rgb
        
    return np.array(final_img)
```

**Parameters:**
*   `resolution`: int, target output image width and height in pixels. Default: 1000 [S1, unofficial].
*   `point_radius`: float, base radius for surfel sprites in normalized coordinates before scaling. Default: 16.0 [S1, unofficial].
*   `azimuth`, `elevation`: float, camera angles in degrees used to construct the rotation matrix [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** Clamping `span` and depth range denominator to `1e-6` prevents errors when points are degenerate or have zero depth spread [S1, unofficial].
*   **Coordinate System:** The rotation matrix construction assumes a specific convention for azimuth and elevation. Deviations may require adjustment of the rotation matrices `ry` and `rx` [S1, unofficial].
*   **Shadow Positioning:** The shadow offset and ground y-position are hardcoded constants. Changes in scene scale or camera angle may require tuning these constants for optimal visual grounding [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** O(N log N) for depth sorting, O(N) for projection, shadow compositing, and surfel rendering [S1, unofficial].
*   **Memory Complexity:** O(N) for intermediate coordinate and depth arrays [S1, unofficial].
*   **Strengths:** Provides visual grounding via soft shadows without altering point semantics; simple CPU-based implementation; depth-aware shading adds perceptual depth cues.
*   **Limitations:** Performance degrades with very large N due to O(N log N) sorting and per-point sprite generation; shadow parameters are heuristic and may not generalize across all scenes; lacks true 3D lighting models.
*   **When to Use:** Quick visualization of point clouds where visual grounding is important; CPU-only environments; small to medium-sized point clouds.
*   **When Not to Use:** Real-time rendering requirements; very large point clouds (>1M points); scenes requiring accurate lighting or shadow casting.

## Background
Originating from the Gaussian-JEPA project, this method is an unofficial visualization pipeline designed to render point clouds with enhanced visual cues [S1, unofficial]. It is part of a broader effort to improve the interpretability of point cloud data in machine learning contexts.

## Sources
[S1] Amazingren/Gaussian-JEPA, https://github.com/Amazingren/Gaussian-JEPA, fetched 2026-10-27 (unofficial)
