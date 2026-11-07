# Dual-Layer DDA Raycasting

Also called: 3D Raycaster, 3D Raycasting with Dual-Layer DDA, DDA Raycasting, DDA impl, Digital Differential Analyzer Raycasting

## Overview
Dual-Layer DDA Raycasting is a real-time rendering technique that extends standard voxel raycasting by supporting two distinct vertical layers (`worldMap[0]` for ceilings and `worldMap[1]` for floors) within a single ray traversal pass [S2, unofficial]. Unlike [Classic (Column Voxel Space)](5024-classic-column-voxel-space.md), which typically handles a single layer, this method simultaneously determines the first occupied cell for both layers using a 3D Digital Differential Analyzer (DDA) [S2, unofficial]. It builds on [Voxel Raycasting with DDA and Vertical Stripe Rendering](2206-voxel-raycasting-with-dda-and-vertical-stripe-rendering.md) by adding the capacity to render both top and bottom surfaces of a voxel column in one go [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A 3D voxel grid `worldMap` with dimensions $D_x \times D_y \times 2$ (where the third dimension represents the two layers: 0 for top/ceiling, 1 for bottom/floor) [S2, unofficial]. Camera position $(rayPosX, rayPosY, rayPosZ)$ and direction $(rayDirX, rayDirY, rayDirZ)$ [S2, unofficial]. Screen height $h$ [S2, unofficial].
*   **Outputs:** A rendered image where each screen column displays vertical stripes for both the ceiling (layer 0) and floor (layer 1) hits, with correct depth and texture coordinates [S2, unofficial].

**Data Structures:**
*   `worldMap`: A 3D array or flattened structure indexed as `worldMap[layer][x][y]` or equivalent, where `layer \in \{0, 1\}` [S2, unofficial].
*   `rayPos`: 3D vector $(rayPosX, rayPosY, rayPosZ)$ [S2, unofficial].
*   `rayDir`: 3D vector $(rayDirX, rayDirY, rayDirZ)$ [S2, unofficial].
*   `sideDist`: 3D vector tracking distance to the next grid boundary for each axis [S2, unofficial].
*   `deltaDist`: 3D vector storing the distance the ray travels to cross one cell in each axis [S2, unofficial].
*   `step`: 3D vector storing the direction of increment (-1 or 1) for each axis [S2, unofficial].

**Parameters:**
*   `max_distance`: The maximum distance the ray travels before stopping if no wall is found. Typical value: 1000 [S1, unofficial].
*   `h`: Screen height in pixels [S2, unofficial].

**Step-by-Step Method:**

1.  **Initialization:**
    For each screen column $x$:
    a.  Compute the ray direction vector $(rayDirX, rayDirY, rayDirZ)$ based on the camera position, direction vector, and camera plane [S2, unofficial].
    b.  Initialize the current map cell coordinates:
        $$mapX = \lfloor rayPosX \rfloor, \quad mapY = \lfloor rayPosY \rfloor, \quad mapZ = \lfloor rayPosZ \rfloor$$
        [S2, unofficial]
    c.  Calculate the step directions for X, Y, and Z axes [S2, unofficial]:
        $$stepX = \begin{cases} -1 & \text{if } rayDirX < 0 \\ 1 & \text{otherwise} \end{cases}, \quad stepY = \begin{cases} -1 & \text{if } rayDirY < 0 \\ 1 & \text{otherwise} \end{cases}, \quad stepZ = \begin{cases} -1 & \text{if } rayDirZ < 0 \\ 1 & \text{otherwise} \end{cases}$$
        [S2, unofficial]
    d.  Compute delta distances for each axis [S2, unofficial]:
        $$deltaDistX = \sqrt{1 + \left(\frac{rayDirY}{rayDirX}\right)^2 + \left(\frac{rayDirZ}{rayDirX}\right)^2}$$
        [S2, unofficial]
        $$deltaDistY = \sqrt{1 + \left(\frac{rayDirX}{rayDirY}\right)^2 + \left(\frac{rayDirZ}{rayDirY}\right)^2}$$
        [S2, unofficial]
        $$deltaDistZ = \sqrt{1 + \left(\frac{rayDirX}{rayDirZ}\right)^2 + \left(\frac{rayDirY}{rayDirZ}\right)^2}$$
        [S2, unofficial]
    e.  Initialize side distances [S2, unofficial]:
        $$sideDistX = \begin{cases} (rayPosX - mapX) \cdot deltaDistX & \text{if } rayDirX < 0 \\ (mapX + 1.0 - rayPosX) \cdot deltaDistX & \text{otherwise} \end{cases}$$
        [S2, unofficial]
        (Similarly for $sideDistY$ and $sideDistZ$) [S2, unofficial].
    f.  Initialize flags `hitA = false` (for layer 0) and `hitB = false` (for layer 1) [S2, unofficial].

2.  **DDA Loop:**
    Perform the DDA loop while `!hitA || !hitB` [S2, unofficial]:
    a.  Compare `sideDistX`, `sideDistY`, and `sideDistZ` [S2, unofficial].
    b.  If `sideDistX` is the smallest:
        $$sideDistX += deltaDistX, \quad mapX += stepX, \quad side = 0$$
        [S2, unofficial]
    c.  Else if `sideDistY` is the smallest:
        $$sideDistY += deltaDistY, \quad mapY += stepY, \quad side = 1$$
        [S2, unofficial]
    d.  Else (if `sideDistZ` is the smallest):
        $$sideDistZ += deltaDistZ, \quad mapZ += stepZ, \quad side = 2$$
        [S2, unofficial]
    e.  Check for wall hit in layer 0 (hitA):
        If `worldMap[0][mapX][mapY] > 0` and `!hitA`, set `hitA = true` [S2, unofficial].
        Calculate perpendicular wall distance:
        $$perpWallDist = \begin{cases} \frac{mapX - rayPosX + (1 - stepX) / 2}{rayDirX} \cdot \frac{1}{2} & \text{if } side == 0 \\ \frac{mapY - rayPosY + (1 - stepY) / 2}{rayDirY} \cdot \frac{1}{2} & \text{if } side == 1 \\ \frac{mapZ - rayPosZ + (1 - stepZ) / 2}{rayDirZ} \cdot \frac{1}{2} & \text{if } side == 2 \end{cases}$$
        [S2, unofficial]
    f.  Check for wall hit in layer 1 (hitB):
        If `worldMap[1][mapX][mapY] > 0` and `!hitB`, set `hitB = true` [S2, unofficial].
        Calculate `perpWallDist` using the same logic as in step 2e [S2, unofficial].

3.  **Rendering:**
    a.  **Render Layer 0 Stripe (Ceiling):**
        If `hitA` is true:
        $$lineHeight = \left\lfloor \frac{h}{perpWallDist} \right\rfloor$$
        [S2, unofficial]
        $$drawStart = \max\left(0, \left\lfloor \frac{h}{2} \right\rfloor\right)$$
        [S2, unofficial]
        $$drawEnd = \min\left(h - 1, \left\lfloor \frac{lineHeight}{2} + \frac{h}{2} \right\rfloor\right)$$
        [S2, unofficial]
        Draw a vertical line from `drawStart` to `drawEnd` with color based on `worldMap[0][mapX][mapY]`, halving brightness if `side == 1` [S2, unofficial].
    b.  **Render Layer 1 Stripe (Floor):**
        If `hitB` is true:
        $$lineHeight = \left\lfloor \frac{h}{perpWallDist} \right\rfloor$$
        [S2, unofficial]
        $$drawStart = \max\left(0, \left\lfloor -\frac{lineHeight}{2} + \frac{h}{2} \right\rfloor\right)$$
        [S2, unofficial]
        $$drawEnd = \min\left(h - 1, \left\lfloor \frac{h}{2} \right\rfloor\right)$$
        [S2, unofficial]
        Draw a vertical line from `drawStart` to `drawEnd` with color based on `worldMap[1][mapX][mapY]`, halving brightness if `side == 1` [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dual_layer_dda_raycasting(world_map, camera_pos, camera_dir, screen_height, max_distance):
    """
    Renders ceiling (layer 0) and floor (layer 1) stripes for each screen column.
    world_map: 3D array [2][Dx][Dy]
    camera_pos: (rayPosX, rayPosY, rayPosZ)
    camera_dir: (rayDirX, rayDirY, rayDirZ)
    screen_height: int h
    max_distance: float
    """
    rayPosX, rayPosY, rayPosZ = camera_pos
    rayDirX, rayDirY, rayDirZ = camera_dir
    
    # Initialize map coordinates
    mapX = int(rayPosX)
    mapY = int(rayPosY)
    mapZ = int(rayPosZ)
    
    # Calculate step directions
    stepX = -1 if rayDirX < 0 else 1
    stepY = -1 if rayDirY < 0 else 1
    stepZ = -1 if rayDirZ < 0 else 1
    
    # Guard against division by zero [G]
    eps = 1e-9
    rdx = rayDirX if abs(rayDirX) > eps else eps
    rdy = rayDirY if abs(rayDirY) > eps else eps
    rdz = rayDirZ if abs(rayDirZ) > eps else eps
    
    # Calculate delta distances
    deltaDistX = math.sqrt(1 + (rdy / rdx)**2 + (rdz / rdx)**2)
    deltaDistY = math.sqrt(1 + (rdx / rdy)**2 + (rdz / rdy)**2)
    deltaDistZ = math.sqrt(1 + (rdx / rdz)**2 + (rdy / rdz)**2)
    
    # Calculate initial side distances
    if rayDirX < 0:
        sideDistX = (rayPosX - mapX) * deltaDistX
    else:
        sideDistX = (mapX + 1.0 - rayPosX) * deltaDistX
        
    if rayDirY < 0:
        sideDistY = (rayPosY - mapY) * deltaDistY
    else:
        sideDistY = (mapY + 1.0 - rayPosY) * deltaDistY
        
    if rayDirZ < 0:
        sideDistZ = (rayPosZ - mapZ) * deltaDistZ
    else:
        sideDistZ = (mapZ + 1.0 - rayPosZ) * deltaDistZ
        
    hitA = False  # Layer 0 hit
    hitB = False  # Layer 1 hit
    side = 0      # 0: vertical face (X), 1: horizontal face (Y), 2: depth face (Z)
    perpWallDist = 0.0
    
    # DDA Loop
    while not hitA or not hitB:
        # Determine which side to step
        if sideDistX < sideDistY:
            if sideDistX < sideDistZ:
                sideDistX += deltaDistX
                mapX += stepX
                side = 0
            else:
                sideDistZ += deltaDistZ
                mapZ += stepZ
                side = 2
        else:
            if sideDistY < sideDistZ:
                sideDistY += deltaDistY
                mapY += stepY
                side = 1
            else:
                sideDistZ += deltaDistZ
                mapZ += stepZ
                side = 2
                
        # Check bounds and hits
        if 0 <= mapX < world_map.shape[1] and 0 <= mapY < world_map.shape[2]:
            if not hitA and world_map[0][mapX][mapY] > 0:
                hitA = True
                # Calculate perpWallDist for layer 0
                if side == 0:
                    perpWallDist = (mapX - rayPosX + (1 - stepX) / 2) / rdx / 2
                elif side == 1:
                    perpWallDist = (mapY - rayPosY + (1 - stepY) / 2) / rdy / 2
                elif side == 2:
                    perpWallDist = (mapZ - rayPosZ + (1 - stepZ) / 2) / rdz / 2
                    
            if not hitB and world_map[1][mapX][mapY] > 0:
                hitB = True
                # Calculate perpWallDist for layer 1 (same logic)
                if side == 0:
                    perpWallDist = (mapX - rayPosX + (1 - stepX) / 2) / rdx / 2
                elif side == 1:
                    perpWallDist = (mapY - rayPosY + (1 - stepY) / 2) / rdy / 2
                elif side == 2:
                    perpWallDist = (mapZ - rayPosZ + (1 - stepZ) / 2) / rdz / 2
                    
    # Rendering
    # Layer 0 (Ceiling)
    if hitA:
        lineHeight = int(screen_height / perpWallDist)
        drawStart = max(0, int(screen_height / 2))
        drawEnd = min(screen_height - 1, int(lineHeight / 2 + screen_height / 2))
        # Draw vertical line from drawStart to drawEnd with color from world_map[0][mapX][mapY]
        # Halve brightness if side == 1
        
    # Layer 1 (Floor)
    if hitB:
        lineHeight = int(screen_height / perpWallDist)
        drawStart = max(0, int(-lineHeight / 2 + screen_height / 2))
        drawEnd = min(screen_height - 1, int(screen_height / 2))
        # Draw vertical line from drawStart to drawEnd with color from world_map[1][mapX][mapY]
        # Halve brightness if side == 1
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If `rayDirX`, `rayDirY`, or `rayDirZ` is zero, the delta distance calculations involve division by zero, resulting in NaN or infinity values [S1, unofficial]. This must be handled by checking for zero direction components and using alternative formulas or clamping [G].
*   **Boundary Conditions:** The algorithm must handle cases where the ray exits the map bounds without hitting a wall, potentially requiring a default background color or termination condition [S2, unofficial].

## Subtopics

### [Voxel Raycasting with Vertical Slicing](2296-voxel-raycasting-with-vertical-slicing.md)
Renders each voxel column as 12 distinct vertical layers, iterating through vertical indices to draw stacked horizontal stripes. Choose when internal voxel structure and thickness visualization are required.

### [Dual-Cell DDA Raycasting for Particles](5053-dual-cell-dda-raycasting-for-particles.md)
Uses a 3D DDA over dual cells for particle rendering with glossy and glass materials. Choose for particle systems requiring refraction and reflection.

### [Raycasting Texture Coordinate Calculation](5470-raycasting-texture-coordinate-calculation.md)
Computes the horizontal texture coordinate for a raycast hit against a grid-aligned wall. Choose when correct texture mapping on vertical and horizontal faces is required.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G]
*   **Memory Complexity:** Not given in the sources. [G]
*   **Measured Performance:** Not given in the sources. [G]
*   **Strengths:** Simultaneous rendering of ceiling and floor layers reduces the number of ray casts per column [S2, unofficial].
*   **Limitations:** Limited to two layers [S2, unofficial]. Requires careful handling of zero direction components [S2, unofficial].
*   **When to Use:** Not given in the sources. [G]
*   **When Not to Use:** Not given in the sources. [G]

## Background
Dual-Layer DDA Raycasting is an extension of classic voxel raycasting techniques, specifically designed to handle both ceiling and floor surfaces in a single pass [S2, unofficial]. The method is commonly used in real-time voxel-based games and simulations where efficient rendering of 3D grids is necessary [S2, unofficial]. The specific implementation details, such as the use of two distinct layers and the rendering of vertical stripes, are documented in various online repositories and tutorials [S2, unofficial].

## Sources
[S1] 3D/ray/casting/dda/main.odin, https://github.com/algo-boyz/odins-raylib/blob/52b8fddb/3D/ray/casting/dda/main.odin, fetched 2026-10-25 (unofficial)
[S2] JCLemme/voxnine, https://github.com/JCLemme/voxnine, fetched 2026-10-28 (unofficial)
