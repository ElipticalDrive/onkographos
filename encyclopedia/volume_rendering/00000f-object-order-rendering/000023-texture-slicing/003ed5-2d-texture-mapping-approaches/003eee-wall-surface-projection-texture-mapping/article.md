# Wall Surface Projection & Texture Mapping

Also called: Axis-aligned Texture Projection, Planar Mapping, renderBlockSurfaces

## Overview
Wall Surface Projection & Texture Mapping renders a 3D volume by projecting individual axis-aligned block faces onto 2D screen coordinates, handling clipping, perspective-correct UV mapping, and per-face shading to draw pixels directly. It differs from standard texture slicing by operating on discrete, visible block faces (often in voxel engines) rather than continuous slicing planes, relying on manual pixel iteration for rasterization instead of hardware fragment shaders.

## How it works
**Inputs and Outputs:**
- Input: A 3D voxel grid (chunk) containing block data, a camera position (player), and a 2D texture map (RGB data).
- Output: A 2D screen buffer with rendered pixels.

**Data Structures:**
- `Block`: Contains `visibleFaces` (boolean flags for 6 faces: XPos, XNeg, YPos, YNeg, ZPos, ZNeg) and active state.
- `Texture`: Contains `textureMap` (flat array of RGB integers), `width`, and `height`.
- `Renderer`: Manages screen dimensions (`SCR_WIDTH`, `SCR_HEIGHT`) and pixel drawing.

**Steps:**
1. **Visibility Culling:** For each block in the chunk, determine which faces are visible by checking if adjacent blocks are inactive or out of bounds [S1, unofficial].
2. **Projection & Clipping:** Project the 3D coordinates of a visible face's corners to screen space. Clip edges that are behind the player or outside the screen bounds [S1, unofficial].
3. **UV Calculation:** Compute texture coordinates (U, V) for the projected face vertices.
4. **Rasterization:** Iterate over the screen pixels covered by the projected face. For each pixel, calculate the corresponding texture index using modulo wrapping for tiling.
5. **Shading & Drawing:** Apply per-face shading to the sampled color and draw the pixel, potentially using distance to the player for depth sorting or fog [S1, unofficial].

**Formulas:**
- **Horizontal Step:**
  $$ \text{horizontalStep} = \frac{\text{texture.width} \times uvU}{xPos2 - xPos1} $$
  Where $xPos1, xPos2$ are the screen X-coordinates of the face's left and right edges.
- **Vertical Position:**
  $$ y1 = bottomPos1 + distBottom \times \frac{x - originalX1 + 0.5}{distX} $$
  $$ y2 = topPos1 + distTop \times \frac{x - originalX1 + 0.5}{distX} $$
  Where $distX = \max(1, xPos2 - xPos1)$ prevents division by zero [S1, unofficial].
- **Vertical Step:**
  $$ \text{verticalStep} = \frac{\text{texture.height} \times uvV}{y2 - y1} $$
- **Texture Index:**
  $$ p = (\text{texture.height} - (\text{verticalStart} \pmod{\text{texture.height}}) - 1) \times 3 \times \text{texture.width} + (\text{horizontalStart} \pmod{\text{texture.width}}) \times 3 $$
  This formula maps screen coordinates to the flat RGB texture array [S1, unofficial].
- **Shading:**
  $$ \text{color} = \max(\text{textureMap}[p + c] - \frac{\text{shade}}{2}, 0) $$
  Where $c$ is the color channel index (0 for R, 1 for G, 2 for B) [S1, unofficial].

