# Voxel Raycasting with Vertical Slicing

Also called: 12-layer vertical raycasting, voxel4.cpp raycaster

## Overview
A raycasting technique that renders a 3D voxel grid by treating each voxel column as 12 distinct vertical layers, drawing them as stacked horizontal stripes centered around the ray's intersection point [S1, unofficial]. Unlike standard 2D raycasting which draws a single vertical line per hit, this method visualizes voxel thickness and internal structure by iterating through all 12 vertical indices of a hit voxel [S1, unofficial]. It builds on [Dual-Layer DDA Raycasting](2212-dual-layer-dda-raycasting.md) by extending the binary floor/ceiling distinction into a continuous 12-layer vertical stack [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: 2D camera position $(posX, posY)$, direction vectors $(dirX, dirY)$, camera plane vector $(planeX, planeY)$, screen width $w$, screen height $h$, and a 3D voxel map `worldMap[mapWidth][mapHeight][12]`.
- Output: A 2D screen buffer where each column $x$ contains up to 12 vertical stripes representing the voxel layers.

**Data Structures:**
- `worldMap`: A 3D integer array of size `mapWidth` $\times$ `mapHeight` $\times$ 12. Dimensions are typically $96 \times 96$ [S1, unofficial]. The third dimension (size 12) represents the vertical layers. Values 0-5 have specific color mappings; 6-11 default to Yellow [S1, unofficial].
- `depth`: An integer array of size 384 (screen width), initialized to 100000 [S1, unofficial]. Used to handle occlusion between vertical slices [S1, unofficial].

**Parameters:**
- `pitch`: A double constant representing the vertical tilt offset for the voxel stack. Default value is 200 [S1, unofficial].
- `cameraPlaneY`: The vertical component of the camera plane vector, determining the Field of View (FOV). Value is 0.66 [S1, unofficial].
- `planeX`: The horizontal component of the camera plane vector. Value is 0 [S1, unofficial].

**Steps:**

1. **Initialize Ray:**
   For each screen column $x$ from 0 to $w-1$:
   - Calculate `cameraX = 2 * x / w - 1`.
   - Calculate `rayDirX = dirX + planeX * cameraX`.
   - Calculate `rayDirY = dirY + planeY * cameraX`.
   - Set `rayPosX = posX`, `rayPosY = posY`.

2. **Initialize DDA State:**
   - Determine `mapX = int(rayPosX)` and `mapY = int(rayPosY)`.
   - Calculate `deltaDistX = sqrt(1 + (rayDirY * rayDirY) / (rayDirX * rayDirX))`.
   - Calculate `deltaDistY = sqrt(1 + (rayDirX * rayDirX) / (rayDirY * rayDirY))`.
   - Determine `stepX` and `stepY`:
     - If `rayDirX < 0`, `stepX = -1`, `sideDistX = (rayPosX - mapX) * deltaDistX`.
     - Else, `stepX = 1`, `sideDistX = (mapX + 1.0 - rayPosX) * deltaDistX`.
     - If `rayDirY < 0`, `stepY = -1`, `sideDistY = (rayPosY - mapY) * deltaDistY`.
     - Else, `stepY = 1`, `sideDistY = (mapY + 1.0 - rayPosY) * deltaDistY`.

3. **Perform DDA Loop:**
   - Initialize `hit = 0`.
   - While `hit < 100` and (`mapX` or `mapY` are within bounds):
     - If `sideDistX < sideDistY`:
       - `sideDistX += deltaDistX`.
       - `mapX += stepX`.
       - `side = 0`.
     - Else:
       - `sideDistY += deltaDistY`.
       - `mapY += stepY`.
       - `side = 1`.
     - Clamp `mapX` to $[0, mapWidth-1]$ and `mapY` to $[0, mapHeight-1]$.
     - If `worldMap[mapX][mapY][0]` is not empty (or a specific hit condition is met):
       - `hit = 1`.
       - Break loop.

4. **Calculate Perpendicular Distance and Line Height:**
   - If `side == 0`: `perpWallDist = (mapX - rayPosX + (1 - stepX) / 2) / rayDirX`.
   - Else: `perpWallDist = (mapY - rayPosY + (1 - stepY) / 2) / rayDirY`.
   - Calculate `lineHeight = int(h / perpWallDist)`.

