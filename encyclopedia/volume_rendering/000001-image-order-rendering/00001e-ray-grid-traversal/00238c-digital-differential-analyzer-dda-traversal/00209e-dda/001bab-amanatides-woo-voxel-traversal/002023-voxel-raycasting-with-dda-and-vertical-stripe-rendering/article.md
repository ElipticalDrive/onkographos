# Voxel Raycasting with DDA and Vertical Stripe Rendering

Also called: 3D Raycasting, Vox5

## Overview
Voxel Raycasting with DDA and Vertical Stripe Rendering is a real-time rendering technique for 3D voxel grids that utilizes a 2D Digital Differential Analyzer (DDA) to cast rays per screen column, determining the first occupied voxel [S1, unofficial]. It renders the voxel as a vertical stripe (line) rather than a full textured quad, handling depth via a per-column depth buffer to support multiple voxels per column (stacking) [S1, unofficial]. Unlike standard raycasting which draws a full vertical line for a wall, this method draws vertical stripes for each depth layer within the voxel, allowing for internal structure visibility and stacking via the depth buffer [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid `worldMap` of dimensions `mapWidth x mapHeight x mapDepth`, a camera position `(posX, posY, posZ)`, a camera direction vector `(dirX, dirY)`, a camera plane vector `(planeX, planeY)` defining the field of view, and a vertical pitch offset `pitch`.
*   **Output:** A 2D image where each screen column is filled with vertical stripes representing the visible voxel layers, updated against a depth buffer.

**Data Structures:**
*   `worldMap`: A 3D array `ColorRGB worldMap[mapWidth][mapHeight][mapDepth]`.
    *   Dimensions: `mapWidth = 96`, `mapHeight = 96`, `mapDepth = 12` [S1, unofficial].
    *   `ColorRGB`: A struct with 3 components (Red, Green, Blue), loaded sequentially from source data [S1, unofficial].
*   `depth`: A 2D array `int depth[2][384]`.
    *   Size: 2 buffers of size 384 (screen height) [S1, unofficial].
    *   Initialization: Values set to 100000 [S1, unofficial].
    *   Purpose: Stores the Z-depth of the last drawn voxel at each screen row to handle occlusion within a column [S1, unofficial].
*   Camera State:
    *   `posX, posY, posZ`: Camera position [S1, unofficial].
    *   `dirX, dirY`: Ray direction vector [S1, unofficial].
    *   `planeX, planeY`: Camera plane vector for FOV calculation [S1, unofficial].
    *   `pitch`: Vertical tilt offset, typically 200 [S1, unofficial].

**Formulas:**
*   **Delta Distances:**
    $$deltaDistX = \frac{|1.0|}{|dirX|}$$
    $$deltaDistY = \frac{|1.0|}{|dirY|}$$
    [S1, unofficial]
*   **Perpendicular Wall Distance (Fisheye Correction):**
    If `side == 0` (hit X-wall):
    $$perpWallDist = \frac{mapX - rayPosX + (1 - stepX) / 2}{dirX}$$
    If `side == 1` (hit Y-wall):
    $$perpWallDist = \frac{mapY - rayPosY + (1 - stepY) / 2}{dirY}$$
    This projects the oblique distance onto the camera axis to prevent the fisheye effect [S1, unofficial].
*   **Line Height on Screen:**
    $$lineHeight = \left\lfloor \frac{h}{perpWallDist} \right\rfloor$$
    where $h$ is the screen height. Note: The source notes that dividing by 2 for square voxels causes a crash, so it is omitted, resulting in vertically stretched voxels [S1, unofficial].
*   **Draw Start/End for Layer `b`:**
    $$drawStart = (lineHeight \times (b - posZ)) + pitch$$
    $$drawEnd = lineHeight + (lineHeight \times (b - posZ)) + pitch$$
    Values are clamped to `[0, h-1]` [S1, unofficial].

**Algorithm Steps:**
1.  **Initialize Ray Parameters:**
    Calculate `deltaDistX` and `deltaDistY` based on `dirX` and `dirY`.
    Determine initial `stepX` and `stepY` based on ray direction:
    If `dirX < 0`: `stepX = -1`, `sideDistX = (posX - mapX) * deltaDistX`
    Else: `stepX = 1`, `sideDistX = (mapX + 1.0 - posX) * deltaDistX`
    Same logic for Y using `posY`, `mapY`, `dirY`, `stepY`, `sideDistY` [S1, unofficial].

2.  **DDA Loop (Find First Hit):**
    Initialize `mapX`, `mapY` to the starting grid cell. Initialize `hit = 0`, `count = 0`.
    While `mapX < 0 || mapY < 0 || (count < 384 && hit < 900)`:
    *   If `sideDistX < sideDistY`:
        *   `sideDistX += deltaDistX`
        *   `mapX += stepX`
        *   `side = 0`
    *   Else:
        *   `sideDistY += deltaDistY`
        *   `mapY += stepY`
        *   `side = 1`
    *   Clamp `mapX` to `[0, mapHeight-1]` and `mapY` to `[0, mapWidth-1]` [S1, unofficial].
    *   Increment `hit`.
    *   If `worldMap[mapX][mapY][0]` is not empty (or hit condition met), break loop. [S1, unofficial] (Note: Source implies loop continues until hit or bounds, then processes).

3.  **Calculate Geometry for Current Hit:**
    *   Calculate `perpWallDist` using the formula above.
    *   Calculate `lineHeight` using the formula above.
    *   **Prepare Horizontal Fill State:**
        Copy current `mapX, mapY, side, sideDistX, sideDistY` to temporary variables `tmapX, tmapY, tside, tsideDistX, tsideDistY`.
        Advance one step again using the same DDA logic to determine the extent of the voxel face.
        Clamp `tmapX, tmapY`.
        Calculate `tperpWallDist` and `tlineHeight` using the same formulas [S1, unofficial].

4.  **Render Vertical Stripes (Depth Layers):**
    Iterate through all depth layers `b` from `0` to `mapDepth - 1`:
    *   Calculate `drawStart` and `drawEnd` for layer `b`.
    *   Get color `color = worldMap[mapX][mapY][b]`.
    *   Apply shading: If `side == 1`, `color = color / 2` [S1, unofficial].
    *   If `color` is not black:
        *   Call `verLineTriDepth(x, drawStart, drawEnd, color, depth[0], 384, &count, 0, depth[1])`.
        *   This function draws the vertical line segment, updating `depth[0]` and `depth[1]` to handle occlusion within the column [S1, unofficial].

5.  **Render Second Pass (Horizontal Fill/Edge):**
    Iterate through all depth layers `b` from `0` to `mapDepth - 1`:
    *   Calculate `tdrawStart` and `tdrawEnd` using `tlineHeight`.
    *   Get `tcolor = worldMap[mapX][mapY][b]`.
    *   Apply shading: `tcolor = tcolor / 3` [S1, unofficial].
    *   Call `verLineTriDepth` with `drawStart` or `tdrawStart` depending on whether `b < posZ`, and `depth[1]` as the primary buffer [S1, unofficial].

6.  **Swap Depth Buffers:**
    At the end of the DDA loop iteration (per voxel hit), copy `depth[1]` to `depth[0]`:
    `memcpy(depth[0], depth[1], 384 * sizeof(int))`
    This prepares `depth[1]` for the next ray or pass [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_voxel_column(screen_x, ray_pos_x, ray_pos_y, ray_dir_x, ray_dir_y, map_width, map_height, map_depth, world_map, screen_height, pitch):
    # 1. Initialize DDA
    delta_dist_x = abs(1.0 / ray_dir_x)
    delta_dist_y = abs(1.0 / ray_dir_y)
    
    map_x = int(ray_pos_x)
    map_y = int(ray_pos_y)
    
    if ray_dir_x < 0:
        step_x = -1
        side_dist_x = (ray_pos_x - map_x) * delta_dist_x
    else:
        step_x = 1
        side_dist_x = (map_x + 1.0 - ray_pos_x) * delta_dist_x
        
    if ray_dir_y < 0:
        step_y = -1
        side_dist_y = (ray_pos_y - map_y) * delta_dist_y
    else:
        step_y = 1
        side_dist_y = (map_y + 1.0 - ray_pos_y) * delta_dist_y
        
    hit = 0
    count = 0
    side = 0
    
    # 2. DDA Loop
    while map_x < 0 or map_y < 0 or (count < 384 and hit < 900):
        if side_dist_x < side_dist_y:
            side_dist_x += delta_dist_x
            map_x += step_x
            side = 0
        else:
            side_dist_y += delta_dist_y
            map_y += step_y
            side = 1
            
        # Clamp map coordinates
        map_x = max(0, min(map_x, map_width - 1))
        map_y = max(0, min(map_y, map_height - 1))
        
        hit += 1
        
        # Check for hit (simplified: assume first non-black voxel is a hit for this structure)
        # In practice, check world_map[map_x][map_y][0] for occupancy
        if world_map[map_x][map_y][0] != RGB_BLACK:
            break
        count += 1
        
    if hit >= 900:
        return # No hit within bounds

    # 3. Calculate Geometry
    if side == 0:
        perp_wall_dist = (map_x - ray_pos_x + (1 - step_x) / 2) / ray_dir_x
    else:
        perp_wall_dist = (map_y - ray_pos_y + (1 - step_y) / 2) / ray_dir_y
        
    line_height = int(screen_height / perp_wall_dist)
    
    # Prepare horizontal fill state
    t_map_x, t_map_y, t_side = map_x, map_y, side
    t_side_dist_x, t_side_dist_y = side_dist_x, side_dist_y
    
    # Advance one step for t variables
    if t_side_dist_x < t_side_dist_y:
        t_side_dist_x += delta_dist_x
        t_map_x += step_x
        t_side = 0
    else:
        t_side_dist_y += delta_dist_y
        t_map_y += step_y
        t_side = 1
        
    t_map_x = max(0, min(t_map_x, map_width - 1))
    t_map_y = max(0, min(t_map_y, map_height - 1))
    
    if t_side == 0:
        t_perp_wall_dist = (t_map_x - ray_pos_x + (1 - step_x) / 2) / ray_dir_x
    else:
        t_perp_wall_dist = (t_map_y - ray_pos_y + (1 - step_y) / 2) / ray_dir_y
        
    t_line_height = int(screen_height / t_perp_wall_dist)
    
    # 4. Render Vertical Stripes
    for b in range(map_depth):
        draw_start = (line_height * (b - ray_pos_z)) + pitch
        draw_end = line_height + (line_height * (b - ray_pos_z)) + pitch
        
        draw_start = max(0, min(draw_start, screen_height - 1))
        draw_end = max(0, min(draw_end, screen_height - 1))
        
        color = world_map[map_x][map_y][b]
        
        if side == 1:
            color = color / 2
            
        if color != RGB_BLACK:
            ver_line_tri_depth(screen_x, draw_start, draw_end, color, depth[0], 384, count, 0, depth[1])
            
    # 5. Render Second Pass (Horizontal Fill/Edge)
    for b in range(map_depth):
        t_draw_start = (t_line_height * (b - ray_pos_z)) + pitch
        t_draw_end = t_line_height + (t_line_height * (b - ray_pos_z)) + pitch
        
        t_draw_start = max(0, min(t_draw_start, screen_height - 1))
        t_draw_end = max(0, min(t_draw_end, screen_height - 1))
        
        t_color = world_map[map_x][map_y][b]
        t_color = t_color / 3
        
        # Determine start/end based on b < ray_pos_z
        if b < ray_pos_z:
            start = draw_start
        else:
            start = t_draw_start
            
        ver_line_tri_depth(screen_x, start, t_draw_end, t_color, depth[1], 384, count, 0, depth[1])
        
    # 6. Swap Depth Buffers
    depth[0] = depth[1].copy() # Simulating memcpy
```

## Subtopics

### [Dual-Layer DDA Raycasting](2212-dual-layer-dda-raycasting.md)
Extends this method to support two distinct vertical layers (ceilings and floors) in a single pass using a 3D DDA, differing from the single-layer focus of the parent.

### [Classic (Column Voxel Space)](5024-classic-column-voxel-space.md)
A simpler technique that casts one ray per screen column using a height-map-based march, differing from this method's per-pixel/vertical-stripe approach.

### [Screen-Space Ray Casting with Layer Traversal](5221-screen-space-ray-casting-with-layer-traversal.md)
Performs ray casting in screen space with explicit layer index handling for multiple refractions, differing from this method's strict 3D grid traversal.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of screen columns and the maximum traversal depth (capped at 900 steps in the source). The DDA traversal is efficient, jumping directly between grid boundaries.
*   **Memory Complexity:** Dominated by the 3D voxel grid `worldMap` and the two depth buffers. For the source dimensions, this is minimal.
*   **Performance:** Designed for real-time rendering on limited hardware (implied by the C-style implementation and lack of complex shading).
*   **Strengths:** Handles internal voxel structure visibility via depth buffering; avoids fisheye distortion; efficient traversal.
*   **Limitations:** Vertically stretched voxels due to aspect ratio handling; limited to 2 depth buffers for occlusion; hardcoded bounds and parameters in the source implementation.
*   **When to use:** Real-time voxel visualization where internal structure is important and performance is critical.
*   **When not to use:** When square voxel aspect ratio is required without workarounds; when deep volume rendering with many layers is needed beyond the fixed depth buffer capacity.

## Background
This method originates from the `voxnine` project by JCLemme, an unofficial implementation of 3D voxel raycasting using DDA [S1, unofficial]. It builds on the Amanatides–Woo Voxel Traversal algorithm for efficient grid traversal. The technique is notable for its use of vertical stripes and dual depth buffers to handle voxel stacking, distinguishing it from standard Wolfenstein 3D-style raycasting.

## Sources
[S1] JCLemme/voxnine, https://github.com/JCLemme/voxnine, fetched 2026-10-28 (unofficial)
