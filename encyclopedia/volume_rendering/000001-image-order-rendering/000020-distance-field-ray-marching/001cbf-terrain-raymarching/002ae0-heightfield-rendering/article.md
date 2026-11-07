# Heightfield Rendering

Also called: 2D Raycast Heightmap Renderer, Comanche Heightmap Algorithm, View-Z slices

## Overview
Heightfield Rendering is a software rendering technique that projects a 2D heightmap onto a 2D screen by casting rays from the camera through the scene depth at discrete view-Z planes, sampling the heightmap to determine visible surface pixels, and filling vertical screen segments from the horizon down to the projected terrain height [S1, S2, S3, S4, unofficial]. It solves the hidden surface problem for terrain by maintaining a per-column occlusion buffer (`yBuffer` or `hiddenY`), ensuring only the closest visible terrain segment is drawn in each screen column [S1, S2, S4, unofficial]. Unlike standard ray marching which casts one ray per pixel or vertical buffer rendering which iterates depth planes differently, this method uses increasing depth intervals (Level of Detail) and variable column widths to optimize performance for distant terrain while maintaining perspective [S2, S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A heightmap texture (grayscale or red channel), a color map texture (RGB), camera parameters (position, rotation, FOV, distance), and screen resolution.
*   **Outputs:** A 2D screen buffer (framebuffer) with the rendered terrain.

**Data Structures:**
*   **Camera (`vsCamera`):** Contains `position` (x, y, z), `rotation` (x, y), `distance` (max view distance), and `fov` (field of view in degrees) [S2, unofficial].
*   **Map (`vsMap`):** Contains `width`, `height`, `shift` (log2 of width for bitwise operations), `altitude` (pointer to heightmap data), and `color` (pointer to colormap data) [S2, unofficial].
*   **Screen Data (`vsScreenData`):** Contains `width`, `height`, `backgroundColor`, and `screen` (1D array of colors indexed by `x + y * width`) [S2, unofficial].
*   **Occlusion Buffer (`yBuffer`):** An integer array of size `WIDTH`, initialized to `HEIGHT` (bottom of screen), tracking the lowest drawn pixel for each column [S2, S4, unofficial].

**Parameters:**
*   `MAX_DEPTH`: Maximum rendering distance (default 1024) [S4, unofficial].
*   `LOD_DISTANCE`: Distance interval for LOD step increase (default 256) [S4, unofficial].
*   `LOD_AMOUNT`: Multiplier for distance step when LOD triggers (default 1.5) [S4, unofficial].
*   `FOG_START`: Distance at which fog begins (default 256) [S4, unofficial].
*   `CLOSE_LOD`: Multiplier for column width near camera (default 1.0) [S4, unofficial].
*   `FOV`: Field of view in degrees (default 90.0) [S4, unofficial].
*   `WIDTH`/`HEIGHT`: Screen resolution (default 800x800) [S4, unofficial].
*   `zScale`: Vertical scaling factor, calculated as `(WIDTH/HEIGHT) * 1.0` [S4, unofficial].
*   `viewFactor`: Horizontal projection factor, calculated as `sin(FOV/2 in radians)` [S4, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Initialize `yBuffer` array of size `WIDTH` with all values set to `HEIGHT` [S2, S4, unofficial].
    *   Calculate `viewFactor = sinf(FOV / 2.0f)` [S4, unofficial].
    *   Calculate `zScale = (float)WIDTH / HEIGHT * 1.0` [S4, unofficial].
    *   Initialize camera position and rotation [S2, S4, unofficial].

2.  **Depth Loop:**
    *   Iterate depth `d` from 1.0 to `MAX_DEPTH`.
    *   Calculate column width: `colWidth = WIDTH / (viewFactor * d * 2 * CLOSE_LOD)`. If `colWidth < 1`, set `colWidth = 1` [S4, unofficial].
    *   After processing columns for depth `d`, update `d` using LOD: `d += ((int)d / LOD_DISTANCE) * LOD_AMOUNT` [S4, unofficial].

3.  **Column Loop:**
    *   Iterate column start positions `sx` from 0 to `WIDTH` with step `colWidth` [S4, unofficial].

4.  **Ray Sampling:**
    *   Calculate sample position in camera space:
        *   `samplePos.x = Lerp(-viewFactor * d, viewFactor * d, sx / WIDTH)`
        *   `samplePos.y = d` [S4, unofficial].
    *   Apply camera rotation:
        *   `samplePos.x = cosRot * samplePos.x - sinRot * samplePos.y`
        *   `samplePos.y = sinRot * samplePos.x + cosRot * samplePos.y` [S4, unofficial].
    *   Translate by player position:
        *   `samplePos.x += playerPos.x`
        *   `samplePos.y += playerPos.y` [S4, unofficial].

5.  **Heightmap Sampling:**
    *   Sample heightmap at `samplePos` using modulo wrapping for bounds:
        *   `heightVal = heightMap[modulo((int)samplePos.x, heightMap.width), modulo((int)samplePos.y, heightMap.height)]` [S4, unofficial].
    *   Calculate world height `z`:
        *   `z = -heightVal * zScale + playerPos.z` [S4, unofficial]. (Note: S4 negates height; S3 uses `camera.height - heightmap`).

6.  **Screen Projection:**
    *   Calculate screen Y coordinate `sy`:
        *   `sy = (z / d) * HEIGHT + HEIGHT / 2` [S4, unofficial].
    *   Clamp `sy` to `[0, HEIGHT - 1]` [S3, unofficial].

7.  **Occlusion Check and Drawing:**
    *   If `sy < yBuffer[sx]` (terrain is closer/lower on screen than previously drawn):
        *   Sample color from colormap at `samplePos` [S4, unofficial].
        *   Apply fog if `d > MAX_DEPTH - FOG_START`: `color.a = (int)(255 * (MAX_DEPTH - d) / FOG_START)` [S4, unofficial].
        *   Draw vertical line from `sy` to `yBuffer[sx]` (or `yBuffer[sx+i]` for each pixel in column width) [S2, S4, unofficial].
        *   Update `yBuffer[sx+i] = sy` for all pixels in the column width [S2, S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def render_heightfield(camera, height_map, color_map, screen_width, screen_height):
    # Constants
    MAX_DEPTH = 1024
    LOD_DISTANCE = 256
    LOD_AMOUNT = 1.5
    FOG_START = 256
    CLOSE_LOD = 1.0
    FOV = 90.0
    
    # Pre-calculated factors
    view_factor = math.sin(math.radians(FOV / 2.0))
    z_scale = (screen_width / screen_height) * 1.0
    
    # Initialize occlusion buffer
    y_buffer = [screen_height] * screen_width
    
    # Camera rotation
    cos_rot = math.cos(-camera.rotation.y)
    sin_rot = math.sin(-camera.rotation.y)
    
    # Depth loop
    d = 1.0
    while d < MAX_DEPTH:
        # Calculate column width
        col_width = int(screen_width / (view_factor * d * 2 * CLOSE_LOD))
        if col_width < 1:
            col_width = 1
            
        # Column loop
        sx = 0
        while sx < screen_width:
            # Sample position in camera space
            sample_x_cam = (-view_factor * d) + ((view_factor * d) - (-view_factor * d)) * (sx / screen_width)
            sample_y_cam = d
            
            # Apply rotation
            sample_x_rot = cos_rot * sample_x_cam - sin_rot * sample_y_cam
            sample_y_rot = sin_rot * sample_x_cam + cos_rot * sample_y_cam
            
            # Translate by player position
            sample_x_world = sample_x_rot + camera.position.x
            sample_y_world = sample_y_rot + camera.position.y
            
            # Sample heightmap (modulo wrapping)
            h_x = sample_x_world % height_map.width
            h_y = sample_y_world % height_map.height
            if h_x < 0: h_x += height_map.width
            if h_y < 0: h_y += height_map.height
            
            height_val = height_map.get_color(h_x, h_y).r # Assuming red channel is height
            world_z = -height_val * z_scale + camera.position.z
            
            # Project to screen
            sy = (world_z / d) * screen_height + (screen_height / 2)
            sy = max(0, min(screen_height - 1, int(sy)))
            
            # Occlusion check
            if sy < y_buffer[sx]:
                # Sample color
                c_x = sample_x_world % color_map.width
                c_y = sample_y_world % color_map.height
                if c_x < 0: c_x += color_map.width
                if c_y < 0: c_y += color_map.height
                
                color = color_map.get_color(c_x, c_y)
                
                # Fog
                if d > MAX_DEPTH - FOG_START:
                    alpha = int(255 * (MAX_DEPTH - d) / FOG_START)
                    color.a = alpha
                
                # Draw vertical line and update buffer
                for i in range(col_width):
                    col_idx = sx + i
                    if col_idx < screen_width:
                        # Draw from sy to y_buffer[col_idx]
                        # Implementation detail: fill pixels in screen buffer
                        for y in range(sy, y_buffer[col_idx]):
                            screen_buffer.set_pixel(col_idx, y, color)
                        y_buffer[col_idx] = sy
            
            sx += col_width
            
        # LOD update
        d += ((int(d) / LOD_DISTANCE) * LOD_AMOUNT)
```

**Edge Cases and Pitfalls:**
*   **Overshooting:** The naive distance field function $D(pos) = h(pos.xz) + pos.y$ can overestimate distance, causing rays to overshoot the heightfield [S5, unofficial]. A shrink factor $\frac{1}{\sqrt{c^2+1}}$ is required, where $c$ is the bound on the absolute value of the heightfield's derivative [S5, unofficial].
*   **Bounds Checking:** Heightmap coordinates wrap around using modulo or bitwise AND with power-of-two dimensions (e.g., 1023 for 1024 width) [S2, S3, S4, unofficial].
*   **Pitch Limits:** Pitch is limited to ±80° and roll is disabled to maintain stable view-Z slicing [S1, unofficial].
*   **Image Dimensions:** Maps should be square, same dimensions, and powers of two for efficient bitwise operations [S2, unofficial].

## Subtopics

### [Vertical Buffer Terrain Rendering](3924-vertical-buffer-terrain-rendering.md)
A software rasterization technique that projects a 2D heightmap onto a screen by iterating through depth planes and drawing horizontal terrain slices from near to far. It differs by using screen-space coordinate stepping and bitwise heightmap indexing, avoiding per-pixel ray casting.

### [GBA Voxel Slice Heightmap Renderer](4080-gba-voxel-slice-heightmap-renderer.md)
A software rendering technique for 3D voxel terrain on constrained hardware that iterates over discrete depth/height slices, maps screen columns to world coordinates using precomputed lookup tables and fixed-point math. It differs by explicitly managing discrete depth slices with precomputed coordinate offsets to optimize for the Game Boy Advance's ARM7TDMI CPU.

## Operating characteristics
*   **Time Complexity:** $O(WIDTH \times (MAX\_DEPTH / LOD\_step))$ in the worst case, but optimized by variable column width which reduces the number of rays cast at greater distances [S4, unofficial].
*   **Memory Complexity:** $O(WIDTH)$ for the occlusion buffer, plus storage for height and color maps [S2, S4, unofficial].
*   **Performance:** Highly efficient for fixed-resolution 2D visualizations on CPU due to avoidance of per-pixel ray casting and use of LOD [S1, S2, S4, unofficial].
*   **Strengths:** Real pitch without horizon shift, efficient hidden surface removal via per-column buffer, supports fog and LOD [S1, S2, S4, unofficial].
*   **Limitations:** Limited pitch (±80°), no roll, requires power-of-two map dimensions for optimal performance, potential for overshooting without shrink factor [S1, S2, S5, unofficial].
*   **When to Use:** Real-time terrain rendering on CPU-constrained systems, retro-style graphics, scenarios where perspective accuracy is needed without full 3D ray marching overhead.
*   **When Not to Use:** When full 3D rotation (roll) is required, when high-precision distance fields are needed without approximation, or when GPU acceleration is available and preferred.

## Background
Heightfield Rendering, also known as the Comanche Heightmap Algorithm, originated from the Voxel Space engine used in the classic Comanche video games [S1, S3, S4, unofficial]. It is a CPU-driven rendering technique that projects rays from the camera through the scene depth, sampling the heightmap at each step to determine visible surface pixels [S4, unofficial]. The method was popularized in open-source implementations such as VoxelSpace and VoxelSpaceEngine, which provide minimalistic retro rendering libraries [S1, S2, unofficial]. It is used in software rendering contexts where hardware acceleration is limited or unavailable, such as on older consoles or embedded systems [S1, S2, S3, S4, unofficial].

## Sources
[S1] varganorbert222/VoxelSpace, https://github.com/varganorbert222/VoxelSpace, fetched 2026-10-24 (unofficial)
[S2] Not-Nik/VoxelSpaceEngine, https://github.com/Not-Nik/VoxelSpaceEngine, fetched 2026-10-27 (unofficial)
[S3] AndreasVlcek/Voxel-Space-Engine-Comanche-Terrain-Rendering, https://github.com/AndreasVlcek/Voxel-Space-Engine-Comanche-Terrain-Rendering, fetched 2026-10-27 (unofficial)
[S4] PaigePalisade/HeightmapViewer, https://github.com/PaigePalisade/HeightmapViewer, fetched 2026-10-28 (unofficial)
[S5] A Note on Ray Marching with Heightfields, https://www.peterstefek.me/ray-marching-heightfields.html, fetched 2026-10-30 (unofficial)
