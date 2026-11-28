# Floor Rendering via Ray Casting

Also called: DDA Floor Casting, Raycast Floor Projection

## Overview
Floor Rendering via Ray Casting is a technique for rendering an infinite textured floor by casting rays from the camera plane through screen pixels to the world grid, interpolating world coordinates $(x, y)$ based on the distance from the camera to the screen row [S1, unofficial]. It differs from standard texture mapping by calculating world-space coordinates directly per pixel using perspective division, allowing for seamless infinite tiling without UV wrapping artifacts. It builds on [Infinite Floor Projection](13461-infinite-floor-projection.md) by explicitly deriving the world coordinate increments from the view frustum geometry rather than relying on pre-projected screen coordinates.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Screen dimensions ($SCREEN\_WIDTH$, $SCREEN\_HEIGHT$), horizon line ($HORIZON$), player position in world/grid space ($playerX$, $playerY$), player orientation vector ($dirX$, $dirY$), and camera plane vector ($planeX$, $planeY$).
*   **Outputs:** A pixel buffer where each pixel $(x, y)$ below the horizon is colored based on the floor grid cell $(cellX, cellY)$ intersected by the ray.

**Data Structures:**
*   `normalizedLeft`: A 2D vector $(normalizedLeftX, normalizedLeftY)$ representing the normalized direction to the left edge of the view frustum at the near plane.
*   `normalizedRight`: A 2D vector $(normalizedRightX, normalizednormalizedRightY)$ representing the normalized direction to the right edge of the view frustum at the near plane.
*   `floorXStart`, `floorYStart`: Floating-point accumulators for the current world grid coordinates at the left edge of the current screen row.
*   `floorXDirInc`, `floorYDirInc`: Floating-point increments for advancing the world grid coordinates across one screen pixel horizontally.

**Steps:**

1.  **Calculate Frustum Edge Vectors:**
    Compute the unnormalized direction vectors for the left and right edges of the view frustum at the near plane. The near plane is implicitly defined by the camera plane geometry.
    $$
    \begin{aligned}
    nearPlaneX1 &= dirX - planeX \\
    nearPlaneY1 &= dirY - planeY \\
    nearPlaneX2 &= dirX + planeX \\
    nearPlaneY2 &= dirY + planeY
    \end{aligned}
    $$
    Normalize these vectors to unit length:
    $$
    \begin{aligned}
    normalizedLeftX &= \frac{nearPlaneX1}{\sqrt{nearPlaneX1^2 + nearPlaneY1^2}} \\
    normalizedLeftY &= \frac{nearPlaneY1}{\sqrt{nearPlaneX1^2 + nearPlaneY1^2}} \\
    normalizedRightX &= \frac{nearPlaneX2}{\sqrt{nearPlaneX2^2 + nearPlaneY2^2}} \\
    normalizedRightY &= \frac{nearPlaneY2}{\sqrt{nearPlaneX2^2 + nearPlaneY2^2}}
    \end{aligned}
    $$
    [S1, unofficial]

2.  **Iterate Screen Rows:**
    Loop through screen rows $y$ from the bottom of the screen ($SCREEN\_HEIGHT$) up to the horizon ($HORIZON$). The floor is rendered for $y > HORIZON$.
    [S1, unofficial]

3.  **Calculate Distance and Increments:**
    For each row $y$, calculate the perspective distance factor $yDistance$ and the horizontal world coordinate increments per pixel.
    $$
    yDistance = \frac{HORIZON}{y - HORIZON}
    $$
    [S1, unofficial]
    $$
    \begin{aligned}
    floorXDirInc &= yDistance \times \frac{normalizedRightX - normalizedLeftX}{SCREEN\_WIDTH} \\
    floorYDirInc &= yDistance \times \frac{normalizedRightY - normalizedLeftY}{SCREEN\_WIDTH}
    \end{aligned}
    $$
    [S1, unofficial]
    *Note:* $yDistance$ represents the distance from the camera to the floor plane at row $y$. In the source implementation, $HORIZON$ is equated to $PLAYER\_HEIGHT$ (camera height). [S1, unofficial]

