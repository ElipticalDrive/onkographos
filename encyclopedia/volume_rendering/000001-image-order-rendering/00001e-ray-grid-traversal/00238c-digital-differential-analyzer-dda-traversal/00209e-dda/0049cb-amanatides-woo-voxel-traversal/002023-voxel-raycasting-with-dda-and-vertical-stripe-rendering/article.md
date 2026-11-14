# Voxel Raycasting with DDA and Vertical Stripe Rendering

Also called: 3D Raycasting, Vox5

## Overview
A real-time rendering technique for 3D voxel grids that uses a 2D Digital Differential Analyzer (DDA) to cast rays per screen column, determining the first occupied voxel [S1, unofficial]. It renders the voxel as a vertical stripe (line) rather than a full textured quad, handling depth via a per-column depth buffer to support multiple voxels per column (stacking) [S1, unofficial].

## How it works
**Inputs:**
- `worldMap`: 3D array of `ColorRGB` values, dimensions `[mapWidth][mapHeight][mapDepth]` [S1, unofficial].
- `posX, posY, posZ`: Camera position coordinates [S1, unofficial].
- `dirX, dirY`: Camera direction vector components [S1, unofficial].
- `planeX, planeY`: Camera plane vector components controlling FOV [S1, unofficial].
- `pitch`: Vertical offset for centering the view [S1, unofficial].
- `h`: Screen height in pixels [S1, unofficial].
- `mapWidth, mapHeight, mapDepth`: Dimensions of the voxel grid [S1, unofficial].

**Outputs:**
- Rendered screen image with vertical stripes representing voxel slices [S1, unofficial].
- Updated depth buffers `depth[0]` and `depth[1]` [S1, unofficial].

**Data Structures:**
- `worldMap[mapWidth][mapHeight][mapDepth]`: 3D array. `mapWidth=96`, `mapHeight=96`, `mapDepth=12` [S1, unofficial]. `ColorRGB` is a struct with 3 components (R, G, B) [S1, unofficial].
- `depth[2][384]`: Two integer arrays of size 384 (screen height). Initialized to 100000. Used to store Z-depth of the last drawn voxel at each screen row to handle transparency/overdrawing within a column [S1, unofficial].
- `side`: Integer (0 or 1) indicating which side of the voxel was hit (0 for X-side, 1 for Y-side) [S1, unofficial].

**Steps:**

1. **Initialize Camera Ray:**
   Calculate ray direction and plane vectors based on camera state [S1, unofficial]. The FOV is controlled by `planeY = 0.66` [S1, unofficial].

2. **Determine Step Direction and Initial Side Distance:**
   For each screen column `x` (from 0 to 383):
   - Calculate ray direction:
     $$dirX = dirX_{base} + planeX \cdot (2x - 384) / 384$$
     $$dirY = dirY_{base} + planeY \cdot (2x - 384) / 384$$
   - Calculate initial map position:
     $$mapX = \lfloor posX \rfloor$$
     $$mapY = \lfloor posY \rfloor$$
   - Calculate delta distances (step sizes) [G]:
     $$deltaDistX = 1.0 / |dirX| \quad \text{if } dirX \neq 0 \text{ else } \infty$$
     $$deltaDistY = 1.0 / |dirY| \quad \text{if } dirY \neq 0 \text{ else } \infty$$
   - Calculate initial side distances:
     If `dirX < 0`:
       $$stepX = -1$$
       $$sideDistX = (posX - mapX) \cdot deltaDistX$$
     Else:
       $$stepX = 1$$
       $$sideDistX = (mapX + 1.0 - posX) \cdot deltaDistX$$
     If `dirY < 0`:
       $$stepY = -1$$
       $$sideDistY = (posY - mapY) \cdot deltaDistY$$
     Else:
       $$stepY = 1$$
       $$sideDistY = (mapY + 1.0 - posY) \cdot deltaDistY$$

