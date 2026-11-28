# Wolfenstein 3D Raycasting

Also called: Wolfenstein 3D technique, Wolfenstein-style rendering, Lodev-style DDA, DDA Ray Casting, Raycasting with Fisheye Correction

## Overview
Wolfenstein 3D Raycasting is a rendering technique for first-person corridors on hardware without GPUs or z-buffers, where the world is a flat 2D grid, the camera is confined to the grid plane, and one ray is cast per screen column to reconstruct a textured 3D scene [S1, unofficial]. It sets itself apart from general voxel traversal methods by operating on a strictly 2D grid map with a fixed-height camera, using the perpendicular distance to the camera plane to correct fisheye distortion and determine wall height, rather than computing full 3D intersections [S1, unofficial][S3, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 2D grid map (`Uint8Array` or `number[][]`), player state (`posX`, `posY`, `dirX`, `dirY`, `planeX`, `planeY`), screen dimensions (`FB_W`, `FB_H`), and field of view defined by the plane vector length [S1, unofficial][S3, unofficial].
- **Output:** A framebuffer image containing vertical wall strips and billboard sprites [S1, unofficial].

**Data Structures:**
- **Map:** A 1D or 2D array representing the grid. Values: `0` (empty floor), `1-4` (wall types), `5` (sliding door) [S1, unofficial][S5, unofficial].
- **Player:** Object with `x`, `y` (position), `dirX`, `dirY` (unit direction vector), `planeX`, `planeY` (camera plane vector perpendicular to `dir`) [S1, unofficial][S3, unofficial].
- **Textures:** Procedurally generated at runtime onto a 64x64 offscreen canvas, then read into a `Uint32Array` for fast pixel access [S1, unofficial].
- **Z-Buffer:** A 1D array of size `FB_W` storing the perpendicular distance to the nearest wall for each screen column, used for sprite occlusion [S1, unofficial].

**Parameters:**
- **FOV:** Determined by the length of the plane vector relative to the direction vector. A ratio of `|plane|/|dir| = 0.66` yields a horizontal FOV of ~66.8 degrees [S1, unofficial].
- **Wall Height:** Real-world height of a wall tile, typically 1 unit [S2, unofficial].

**Steps:**

1. **Initialize Camera Vectors:**
   For each screen column `x` from `0` to `FB_W - 1`:
   - Calculate `cameraX = 2 * x / FB_W - 1` (ranges from -1 to 1) [S1, unofficial].
   - Calculate ray direction:
     $$ rayDirX = dirX + planeX \cdot cameraX $$
     $$ rayDirY = dirY + planeY \cdot cameraX $$
     [S1, unofficial][S3, unofficial]

2. **DDA Grid Traversal:**
   - Initialize current grid cell: `mapX = floor(posX)`, `mapY = floor(posY)` [S3, unofficial].
   - Calculate step distances:
     $$ \Delta_x = \left| \frac{1}{rayDirX} \right|, \quad \Delta_y = \left| \frac{1}{rayDirY} \right| $$
     Handle `rayDirX == 0` or `rayDirY == 0` by setting $\Delta$ to a large value (e.g., `1e30`) [S2, unofficial][S3, unofficial].
   - Determine step directions and initial side distances:
     - If `rayDirX < 0`: `stepX = -1`, `sideDistX = (posX - mapX) * \Delta_x` [S3, unofficial].
     - Else: `stepX = 1`, `sideDistX = (mapX + 1 - posX) * \Delta_x` [S3, unofficial].
     - If `rayDirY < 0`: `stepY = -1`, `sideDistY = (posY - mapY) * \Delta_y` [S3, unofficial].
     - Else: `stepY = 1`, `sideDistY = (mapY + 1 - posY) * \Delta_y` [S3, unofficial].
   - Loop until a wall is hit:
     - If `sideDistX < sideDistY`:
       - `sideDistX += \Delta_x`
       - `mapX += stepX`
       - `side = 0` (hit vertical grid line)
     - Else:
       - `sideDistY += \Delta_y`
       - `mapY += stepY`
       - `side = 1` (hit horizontal grid line)
     - Check if `map[mapY][mapX] > 0`. If so, break [S3, unofficial].
     - *Edge Case (Sliding Doors):* If the hit tile is `5`, advance the ray to the center line of the door cell. If the ray exits the cell before reaching the midline or hits the closed part, continue traversal [S1, unofficial].

3. **Calculate Perpendicular Distance:**
   - Compute the fisheye-corrected distance:
     $$ dist = side === 0 ? sideDistX - \Delta_x : sideDistY - \Delta_y $$
     This value represents the distance perpendicular to the camera plane, correcting for the bowing effect of raw ray length [S1, unofficial][S5, unofficial].

4. **Render Wall Strip:**
   - Calculate line height:
     $$ lineHeight = \frac{FB_H}{dist} $$
     [S1, unofficial]
   - Calculate draw range:
     $$ drawStart = \max(0, \lfloor (FB_H - lineHeight) / 2 \rfloor) $$
     $$ drawEnd = \min(FB_H - 1, \lfloor (FB_H + lineHeight) / 2 \rfloor) $$
     [S1, unofficial]
   - Calculate texture coordinate `wallX`:
     - If `side == 0`: `wallX = posY + dist * rayDirY - floor(posY + dist * rayDirY)` [S2, unofficial].
     - Else: `wallX = posX + dist * rayDirX - floor(posX + dist * rayDirX)` [S2, unofficial].
   - Mirror texture X if necessary to prevent backward textures:
     - If `(side == 0 && rayDirX > 0) || (side == 1 && rayDirY < 0)`: `wallX = 1 - wallX` [S1, unofficial].
   - Draw vertical line from `drawStart` to `drawEnd` using `wallX` for texture sampling [S1, unofficial]. Shade based on `side` for depth cues [S1, unofficial][S2, unofficial].
   - Store `dist` in `zbuffer[x]` for sprite occlusion [S1, unofficial].

5. **Render Sprites (Billboards):**
   - Sort sprites by squared distance to player (far to near) [S1, unofficial].
   - For each sprite:
     - Transform sprite position `(sx, sy)` to camera space:
       $$ invDet = \frac{1}{planeX \cdot dirY - dirX \cdot planeY} $$
       $$ transformX = invDet \cdot (dirY \cdot (sx - posX) - dirX \cdot (sy - posY)) $$
       $$ transformY = invDet \cdot (-planeY \cdot (sx - posX) + planeX \cdot (sy - posY)) $$
       [S1, unofficial]
     - If `transformY <= 0.05`, skip (behind camera) [S1, unofficial].
     - Calculate screen X: `screenX = floor((FB_W / 2) * (1 + transformX / transformY))` [S1, unofficial].
     - Calculate sprite size: `size = abs(floor(FB_H / transformY))` [S1, unofficial].
     - For each column `x` in sprite range:
       - If `transformY >= zbuffer[x]`, skip (occluded by wall) [S1, unofficial].
       - Sample texture, skip if alpha < 128, shade and write to framebuffer [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cast_ray_dda(map, pos_x, pos_y, dir_x, dir_y, plane_x, plane_y, fb_w, fb_h, tex_size=64):
    map_w = len(map[0])
    map_h = len(map)
    zbuffer = [0.0] * fb_w

    for x in range(fb_w):
        camera_x = 2.0 * x / fb_w - 1.0
        ray_dir_x = dir_x + plane_x * camera_x
        ray_dir_y = dir_y + plane_y * camera_x

        map_x = int(pos_x)
        map_y = int(pos_y)

        delta_dist_x = abs(1.0 / ray_dir_x) if ray_dir_x != 0 else 1e30
        delta_dist_y = abs(1.0 / ray_dir_y) if ray_dir_y != 0 else 1e30

        if ray_dir_x < 0:
            step_x = -1
            side_dist_x = (pos_x - map_x) * delta_dist_x
        else:
            step_x = 1
            side_dist_x = (map_x + 1 - pos_x) * delta_dist_x

        if ray_dir_y < 0:
            step_y = -1
            side_dist_y = (pos_y - map_y) * delta_dist_y
        else:
            step_y = 1
            side_dist_y = (map_y + 1 - pos_y) * delta_dist_y

        hit = False
        side = 0
        guard = 0
        while not hit and guard < 128:
            guard += 1
            if side_dist_x < side_dist_y:
                side_dist_x += delta_dist_x
                map_x += step_x
                side = 0
            else:
                side_dist_y += delta_dist_y
                map_y += step_y
                side = 1

            if map_x < 0 or map_x >= map_w or map_y < 0 or map_y >= map_h:
                hit = True
                continue

            tile = map[map_y][map_x]
            if tile == 0:
                continue
            if tile == 5:
                # Sliding door logic from [S1, unofficial]: advance to center line,
                # check openness, continue traversal if gap not reached.
                hit = True
                continue
            hit = True

        if side == 0:
            perp_dist = side_dist_x - delta_dist_x
        else:
            perp_dist = side_dist_y - delta_dist_y

        zbuffer[x] = perp_dist

        line_height = int(fb_h / perp_dist)
        draw_start = max(0, (fb_h - line_height) // 2)
        draw_end = min(fb_h - 1, (fb_h + line_height) // 2)

        if side == 0:
            wall_x = pos_y + perp_dist * ray_dir_y
        else:
            wall_x = pos_x + perp_dist * ray_dir_x
        wall_x -= int(wall_x)

        if (side == 0 and ray_dir_x > 0) or (side == 1 and ray_dir_y < 0):
            wall_x = 1.0 - wall_x

        # Fixed-point vertical texture step [S1, unofficial]
        step = tex_size / line_height
        tex_pos = (draw_start - (fb_h - line_height) / 2) * step
        for y in range(draw_start, draw_end + 1):
            tex_y = int(tex_pos) & (tex_size - 1)
            tex_pos += step
            # Write shaded texel to framebuffer [G]
            # fb[y * fb_w + x] = shade(tex[tex_y * tex_size + int(wall_x * tex_size)])
    return zbuffer
```

## Operating characteristics
Highly efficient on CPU-only hardware; the original engine ran on 286/386-class processors [S1, unofficial]. Optimizations like drawing every second column can double FPS on low-resolution displays with negligible visual loss [S5, unofficial]. Time, memory complexity, strengths, limitations, and usage guidelines are not given in the sources.

## Background
Wolfenstein 3D (id Software, 1992) rendered first-person corridors on 286/386-class hardware without GPUs, z-buffers, or polygons [S1, unofficial]. The technique projects a 2D grid map into a 3D perspective viewport using one ray per screen column [S3, unofficial]. It is a specific application of the Digital Differential Analyzer (DDA) algorithm for grid traversal, popularized by Lodev's tutorial [S1, unofficial][S3, unofficial].

## Sources

[S1] A minimal Wolfenstein 3D raycasting engine, https://4rknova.com/blog/2026/07/06/wolfenstein-3d-raycasting, fetched 2026-10-24 (unofficial)
[S2] mithraskuipers/InteractiveLearning-Raycasting, https://github.com/mithraskuipers/InteractiveLearning-Raycasting, fetched 2026-10-24 (unofficial)
[S3] Developing a 3D Raycasting Engine from Scratch: Vector Math and ..., https://dev.to/ebendttl/developing-a-3d-raycasting-engine-from-scratch-vector-math-and-rendering-3n28, fetched 2026-10-25 (unofficial)
[S4] FIRST RAY CASTING USING DDA, https://github.com/elhiba/cub3d/pull/8, fetched 2026-10-25 (unofficial)
[S5] Как Я Заставил Esp8266 Рисовать 3d, Стрелять И Летать: Полный ..., https://habr.com/ru/articles/1097380/, fetched 2026-10-26 (unofficial)