4.  **Initialize Row Start Coordinates:**
    Calculate the starting world grid coordinates for the left edge of the current row. This is the player's grid position plus the offset to the left edge of the view frustum scaled by the distance.
    $$
    \begin{aligned}
    floorXStart &= worldToGrid(playerX) + yDistance \times normalizedLeftX \\
    floorYStart &= worldToGrid(playerY) + yDistance \times normalizedLeftY
    \end{aligned}
    $$
    [S1, unofficial]
    *Note:* $worldToGrid$ converts world coordinates to grid indices. If the grid size is 1.0, this may be an identity or scaling function. [S1, unofficial]

5.  **Iterate Screen Columns:**
    For each column $x$ from $0$ to $SCREEN\_WIDTH - 1$:
    a. Determine the grid cell indices:
    $$
    \begin{aligned}
    cellX &= \lfloor |floorXStart| \rfloor \\
    cellY &= \lfloor |floorYStart| \rfloor
    \end{aligned}
    $$
    [S1, unofficial]
    b. Update the start coordinates for the next pixel in the row:
    $$
    \begin{aligned}
    floorXStart &+= floorXDirInc \\
    floorYStart &+= floorYDirInc
    \end{aligned}
    $$
    [S1, unofficial]
    c. Render the pixel using $cellX$ and $cellY$. For example, a checkerboard pattern can be applied:
    $$
    \text{if } ((cellX + cellY) \pmod 2) \neq 0 \text{ then color} = (0, 0, 255, 255) \text{ else color} = (255, 255, 255, 255)
    $$
    [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def render_floor(SCREEN_WIDTH, SCREEN_HEIGHT, HORIZON, playerX, playerY, 
                 dirX, dirY, planeX, planeY, worldToGrid):
    """
    Renders the floor for a single frame.
    Assumes dirX, dirY, planeX, planeY are already normalized or consistent with the camera setup.
    """
    # 1. Calculate normalized frustum edge vectors
    nearPlaneX1 = dirX - planeX
    nearPlaneY1 = dirY - planeY
    nearPlaneX2 = dirX + planeX
    nearPlaneY2 = dirY + planeY
    
    lenLeft = math.sqrt(nearPlaneX1**2 + nearPlaneY1**2)
    lenRight = math.sqrt(nearPlaneX2**2 + nearPlaneY2**2)
    
    # Guard against division by zero if frustum edges are degenerate
    if lenLeft == 0 or lenRight == 0:
        return

    normalizedLeftX = nearPlaneX1 / lenLeft
    normalizedLeftY = nearPlaneY1 / lenLeft
    normalizedRightX = nearPlaneX2 / lenRight
    normalizedRightY = nearPlaneY2 / lenRight

    # Pre-calculate constant differences for increments
    diffX = normalizedRightX - normalizedLeftX
    diffY = normalizedRightY - normalizedLeftY
    
    # Player grid position
    gridPlayerX = worldToGrid(playerX)
    gridPlayerY = worldToGrid(playerY)

    # 2. Iterate screen rows from bottom to horizon
    for y in range(SCREEN_HEIGHT, HORIZON, -1):
        # 3. Calculate distance factor
        # yDistance is proportional to the distance from camera to floor at this row
        yDistance = HORIZON / (y - HORIZON)
        
        # Calculate increments per pixel
        floorXDirInc = yDistance * diffX / SCREEN_WIDTH
        floorYDirInc = yDistance * diffY / SCREEN_WIDTH
        
        # 4. Initialize start coordinates for this row
        floorXStart = gridPlayerX + yDistance * normalizedLeftX
        floorYStart = gridPlayerY + yDistance * normalizedLeftY
        
        # 5. Iterate screen columns
        for x in range(SCREEN_WIDTH):
            # Determine grid cell
            cellX = math.floor(abs(floorXStart))
            cellY = math.floor(abs(floorYStart))
            
            # Update for next pixel
            floorXStart += floorXDirInc
            floorYStart += floorYDirInc
            
            # Render pixel (Example: Checkerboard)
            # In a real implementation, use cellX, cellY to sample texture
            if (cellX + cellY) % 2 != 0:
                color = (0, 0, 255, 255) # Blue
            else:
                color = (255, 255, 255, 255) # White
            
            # Write to buffer at index (y * SCREEN_WIDTH + x)
            # buffer[(y * SCREEN_WIDTH + x) * 4 : (y * SCREEN_WIDTH + x) * 4 + 4] = color

```

**Parameters:**
*   $PLAYER\_HEIGHT$: The vertical position of the camera. In the source implementation, this is equated to $HORIZON$ [S1, unofficial].
*   $HORIZON$: The y-coordinate on the screen corresponding to the camera height (vanishing line). Floor rendering occurs for screen $y$ values from $SCREEN\_HEIGHT$ down to $HORIZON + 1$. [S1, unofficial]
*   $SCREEN\_HEIGHT$: The total height of the canvas in pixels. The bottom pixel is at $y = SCREEN\_HEIGHT$. [S1, unofficial]
*   $SCREEN\_WIDTH$: The total width of the canvas in pixels. Used to calculate ray increment and floor step increments. [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **Wall/Floor Sync:** The floor rendering may appear 'out of sync' with wall rendering, where floor tiles appear to slide or change color near walls as the player moves. This is attributed to a discrepancy between the distance used for wall height calculation (from DDA) and the distance used for floor interpolation ($yDistance$), although these are described as 'two different types of distances'. [S1, unofficial]
*   **Horizon Division:** The formula for $yDistance$ involves division by $(y - HORIZON)$. If $y$ approaches $HORIZON$, the distance approaches infinity, potentially causing floating-point overflow or extreme coordinate values. The loop condition $y > HORIZON$ prevents division by zero but requires handling large coordinates near the horizon.
*   **Grid Coordinate Sign:** The use of `Math.abs` before `Math.floor` implies that the floor grid is symmetric around the origin or that the sign of the world coordinate is ignored for cell indexing. This may lead to incorrect texture wrapping if the player's position is not centered or if the grid is not symmetric. [S1, unofficial]

## Operating characteristics
*   **Time Complexity:** $O(W \times H)$, where $W$ is screen width and $H$ is screen height (specifically $H - HORIZON$). Each pixel requires a constant number of arithmetic operations.
*   **Memory Complexity:** $O(1)$ auxiliary space, excluding the output buffer.
*   **Strengths:** Simple to implement; provides infinite floor tiling without UV mapping; consistent with ray-cast wall rendering.
*   **Limitations:** Potential visual artifacts near walls due to distance calculation mismatch; no perspective-correct texture filtering (nearest-neighbor sampling implied by integer cell indexing); performance is linear with screen resolution.
*   **When to use:** Simple 2.5D raycasting engines (e.g., Wolfenstein 3D style) where performance is critical and high-fidelity texture mapping is not required.
*   **When not to use:** When perspective-correct texture mapping, mipmapping, or complex floor materials are needed.

## Background
This technique is described in the context of implementing a raycaster with both wall and floor rendering. It is presented as a method to solve the problem of rendering an infinite floor in a raycasting engine, where the floor coordinates are interpolated based on the distance from the camera. The source notes that the basic idea uses $PLAYER\_HEIGHT$ (position of camera) and the formula $PLAYER\_HEIGHT / (y - HORIZON)$ to interpolate floor coordinates. [S1, unofficial]

## Sources
[S1] Raycast DDA: Wall rendering and floor rendering out of sync., https://gamedev.net/forums/topic/717763-raycast-dda-wall-rendering-and-floor-rendering-out-of-sync/, fetched 2026-10-26 (unofficial)