3. **Perform DDA Loop:**
   Initialize `hit = 0`, `count = 0`.
   While `mapX < 0 || mapY < 0 || (count < 384 && hit < 900)`:
   - If `sideDistX < sideDistY`:
     $$sideDistX += deltaDistX$$
     $$mapX += stepX$$
     $$side = 0$$
   - Else:
     $$sideDistY += deltaDistY$$
     $$mapY += stepY$$
     $$side = 1$$
   - Clamp `mapX` to `[0, mapHeight-1]` and `mapY` to `[0, mapWidth-1]` [S1, unofficial].
   - Increment `hit`.
   - If `worldMap[mapX][mapY][0]` is not empty (or some hit condition is met):
     - Break loop.
   - Increment `count`.

4. **Calculate Perpendicular Wall Distance:**
   If `side == 0`:
     $$perpWallDist = (mapX - posX + (1 - stepX) / 2) / dirX$$
   Else:
     $$perpWallDist = (mapY - posY + (1 - stepY) / 2) / dirY$$
   This corrects for the fisheye effect [S1, unofficial].

5. **Calculate Line Height:**
   $$lineHeight = \lfloor h / perpWallDist \rfloor$$
   Note: The code comments mention dividing by 2 for square voxels but notes it crashes, so it is not divided [S1, unofficial].

6. **Iterate Through Depth Layers:**
   For `b` from 0 to `mapDepth - 1`:
   - Calculate `drawStart`:
     $$drawStart = lineHeight \cdot (b - posZ) + pitch$$
     Clamp `drawStart` to `[0, h-1]`.
   - Calculate `drawEnd`:
     $$drawEnd = lineHeight + lineHeight \cdot (b - posZ) + pitch$$
     Clamp `drawEnd` to `[0, h-1]`.
   - Get color: `color = worldMap[mapX][mapY][b]` [S1, unofficial].
   - Apply shading: If `side == 1`, `color = color / 2` [S1, unofficial].
   - Render voxel slice: If `color != RGB_Black`, call `verLineTriDepth(x, drawStart, drawEnd, color, depth[0], 384, &count, 0, depth[1])`. This function draws the vertical line segment, updating the depth buffer `depth[0]` and `depth[1]` to handle occlusion within the column [S1, unofficial].

7. **Render Second Pass (Horizontal Fill/Edge):**
   - Copy current state to `tmapX, tmapY, tside` [S1, unofficial].
   - Advance one step again using the same DDA logic (`if(sideDistX < sideDistY)...`). Clamp `tmapX, tmapY` [S1, unofficial].
   - Calculate `tperpWallDist` and `tlineHeight` using the same formulas as above [S1, unofficial].
   - For `b` from 0 to `mapDepth - 1`:
     - Calculate `tdrawStart`:
       $$tdrawStart = tlineHeight \cdot (b - posZ) + pitch$$
       Clamp `tdrawStart` to `[0, h-1]`.
     - Calculate `tdrawEnd`:
       $$tdrawEnd = tlineHeight + tlineHeight \cdot (b - posZ) + pitch$$
       Clamp `tdrawEnd` to `[0, h-1]`.
     - Get `tcolor = worldMap[mapX][mapY][b]` [S1, unofficial].
     - Apply shading: `tcolor = tcolor / 3` [S1, unofficial].
     - Call `verLineTriDepth` with `drawStart` or `tdrawStart` depending on `b < posZ`, and `depth[1]` as the primary buffer [S1, unofficial].

