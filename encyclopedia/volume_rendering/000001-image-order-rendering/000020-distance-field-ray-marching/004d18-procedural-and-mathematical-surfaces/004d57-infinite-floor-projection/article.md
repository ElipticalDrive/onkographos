# Infinite Floor Projection

Also called: testFloor

## Overview
Infinite Floor Projection is a technique for rendering an infinite textured floor by projecting screen coordinates into world space using perspective division, then applying player rotation and a checkerboard pattern [S1, unofficial]. It calculates world-space coordinates directly per pixel using perspective division, allowing for seamless infinite tiling without UV wrapping artifacts [S1, unofficial].

## How it works
**Inputs:**
*   Screen dimensions: `SCR_WIDTH`, `SCR_HEIGHT` [S1, unofficial].
*   Player state: `player.lookAngle` (rotation), `player.x`, `player.y`, `player.z` (position) [S1, unofficial].
*   Constants: `MOVE_VERTICAL_SCALE = 16.0`, `LOOK_ANGLE_MULTIPLIER = 6.2` (approximates $2\pi$) [S1, unofficial].

**Outputs:**
*   Pixel colors for a checkerboard pattern [S1, unofficial].

**Data Structures:**
*   Scalar variables and simple iteration loops [G].

**Steps:**
1.  **Calculate Look Angle Offset:** Compute `lookAngle = -player.lookAngle * 6.2`. Clamp the result to the range `[0, SCR_HEIGHT]` [S1, unofficial].
2.  **Calculate Vertical Scale:** Compute `moveVertical = player.z / 16.0`. If `moveVertical` equals zero, set it to `0.0001` to prevent division by zero [S1, unofficial].
3.  **Iterate Screen Rows:** Loop `y` from `0` to `SCR_HEIGHT` [G].
4.  **Iterate Screen Columns:** Loop `x` from `-SCR_WIDTH // 2` to `SCR_WIDTH // 2` [S1, unofficial].
5.  **Compute Depth Proxy:** Calculate `z = y + lookAngle`. If `z <= 0`, set `z = 0.0001` [S1, unofficial].
6.  **Project to World Space:** Calculate intermediate coordinates:
    $$ fx = \frac{x}{z} \times moveVertical $$
    $$ fy = \frac{200}{z} \times moveVertical $$
    [S1, unofficial].
7.  **Apply Player Rotation:** Rotate the coordinates using the player's look angle:
    $$ rx = fx \times \sin(\text{lookAngle}) - fy \times \cos(\text{lookAngle}) + \frac{\text{player.y}}{60.0} $$
    $$ ry = fx \times \cos(\text{lookAngle}) + fy \times \sin(\text{lookAngle}) - \frac{\text{player.x}}{60.0} $$
    [S1, unofficial].
8.  **Mirror Coordinates:** Handle negative values for seamless tiling:
    *   If `rx < 0`, set `rx = -rx + 1`.
    *   If `ry < 0`, set `ry = -ry + 1` [S1, unofficial].
9.  **Cull Invalid Tiles:** Skip rendering if `rx <= 0` or `ry <= 0` or `rx >= 5` or `ry >= 5` [S1, unofficial].
10. **Determine Pattern:** Draw red if `rx % 2 == ry % 2`, otherwise draw blue [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_infinite_floor(player, scr_width, scr_height):
    LOOK_ANGLE_MULTIPLIER = 6.2
    MOVE_VERTICAL_SCALE = 16.0
    MIN_Z = 0.0001
    MIN_MOVE_VERTICAL = 0.0001
    FLOOR_OFFSET = 60.0
    TILE_LIMIT = 5
    RED = (255, 0, 0)
    BLUE = (0, 0, 255)

    # 1. Calculate Look Angle Offset
    look_angle = -player.lookAngle * LOOK_ANGLE_MULTIPLIER
    look_angle = max(0.0, min(look_angle, scr_height))

    # 2. Calculate Vertical Scale
    move_vertical = player.z / MOVE_VERTICAL_SCALE
    if move_vertical == 0:
        move_vertical = MIN_MOVE_VERTICAL

    # 3 & 4. Iterate Screen Pixels
    for y in range(0, scr_height):
        for x in range(-scr_width // 2, scr_width // 2):
            # 5. Compute Depth Proxy
            z = y + look_angle
            if z <= 0:
                z = MIN_Z

            # 6. Project to World Space
            fx = (x / z) * move_vertical
            fy = (200 / z) * move_vertical

            # 7. Apply Player Rotation
            rx = fx * sin(look_angle) - fy * cos(look_angle) + (player.y / FLOOR_OFFSET)
            ry = fx * cos(look_angle) + fy * sin(look_angle) - (player.x / FLOOR_OFFSET)

            # 8. Mirror Coordinates
            if rx < 0:
                rx = -rx + 1
            if ry < 0:
                ry = -ry + 1

            # 9. Cull Invalid Tiles
            if rx <= 0 or ry <= 0 or rx >= TILE_LIMIT or ry >= TILE_LIMIT:
                continue

            # 10. Determine Pattern
            if (rx % 2) == (ry % 2):
                # Set pixel color at screen coordinates (x + scr_width // 2, y)
                pass
            else:
                # Set pixel color at screen coordinates (x + scr_width // 2, y)
                pass
```

**Parameters:**
*   `moveVertical` scaling factor: 16.0 [S1, unofficial].
*   Look angle multiplier: 6.2 (approximates $2\pi$) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** `z` and `moveVertical` are clamped to `0.0001` to prevent division by zero errors [S1, unofficial].
*   **Coordinate Wrapping:** The mirroring step (`rx = -rx + 1`) handles negative world coordinates to create a seamless tiling effect without explicit modulo operations on negative numbers [S1, unofficial].
*   **Tiling Limit:** The culling step (`rx >= 5`, `ry >= 5`) restricts rendering to a $5 \times 5$ tile area [S1, unofficial].

## Subtopics

### [Floor Rendering via Ray Casting](17901-floor-rendering-via-ray-casting.md)
Floor Rendering via Ray Casting is a technique for rendering an infinite textured floor by casting rays from the camera plane through screen pixels to the world grid, interpolating world coordinates $(x, y)$ based on the distance from the camera to the screen row. It differs from standard texture mapping by calculating world-space coordinates directly per pixel using perspective division, allowing for seamless infinite tiling without UV wrapping artifacts. It builds on [Infinite Floor Projection](13461-infinite-floor-projection.md) by explicitly deriving the world coordinate increments from the view frustum geometry rather than relying on pre-projected screen coordinates.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of pixels rendered, as each pixel is processed in a nested loop [G].
*   **Memory Complexity:** $O(1)$, as only scalar variables are required [G].
*   **Strengths:** Direct world-space coordinate calculation and seamless infinite tiling without UV mapping [S1, unofficial].
*   **Limitations:** Fixed $5 \times 5$ tile culling limit and approximate perspective projection using screen coordinates rather than full 3D ray casting [S1, unofficial].

## Background
Not given in the sources.

## Sources
[S1] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
