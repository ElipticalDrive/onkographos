# Voxel Raycasting with Vertical Slicing

Also called: 12-layer vertical raycasting, voxel4.cpp raycaster

## Overview
Voxel Raycasting with Vertical Slicing is a 3D rendering technique that visualizes voxel thickness by treating each voxel column as 12 distinct vertical layers. Unlike standard 2D raycasting which draws a single vertical line per hit, this method draws 12 vertical stripes per hit, spaced by `lineHeight` intervals and centered around the intersection point, allowing for the visualization of internal structure and depth [S1, unofficial]. It builds on [Dual-Layer DDA Raycasting](2212-dual-layer-dda-raycasting.md) by extending the vertical dimension from 2 layers to 12, enabling more complex volumetric representation while maintaining the efficiency of DDA traversal.

## How it works
### Inputs and Outputs
- **Inputs**: 
  - `worldMap`: A 3D integer array `int worldMap[mapWidth][mapHeight][12]` representing the voxel grid. Dimensions are `mapWidth = 96` and `mapHeight = 96`. The third dimension (size 12) represents vertical layers. Values 0-5 have specific color mappings; 6-11 default to Yellow [S1, unofficial].
  - `cameraPos`: `(posX, posY)` representing the camera position.
  - `cameraDir`: `(dirX, dirY)` representing the camera direction vector.
  - `cameraPlane`: `(planeX, planeY)` where `planeX = 0` and `planeY = 0.66`, determining the Field of View (FOV) [S1, unofficial].
  - `screenWidth`: Width of the output screen in pixels.
  - `screenHeight`: Height of the output screen in pixels.
  - `pitch`: A double constant representing the vertical tilt offset for the voxel stack. Default value is 200 [S1, unofficial].
- **Outputs**: 
  - A 2D screen buffer where each column `x` contains 12 vertical stripes corresponding to the 12 layers of the hit voxel.

### Data Structures
- **World Map**: `int worldMap[96][96][12]`. Each element stores an integer representing the layer's color or state.
- **Depth Buffer**: `int depth[384]`. Initialized to 100000. Used to handle occlusion between the 12 vertical slices of the same voxel or across rays. The array is reset per ray in the loop [S1, unofficial].

### Steps
1. **Initialize Ray Position and Direction**:
   - Set `rayPosX = posX` and `rayPosY = posY`.
   - For each screen column `x` from 0 to `screenWidth - 1`:
     - Calculate `cameraX = 2 * x / screenWidth - 1`.
     - Calculate `rayDirX = dirX + planeX * cameraX`.
     - Calculate `rayDirY = dirY + planeY * cameraX`.

2. **Initialize DDA State**:
   - Determine `stepX` and `stepY` (+1 or -1) based on the signs of `rayDirX` and `rayDirY`.
   - Calculate initial `sideDistX` and `sideDistY` to the first grid boundary:
     - If `rayDirX < 0`, `stepX = -1`, `sideDistX = (rayPosX - mapX) * deltaDistX`.
     - Else, `stepX = 1`, `sideDistX = (mapX + 1.0 - rayPosX) * deltaDistX`.
     - Same logic applies for Y: If `rayDirY < 0`, `stepY = -1`, `sideDistY = (rayPosY - mapY) * deltaDistY`; else `stepY = 1`, `sideDistY = (mapY + 1.0 - rayPosY) * deltaDistY`.
   - Calculate `deltaDistX = sqrt(1 + (rayDirY * rayDirY) / (rayDirX * rayDirX))` and `deltaDistY = sqrt(1 + (rayDirX * rayDirX) / (rayDirY * rayDirY))` [S1, unofficial].

3. **Perform DDA Loop**:
   - Initialize `mapX = int(rayPosX)` and `mapY = int(rayPosY)`.
   - Initialize `hit = 0`.
   - While `mapX` or `mapY` are out of bounds (clamped to 0..width-1/height-1) or `hit < 100`:
     - If `sideDistX < sideDistY`:
       - Add `deltaDistX` to `sideDistX`.
       - Increment `mapX` by `stepX`.
       - Set `side = 0`.
     - Else:
       - Add `deltaDistY` to `sideDistY`.
       - Increment `mapY` by `stepY`.
       - Set `side = 1`.
     - Clamp `mapX` to `[0, mapHeight-1]` and `mapY` to `[0, mapWidth-1]` [S1, unofficial].
     - Increment `hit`.

