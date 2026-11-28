# Raycasting Texture Coordinate Calculation

Also called: u-coordinate derivation, wallX calculation

## Overview
This method computes the horizontal texture coordinate (commonly `wallX` or the "u" coordinate) for a raycast intersection against an axis-aligned grid wall [S1, unofficial]. It converts the 3D hit point into a normalized $[0, 1)$ texture space value by exploiting the grid's axis alignment: the coordinate depends strictly on the hit face type (vertical or horizontal), using the perpendicular distance and ray direction to determine the position along the wall's local axis [G]. It differs from general 3D texture mapping by avoiding full projection matrices and instead deriving the coordinate directly from DDA traversal state [G].

## How it works
**Inputs:**
- `side`: Integer indicating the hit face type. `0` for vertical face (constant-X grid line), `1` for horizontal face (constant-Y grid line) [S1, unofficial].
- `px`, `py`: The camera position in world coordinates ($x, y$) [G].
- `dirX`, `dirY`: The ray direction vector components [G].
- `perpDist`: The perpendicular distance from the camera to the hit wall plane [S1, unofficial].
- `mapX`, `mapY`: The integer grid coordinates of the cell where the hit occurred [G].
- `texWidth`: The width of the texture image in pixels [S1, unofficial].

**Outputs:**
- `wallX`: The wrapped texture coordinate in $[0, 1)$ [S1, unofficial].
- `texX`: The integer column index in the texture image [S1, unofficial].

**Data Structures:**
No complex data structures are required; the calculation uses scalar values derived from the raycasting traversal [G].

**Steps:**
1. **Determine the varying axis:** Identify which world-space coordinate varies along the hit face [S1, unofficial].
   - If `side == 0` (vertical face), the coordinate varies along Y. Use `py` and `dirY` [S1, unofficial].
   - If `side == 1` (horizontal face), the coordinate varies along X. Use `px` and `dirX` [S1, unofficial].
2. **Compute raw world-space coordinate:** Calculate the position along the wall face using the formula:
   $$
   wallX_{raw} = \begin{cases} 
   py + perpDist \cdot dirY & \text{if } side == 0 \\
   px + perpDist \cdot dirX & \text{if } side == 1 
   \end{cases}
   $$
   [S1, unofficial]
3. **Wrap to $[0, 1)$:** Remove the integer part to get the fractional position within the specific wall segment [S1, unofficial].
   $$
   wallX = wallX_{raw} - \lfloor wallX_{raw} \rfloor
   $$
   [S1, unofficial]
4. **Compute texture column index:** Map the normalized coordinate to the texture's pixel width [S1, unofficial].
   $$
   texX = \lfloor wallX \cdot texWidth \rfloor
   $$
   [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_texture_coordinate(side, px, py, dirX, dirY, perpDist, mapX, mapY, texWidth):
    # Step 1 & 2: Determine axis and compute raw world-space coordinate
    if side == 0:
        side_axis_val = py
        side_dir_val = dirY
    else:
        side_axis_val = px
        side_dir_val = dirX
    
    raw_wall_x = side_axis_val + perpDist * side_dir_val
    
    # Step 3: Wrap to [0, 1)
    wall_x = raw_wall_x - math.floor(raw_wall_x)
    
    # Step 4: Compute texture column index
    tex_x = math.floor(wall_x * texWidth)
    
    return wall_x, tex_x
```

**Parameters:**
- `texWidth`: Must be positive [G].

**Edge Cases and Pitfalls:**
- **Incorrect Side Logic:** If `side` is misinterpreted (e.g., swapping vertical/horizontal logic), textures will appear sheared or scroll in the wrong direction when rays hit the opposite face type [S1, unofficial].
- **Negative Coordinates:** The wrapping step `wallX -= Math.floor(wallX)` correctly handles negative `raw_wall_x` values, ensuring the result is always in $[0, 1)$ [S1, unofficial].
- **Precision:** Floating-point precision errors in `perpDist` or direction vectors may cause slight jitter in `wallX` at grazing angles [G].

## Subtopics

### Sliding Door Raycasting
A variant for handling thin walls recessed to the midline of a cell (tile id 5) [S2, unofficial]. Instead of stopping at the cell boundary, the ray advances to the door's centerline (`distToMid`) [S2, unofficial]. The texture coordinate `wx` is calculated at this midline [S2, unofficial]. If the door is open (`slide > 0`), rays hitting within the opened gap (`wx < slide`) pass through without rendering the door [S2, unofficial].

**Steps:**
1. If tile is 5, calculate `distToMid`:
   $$
   distToMid = \begin{cases} 
   (sideDistX - deltaDistX) + \frac{deltaDistX}{2} & \text{if } side == 0 \\
   (sideDistY - deltaDistY) + \frac{deltaDistY}{2} & \text{if } side == 1 
   \end{cases}
   $$
   [S2, unofficial]
2. Calculate `other` (distance to the other side of the cell):
   $$
   other = \begin{cases} 
   sideDistY & \text{if } side == 0 \\
   sideDistX & \text{if } side == 1 
   \end{cases}
   $$
   If `distToMid >= other`, the ray exits the cell before the midline; continue marching [S2, unofficial].
3. Calculate texture coordinate `wx` at midline:
   $$
   wx = \begin{cases} 
   player.y + distToMid \cdot rayDirY - mapY & \text{if } side == 0 \\
   player.x + distToMid \cdot rayDirX - mapX & \text{if } side == 1 
   \end{cases}
   $$
   If `wx < 0` or `wx >= 1`, continue [S2, unofficial].
4. Get door openness `slide`. If `wx < slide`, the ray passes through the open gap; continue [S2, unofficial].
5. Return hit with `wallX: wx` and `doorShift: -slide` [S2, unofficial].

## Operating characteristics
- Time complexity: Not given in the sources. [G]
- Memory complexity: Not given in the sources. [G]
- Strengths: Not given in the sources. [G]
- Limitations: Not given in the sources. [G]
- When to use: Not given in the sources. [G]
- When not to use: Not given in the sources. [G]

## Background
This method is fundamental to early raycasting engines, particularly those used in first-person shooters like Wolfenstein 3D [S2, unofficial]. It leverages the simplicity of grid-based collision detection to derive texture coordinates without full 3D projection matrices [G]. The technique is often associated with the implementation details of DDA-based ray traversal, where the side of the hit face is known immediately upon intersection [S1, S2, unofficial].

## Sources
[S1] mithraskuipers/InteractiveLearning-Raycasting, https://github.com/mithraskuipers/InteractiveLearning-Raycasting, fetched 2026-10-24 (unofficial)
[S2] A minimal Wolfenstein 3D raycasting engine, https://4rknova.com/blog/2026/07/06/wolfenstein-3d-raycasting, fetched 2026-10-24 (unofficial)