8. **Swap Depth Buffers:**
   After processing all depth layers for a voxel hit, swap depth buffers:
   $$memcpy(depth[0], depth[1], 384 \cdot sizeof(int))$$
   This prepares `depth[1]` for the next ray or pass [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_voxel_column(x, posX, posY, posZ, dirX, dirY, planeX, planeY, worldMap, depth, h, pitch, mapWidth, mapHeight, mapDepth):
    # Step 1: Initialize Camera Ray
    rayDirX = dirX + planeX * (2 * x - 384) / 384
    rayDirY = dirY + planeY * (2 * x - 384) / 384

    # Step 2: Determine Step Direction and Initial Side Distance
    mapX = int(posX)
    mapY = int(posY)

    # Delta distances [G]
    rayDirX_abs = abs(rayDirX)
    rayDirY_abs = abs(rayDirY)
    deltaDistX = 1.0 / rayDirX_abs if rayDirX_abs > 0 else float('inf')
    deltaDistY = 1.0 / rayDirY_abs if rayDirY_abs > 0 else float('inf')

    if rayDirX < 0:
        stepX = -1
        sideDistX = (posX - mapX) * deltaDistX
    else:
        stepX = 1
        sideDistX = (mapX + 1.0 - posX) * deltaDistX

    if rayDirY < 0:
        stepY = -1
        sideDistY = (posY - mapY) * deltaDistY
    else:
        stepY = 1
        sideDistY = (mapY + 1.0 - posY) * deltaDistY

    hit = 0
    count = 0
    side = 0

    # Step 3: Perform DDA Loop
    while mapX < 0 or mapY < 0 or (count < 384 and hit < 900):
        if sideDistX < sideDistY:
            sideDistX += deltaDistX
            mapX += stepX
            side = 0
        else:
            sideDistY += deltaDistY
            mapY += stepY
            side = 1
        
        # Clamp mapX and mapY
        mapX = max(0, min(mapX, mapHeight - 1))
        mapY = max(0, min(mapY, mapWidth - 1))
        
        hit += 1
        
        # Check for hit (assuming non-black color indicates a voxel)
        if mapX < mapWidth and mapY < mapHeight and mapDepth > 0:
            if worldMap[mapX][mapY][0] != (0, 0, 0):
                break
        count += 1

    # Step 4: Calculate Perpendicular Wall Distance
    if side == 0:
        perpWallDist = (mapX - posX + (1 - stepX) / 2) / rayDirX
    else:
        perpWallDist = (mapY - posY + (1 - stepY) / 2) / rayDirY

    # Step 5: Calculate Line Height
    lineHeight = int(h / perpWallDist)

    # Step 6: Iterate Through Depth Layers
    for b in range(mapDepth):
        drawStart = int(lineHeight * (b - posZ) + pitch)
        drawStart = max(0, min(drawStart, h - 1))
        
        drawEnd = lineHeight + lineHeight * (b - posZ) + pitch
        drawEnd = max(0, min(drawEnd, h - 1))
        
        color = worldMap[mapX][mapY][b]
        
        # Apply shading
        if side == 1:
            color = tuple(c / 2 for c in color)
        
        if color != (0, 0, 0):
            # Inline verLineTriDepth logic for depth update
            for y in range(drawStart, drawEnd + 1):
                if 0 <= y < 384:
                    depth[0][y] = b
                    depth[1][y] = b

    # Step 7: Render Second Pass (Horizontal Fill/Edge)
    tmapX, tmapY, tside = mapX, mapY, side
    tsideDistX, tsideDistY = sideDistX, sideDistY
    
    # Advance one step again
    if tsideDistX < tsideDistY:
        tsideDistX += deltaDistX
        tmapX += stepX
        tside = 0
    else:
        tsideDistY += deltaDistY
        tmapY += stepY
        tside = 1
    
    tmapX = max(0, min(tmapX, mapHeight - 1))
    tmapY = max(0, min(tmapY, mapWidth - 1))
    
    if tside == 0:
        tperpWallDist = (tmapX - posX + (1 - stepX) / 2) / rayDirX
    else:
        tperpWallDist = (tmapY - posY + (1 - stepY) / 2) / rayDirX
    
    tlineHeight = int(h / tperpWallDist)
    
    for b in range(mapDepth):
        tdrawStart = int(tlineHeight * (b - posZ) + pitch)
        tdrawStart = max(0, min(tdrawStart, h - 1))
        
        tdrawEnd = tlineHeight + tlineHeight * (b - posZ) + pitch
        tdrawEnd = max(0, min(tdrawEnd, h - 1))
        
        tcolor = worldMap[mapX][mapY][b]
        tcolor = tuple(c / 3 for c in tcolor)
        
        # Call verLineTriDepth with depth[1] as primary buffer
        start = drawStart if b < posZ else tdrawStart
            
        # Simulate depth update for depth[1]
        for y in range(start, tdrawEnd + 1):
            if 0 <= y < 384:
                depth[1][y] = b

    # Step 8: Swap Depth Buffers
    depth[0][:] = depth[1][:]
```

## Subtopics

### [Dual-Layer DDA Raycasting](2212-dual-layer-dda-raycasting.md)
Dual-Layer DDA Raycasting is a real-time rendering technique that extends standard voxel raycasting by supporting two distinct vertical layers (`worldMap[0]` for ceilings and `worldMap[1]` for floors) within a single ray traversal pass. Unlike [Classic (Column Voxel Space)](5024-classic-column-voxel-space.md), which typically handles a single layer, this method simultaneously determines the first occupied cell for both layers using a 3D Digital Differential Analyzer (DDA). It builds on [Voxel Raycasting with DDA and Vertical Stripe Rendering](2206-voxel-raycasting-with-dda-and-vertical-stripe-rendering.md) by adding the capacity to render both top and bottom surfaces of a voxel column in one go.

### [Classic (Column Voxel Space)](5024-classic-column-voxel-space.md)
Classic rendering is a real-time terrain visualization technique that redraws the camera frustum every frame by marching world-vertical columns from the camera to the horizon. It differs from its sibling [Voxel Raycasting with DDA and Vertical Stripe Rendering](2206-voxel-raycasting-with-dda-and-vertical-stripe-rendering.md) by casting one ray per screen column rather than per pixel, and from [Dual-Layer DDA Raycasting](2212-dual-layer-dda-raycasting.md) by using a height-map-based column march instead of a dual-layer DDA approach.

### [Screen-Space Ray Casting with Layer Traversal](5221-screen-space-ray-casting-with-layer-traversal.md)
A real-time rendering technique that performs ray casting in screen space by marching pixel by pixel along the projected ray direction to intersect voxel columns and find entry/exit points for volume rendering. It distinguishes itself from its parent [Voxel Raycasting with DDA and Vertical Stripe Rendering](2206-voxel-raycasting-with-dda-and-vertical-stripe-rendering.md) by explicitly handling multiple surface layers within a column via a layer index, enabling the tracing of multiple refractions and deep ambient occlusion by leveraging full knowledge of surface layers rather than just the front-most surface.

### [Voxel Polygon Rendering](6747-voxel-polygon-rendering.md)
Voxel Polygon Rendering is a hybrid real-time rendering technique that projects a 2D voxel height map onto the screen as vertical bars, solving the problem of rendering high-detail terrain at constant speed without traditional 6DOF polygon operations. It sets itself apart from its sibling [Voxel Raycasting with DDA and Vertical Stripe Rendering](2206-voxel-raycasting-with-dda-and-vertical-stripe-rendering.md) by introducing "voxel polygons" to vary local detail and an s-buffer (vertical c-buffer) to handle floating geometry, whereas the sibling relies on a simpler per-column depth buffer for stacking.

## Operating characteristics
- **Time Complexity:** Not given in the sources. [S1, unofficial]
- **Memory Complexity:** O(W * H * D) for the voxel grid, where W, H, D are the dimensions of the grid [S1, unofficial]. O(H) for the depth buffers, where H is the screen height [S1, unofficial].
- **Measured Performance:** The method is designed for real-time rendering [S1, unofficial]. The specific performance metrics are not given in the sources. [S1, unofficial]
- **Scaling:** Not given in the sources.
- **Strengths:** Real-time performance, simple implementation, supports stacking of voxels within a column [S1, unofficial].
- **Limitations:** Fisheye effect if not corrected, vertically stretched voxels if aspect ratio is not handled correctly, limited to grid-aligned voxels [S1, unofficial].
- **When to use:** Real-time rendering of voxel grids where performance is critical and high visual fidelity is not the primary concern [G].
- **When not to use:** When high visual fidelity, anti-aliasing, or complex lighting effects are required [G].

## Background
The method is described in the source [S1, unofficial], which is an unofficial repository. The specific origin and history are not provided in the sources. [S1, unofficial]

## Sources
[S1] JCLemme/voxnine, https://github.com/JCLemme/voxnine, fetched 2026-10-28 (unofficial)