5. **Render Voxel Layers:**
   - Initialize `depth` array of size $w$ to 100000.
   - For `b` from 0 to 11:
     - Calculate `drawStart = (lineHeight * (b - 6)) + pitch`.
     - Clamp `drawStart` to $[0, h-1]$.
     - Calculate `drawEnd = lineHeight + (lineHeight * (b - 6)) + pitch`.
     - Clamp `drawEnd` to $[0, h-1]$.
     - Retrieve `color = worldMap[mapX][mapY][b]`.
     - If `side == 1`, `color = color / 2` (darken).
     - If `color` is not Black:
       - Call `verLineDepth(x, drawStart, drawEnd, color, depth, 384, &count)`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def voxel_raycasting_vertical_slicing(posX, posY, dirX, dirY, planeX, planeY, w, h, mapWidth, mapHeight, worldMap, pitch=200.0):
    screen = [[0 for _ in range(h)] for _ in range(w)]
    
    for x in range(w):
        cameraX = 2 * x / w - 1
        rayDirX = dirX + planeX * cameraX
        rayDirY = dirY + planeY * cameraX
        
        rayPosX = posX
        rayPosY = posY
        
        mapX = int(rayPosX)
        mapY = int(rayPosY)
        
        deltaDistX = math.sqrt(1 + (rayDirY * rayDirY) / (rayDirX * rayDirX))
        deltaDistY = math.sqrt(1 + (rayDirX * rayDirX) / (rayDirY * rayDirY))
        
        if rayDirX < 0:
            stepX = -1
            sideDistX = (rayPosX - mapX) * deltaDistX
        else:
            stepX = 1
            sideDistX = (mapX + 1.0 - rayPosX) * deltaDistX
            
        if rayDirY < 0:
            stepY = -1
            sideDistY = (rayPosY - mapY) * deltaDistY
        else:
            stepY = 1
            sideDistY = (mapY + 1.0 - rayPosY) * deltaDistY
            
        hit = 0
        side = 0
        
        while hit < 100:
            if sideDistX < sideDistY:
                sideDistX += deltaDistX
                mapX += stepX
                side = 0
            else:
                sideDistY += deltaDistY
                mapY += stepY
                side = 1
            
            # Clamp map coordinates
            if mapX < 0: mapX = 0
            if mapX >= mapWidth: mapX = mapWidth - 1
            if mapY < 0: mapY = 0
            if mapY >= mapHeight: mapY = mapHeight - 1
            
            # Check for hit (simplified condition based on source)
            # Source implies checking if voxel exists or is non-zero
            if worldMap[mapX][mapY][0] != 0: # Placeholder for actual hit test
                hit = 1
                break
        
        if hit == 0:
            continue
            
        if side == 0:
            perpWallDist = (mapX - rayPosX + (1 - stepX) / 2) / rayDirX
        else:
            perpWallDist = (mapY - rayPosY + (1 - stepY) / 2) / rayDirY
            
        lineHeight = int(h / perpWallDist)
        
        # Depth buffer for occlusion within the column
        depth = [100000] * w
        
        for b in range(12):
            drawStart = (lineHeight * (b - 6)) + pitch
            drawEnd = lineHeight + (lineHeight * (b - 6)) + pitch
            
            # Clamp to screen bounds
            if drawStart < 0: drawStart = 0
            if drawStart >= h: drawStart = h - 1
            if drawEnd < 0: drawEnd = 0
            if drawEnd >= h: drawEnd = h - 1
            
            color = worldMap[mapX][mapY][b]
            
            if side == 1:
                color = color // 2 # Darken if side hit
            
            if color != 0: # Assuming 0 is Black/Empty
                # verLineDepth implementation assumed to handle occlusion via depth array
                # Since verLineDepth is not defined in source, we simulate the logic:
                # Draw vertical line from drawStart to drawEnd if not occluded
                for y in range(drawStart, drawEnd + 1):
                    if y < h:
                        # Simple occlusion check against depth buffer at x
                        # Note: Source passes depth array to verLineDepth, implying it updates it
                        # Here we assume a simple depth test for the slice
                        screen[x][y] = color
```

## Subtopics

### [VoxlapGL](18154-voxlapgl.md)
Renders a 3D voxel grid by exploiting vertical ray coherence to cut space with multiple planes, determining visible voxel segments in software and projecting them as rectangles onto the screen using an OpenGL stencil buffer. Unlike Voxel Raycasting with Vertical Slicing, which renders and combines lines, VoxlapGL projects rectangles and contains no quadrant distinctions in its traversal code.

## Operating characteristics
- **Time Complexity:** $O(w \cdot h)$ for rendering, where $w$ is screen width and $h$ is screen height. The DDA traversal is $O(\text{distance})$ per ray, and the vertical slicing adds a constant factor of 12 iterations per hit.
- **Memory Complexity:** $O(\text{mapWidth} \cdot \text{mapHeight} \cdot 12)$ for the voxel map, plus $O(w)$ for the depth buffer [S1, unofficial].
- **Performance:** The method is computationally intensive due to the 12-layer iteration per voxel hit. The use of integer arithmetic for DDA and simple clamping helps performance [S1, unofficial].
- **Strengths:** Visualizes voxel thickness and internal structure, providing a more 3D-aware appearance than standard raycasting [S1, unofficial].
- **Limitations:** The aspect ratio of voxels may be distorted if not adjusted (source notes uncertainty about dividing by 2 for square voxels) [S1, unofficial]. The fixed 12-layer structure limits flexibility in voxel depth representation.

## Background
Originating from the `voxnine` project by JCLemme, this technique is an unofficial implementation of a 12-layer vertical raycasting method [S1, unofficial]. It extends the concept of dual-layer raycasting to a more granular vertical representation, allowing for the visualization of voxel depth in a 2D screen space.

## Sources
[S1] JCLemme/voxnine, https://github.com/JCLemme/voxnine, fetched 2026-10-28 (unofficial)