4. **Render Voxel Layers**:
   - Calculate `perpWallDist`:
     - If `side == 0`, `perpWallDist = (mapX - rayPosX + (1 - stepX) / 2) / rayDirX`.
     - If `side == 1`, `perpWallDist = (mapY - rayPosY + (1 - stepY) / 2) / rayDirY` [S1, unofficial].
   - Calculate `lineHeight = (int)(screenHeight / perpWallDist)`. Note: The source code uses this formula without dividing by 2 for square voxels, despite a comment suggesting it [S1, unofficial].
   - Initialize `depth` array of size 384 with 100000.
   - Loop `b` from 0 to 11:
     - Calculate `drawStart = ((lineHeight)*(b-6)) + pitch`.
     - Clamp `drawStart` to `[0, screenHeight-1]`.
     - Calculate `drawEnd = lineHeight + ((lineHeight)*(b-6)) + pitch`.
     - Clamp `drawEnd` to `[0, screenHeight-1]`.
     - Retrieve color from `worldMap[mapX][mapY][b]`.
     - If `side == 1`, divide color brightness by 2.
     - If color is not Black, call `verLineDepth(x, drawStart, drawEnd, color, depth, 384, &count)` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def voxel_raycasting_vertical_slicing(world_map, map_width, map_height, camera_pos, camera_dir, camera_plane, screen_width, screen_height, pitch=200):
    screen_buffer = [[0 for _ in range(screen_height)] for _ in range(screen_width)]
    
    for x in range(screen_width):
        camera_x = 2 * x / screen_width - 1
        ray_dir_x = camera_dir[0] + camera_plane[0] * camera_x
        ray_dir_y = camera_dir[1] + camera_plane[1] * camera_x
        
        ray_pos_x = camera_pos[0]
        ray_pos_y = camera_pos[1]
        
        map_x = int(ray_pos_x)
        map_y = int(ray_pos_y)
        
        if ray_dir_x < 0:
            step_x = -1
            side_dist_x = (ray_pos_x - map_x) * math.sqrt(1 + (ray_dir_y * ray_dir_y) / (ray_dir_x * ray_dir_x))
        else:
            step_x = 1
            side_dist_x = (map_x + 1.0 - ray_pos_x) * math.sqrt(1 + (ray_dir_y * ray_dir_y) / (ray_dir_x * ray_dir_x))
            
        if ray_dir_y < 0:
            step_y = -1
            side_dist_y = (ray_pos_y - map_y) * math.sqrt(1 + (ray_dir_x * ray_dir_x) / (ray_dir_y * ray_dir_y))
        else:
            step_y = 1
            side_dist_y = (map_y + 1.0 - ray_pos_y) * math.sqrt(1 + (ray_dir_x * ray_dir_x) / (ray_dir_y * ray_dir_y))
            
        delta_dist_x = math.sqrt(1 + (ray_dir_y * ray_dir_y) / (ray_dir_x * ray_dir_x))
        delta_dist_y = math.sqrt(1 + (ray_dir_x * ray_dir_x) / (ray_dir_y * ray_dir_y))
        
        hit = 0
        side = 0
        
        while (0 <= map_x < map_width and 0 <= map_y < map_height) and hit < 100:
            if side_dist_x < side_dist_y:
                side_dist_x += delta_dist_x
                map_x += step_x
                side = 0
            else:
                side_dist_y += delta_dist_y
                map_y += step_y
                side = 1
            hit += 1
            
        if side == 0:
            perp_wall_dist = (map_x - ray_pos_x + (1 - step_x) / 2) / ray_dir_x
        else:
            perp_wall_dist = (map_y - ray_pos_y + (1 - step_y) / 2) / ray_dir_y
            
        line_height = int(screen_height / perp_wall_dist)
        
        depth_buffer = [100000] * 384
        
        for b in range(12):
            draw_start = (line_height * (b - 6)) + pitch
            draw_end = line_height + (line_height * (b - 6)) + pitch
            
            draw_start = max(0, min(screen_height - 1, draw_start))
            draw_end = max(0, min(screen_height - 1, draw_end))
            
            color = world_map[map_x][map_y][b]
            
            if side == 1:
                color = color // 2
                
            if color != 0:
                # Assuming verLineDepth is a function that draws a vertical line with depth testing
                # This is a placeholder for the actual drawing logic
                for y in range(draw_start, draw_end + 1):
                    if depth_buffer[y] > hit:
                        screen_buffer[x][y] = color
                        depth_buffer[y] = hit
                        
    return screen_buffer
```

### Parameters
- `pitch`: Default value is 200. It shifts the vertical position of the voxel layers on the screen [S1, unofficial].
- `cameraPlaneY`: Value is 0.66. The horizontal component `planeX` is 0 [S1, unofficial].

### Edge Cases and Pitfalls
- **Voxel Aspect Ratio**: The source code contains a comment indicating uncertainty about voxel aspect ratio: 'This needs to be divided by 2 to make square voxels, but then it crashes. ??? wait nvm'. The implementation uses `h / perpWallDist` without the division by 2 [S1, unofficial].
- **Boundary Conditions**: The DDA loop clamps `mapX` and `mapY` to `[0, mapHeight-1]` and `[0, mapWidth-1]` respectively. If the ray goes out of bounds, the loop terminates [S1, unofficial].
- **Depth Buffer Reset**: The depth buffer is reset per ray in the loop. This suggests per-ray depth management, but the array is passed to `verLineDepth`, which might imply global depth management depending on its implementation [S1, unofficial].

## Operating characteristics
- **Time Complexity**: O(N) where N is the number of screen columns, as each column requires a DDA traversal and 12 layer draws.
- **Memory Complexity**: O(W * H) for the screen buffer and O(W) for the depth buffer per ray.
- **Measured Performance**: Not given in the sources.
- **Scaling**: The method scales linearly with screen width. The number of layers (12) is fixed, so it does not scale with resolution.
- **Strengths**: Visualizes voxel thickness and internal structure. Efficient DDA traversal.
- **Limitations**: Fixed number of layers (12). Potential aspect ratio issues. Unclear depth management across rays.
- **When to Use**: When visualizing voxel depth is important and performance is not critical.
- **When Not to Use**: When high performance is required or when more than 12 layers are needed.

## Background
- **Origin**: The method is described in the `voxnine` repository by JCLemme [S1, unofficial].
- **History and Development**: The method is an extension of 2D raycasting, adding vertical layers to represent 3D voxels.
- **Where it is Used**: Used in the `voxnine` project for rendering 3D voxel grids [S1, unofficial].

## Sources
[S1] JCLemme/voxnine, https://github.com/JCLemme/voxnine, fetched 2026-10-28 (unofficial)