**Pitfalls:**
- **Division by Zero:** If $xPos2 - xPos1 = 0$, the horizontal step is undefined. The source handles this by forcing $distX = 1$ [S1, unofficial].
- **Clipping:** Edges behind the player ($y < 1$) must be clipped to avoid rendering artifacts [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_wall_surface(renderer, block, texture, player):
    # 1. Get visible face corners in 3D
    corners = get_face_corners(block)
    
    # 2. Project to screen space
    screen_corners = []
    for corner in corners:
        screen_corners.append(project_to_screen(corner, player))
        
    # 3. Clip behind player and screen bounds
    clipped_corners = clip_screen_edges(screen_corners, player, renderer.SCR_WIDTH, renderer.SCR_HEIGHT)
    
    if not clipped_corners:
        return

    # 4. Calculate UVs and steps
    # Assuming a quad face with corners [0,1,2,3]
    x1, y1 = clipped_corners[0].x, clipped_corners[0].y
    x2, y2 = clipped_corners[1].x, clipped_corners[1].y
    
    dist_x = max(1, x2 - x1)
    horizontal_step = (texture.width * uv_u) / dist_x
    
    # Calculate y bounds for each x
    for x in range(int(x1), int(x2) + 1):
        # Interpolate y1 and y2 for current x
        y_top = y1 + (y2 - y1) * (x - x1) / dist_x
        y_bottom = y1 + (y2 - y1) * (x - x1) / dist_x # Simplified for quad
        
        # Clamp y to screen
        y_start = max(0, int(y_top))
        y_end = min(renderer.SCR_HEIGHT, int(y_bottom))
        
        if y_start >= y_end:
            continue
            
        dist_y = max(1, y_end - y_start)
        vertical_step = (texture.height * uv_v) / dist_y
        
        horizontal_start = x * horizontal_step
        vertical_start = 0
        
        for y in range(y_start, y_end):
            # Calculate texture index
            p = ((texture.height - (int(vertical_start) % texture.height) - 1) * 3 * texture.width + 
                 (int(horizontal_start) % texture.width) * 3)
            
            # Apply shading
            r = max(texture.textureMap[p + 0] - renderer.shade / 2, 0)
            g = max(texture.textureMap[p + 1] - renderer.shade / 2, 0)
            b = max(texture.textureMap[p + 2] - renderer.shade / 2, 0)
            
            # Draw pixel
            renderer.draw_pixel(x, y, RgbColor(r, g, b), distance_to_player)
            
            vertical_start += vertical_step
            horizontal_start += horizontal_step
```

## Subtopics

### 2D Texture-Based Volume Rendering
A broader category that includes this method. It uses 2D textures to store volume data and maps them to proxy geometry. This method is a specific implementation within that category, often used in voxel engines for performance. [2D Texture-Based Volume Rendering](916-2d-texture-based-volume-rendering.md)

### Palette Texture Volume Rendering
Uses a palette of colors indexed by the volume data. This method typically uses direct RGB textures. Palette mapping is less common in modern real-time voxel rendering due to color depth limitations. [Palette Texture Volume Rendering](6502-palette-texture-volume-rendering.md)

### Cabral Texture Mapping
Uses normal vectors to compute lighting on the texture-mapped surfaces. This method may or may not use normal mapping, but Cabral's approach specifically integrates normals into the projection. [Cabral Texture Mapping](7257-cabral-texture-mapping.md)

### Multi-constraints-based projective texture mapping
Uses multiple constraints to optimize the projection of textures onto complex surfaces. This method is simpler and relies on axis-aligned faces, making it less flexible for curved surfaces but faster for voxel grids. [Multi-constraints-based projective texture mapping](12075-multi-constraints-based-projective-texture-mapping.md)

### Projective Space Texture Mapping with Multi-Resolution Templates
Uses multi-resolution templates to handle texture mapping in projective space. This method is more complex and designed for high-quality rendering, whereas wall surface projection is optimized for simple, discrete blocks. [Projective Space Texture Mapping with Multi-Resolution Templates](13236-projective-space-texture-mapping-with-multi-resolution-templ.md)

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of visible pixels on the screen. The complexity is linear with respect to the screen area covered by visible faces.
- **Memory Complexity:** $O(V)$ where $V$ is the number of voxels in the chunk, for storing block data. Texture memory is $O(T)$ where $T$ is the texture size.
- **Performance:** Very fast for simple voxel grids due to minimal overhead and no complex geometry processing. However, it lacks hardware acceleration for blending and shading, relying on CPU-side pixel iteration.
- **Strengths:** Simple implementation, low memory footprint, suitable for real-time voxel engines.
- **Limitations:** No hardware blending, limited to axis-aligned faces, no support for curved surfaces or complex lighting without additional CPU work.
- **When to Use:** Real-time voxel rendering, simple 3D environments, educational purposes.
- **When Not to Use:** High-fidelity rendering, complex geometry, curved surfaces, or when hardware acceleration is required.

## Background
Originated in early 3D engines and voxel games, this method is a practical approach to rendering 3D volumes using 2D textures. It is commonly found in indie voxel games and educational 3D engines. The method is documented in the source [S1, unofficial], which provides a complete implementation in C++.

## Sources
[S1] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
