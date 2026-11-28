# Amanatides–Woo grid traversal

Also called: Line walk traversal on extended grid lines, traverseGridSegmentFloat

## Overview
This method traverses the grid cells intersected by a line segment defined by two points, calling a visitor function for each cell in traversal order [S2, unofficial]. It is a 2D floating-point implementation designed for uniform grid traversal [S2, unofficial].

## How it works
**Inputs:**
- `start`: A 2D point $(start.x, start.y)$ representing the beginning of the line segment [S2, unofficial].
- `end`: A 2D point $(end.x, end.y)$ representing the end of the line segment [S2, unofficial].
- `visitor`: A callable object with a method `visit(int, int)` that is invoked for each traversed cell $(x, y)$ [S2, unofficial].

**Outputs:**
- The `visitor.visit` function is called for every grid cell intersected by the segment, in the order they are encountered from `start` to `end` [S2, unofficial].

**Data Structures:**
- The algorithm uses scalar variables for coordinates, deltas, and t-max values [S2, unofficial].

**Parameters:**
- `FL_FLT_MAX`: A large floating-point constant representing infinity, used when a direction component is zero to prevent division by zero [S2, unofficial].

**Steps:**
1. **Compute Initial Cell Coordinates:**
   Determine the grid cell indices containing the start and end points:
   $$x_0 = \lfloor start.x \rfloor, \quad y_0 = \lfloor start.y \rfloor$$
   $$x_1 = \lfloor end.x \rfloor, \quad y_1 = \lfloor end.y \rfloor$$
   [S2, unofficial]

2. **Compute Step Directions:**
   Determine the direction of traversal along each axis:
   $$stepX = \begin{cases} 1 & \text{if } x_1 > x_0 \\ -1 & \text{if } x_1 < x_0 \\ 0 & \text{otherwise} \end{cases}$$
   $$stepY = \begin{cases} 1 & \text{if } y_1 > y_0 \\ -1 & \text{if } y_1 < y_0 \\ 0 & \text{otherwise} \end{cases}$$
   [S2, unofficial]

3. **Compute Direction Deltas:**
   Calculate the total displacement of the segment:
   $$dx = end.x - start.x$$
   $$dy = end.y - start.y$$
   [S2, unofficial]

4. **Compute T-Deltas (Inverse Step Size):**
   Calculate the change in parameter $t$ required to cross one grid cell in each direction. If the direction component is zero, the t-delta is set to infinity:
   $$tDeltaX = \begin{cases} \left| \frac{1}{dx} \right| & \text{if } dx \neq 0 \\ FL\_FLT\_MAX & \text{otherwise} \end{cases}$$
   $$tDeltaY = \begin{cases} \left| \frac{1}{dy} \right| & \text{if } dy \neq 0 \\ FL\_FLT\_MAX & \text{otherwise} \end{cases}$$
   [S2, unofficial]

5. **Compute Next Intersection Coordinates:**
   Determine the coordinate of the next grid boundary to cross in each direction:
   $$nextX = \begin{cases} \lfloor start.x \rfloor + 1 & \text{if } stepX > 0 \\ \lfloor start.x \rfloor & \text{otherwise} \end{cases}$$
   $$nextY = \begin{cases} \lfloor start.y \rfloor + 1 & \text{if } stepY > 0 \\ \lfloor start.y \rfloor & \text{otherwise} \end{cases}$$
   [S2, unofficial]

6. **Compute Initial T-Max Values:**
   Calculate the parameter $t$ at which the segment first intersects the next grid boundary in each direction. If the direction component is zero, the t-max is infinity:
   $$tMaxX = \begin{cases} \left| \frac{nextX - start.x}{dx} \right| & \text{if } dx \neq 0 \\ FL\_FLT\_MAX & \text{otherwise} \end{cases}$$
   $$tMaxY = \begin{cases} \left| \frac{nextY - start.y}{dy} \right| & \text{if } dy \neq 0 \\ FL\_FLT\_MAX & \text{otherwise} \end{cases}$$
   [S2, unofficial]

7. **Initialize Loop State:**
   Set the maximum valid $t$ to 1.0 (the end of the segment). Initialize current cell coordinates to the start cell:
   $$maxT = 1.0$$
   $$currentX = x_0$$
   $$currentY = y_0$$
   [S2, unofficial]

8. **Traversal Loop:**
   Repeat the following steps until the end of the segment is reached:
   a. Call `visitor.visit(currentX, currentY)` [S2, unofficial].
   b. Find the minimum t-max: $t = \min(tMaxX, tMaxY)$ [S2, unofficial].
   c. If $t > maxT$, break the loop [S2, unofficial].
   d. If $tMaxX < tMaxY$:
      - Increment $tMaxX$ by $tDeltaX$: $tMaxX \leftarrow tMaxX + tDeltaX$ [S2, unofficial].
      - Increment $currentX$ by $stepX$: $currentX \leftarrow currentX + stepX$ [S2, unofficial].
   e. Else (if $tMaxY \le tMaxX$):
      - Increment $tMaxY$ by $tDeltaY$: $tMaxY \leftarrow tMaxY + tDeltaY$ [S2, unofficial].
      - Increment $currentY$ by $stepY$: $currentY \leftarrow currentY + stepY$ [S2, unofficial].
   [S2, unofficial]

9. **Final Cell Check:**
   After the loop, if the current cell is not the end cell $(x_1, y_1)$, explicitly visit the end cell to ensure it is included:
   If $currentX \neq x_1$ or $currentY \neq y_1$, call `visitor.visit(x1, y1)` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def traverse_grid_segment_float(start_x, start_y, end_x, end_y, visitor):
    INF = float('inf')
    
    # 1. Compute Initial Cell Coordinates
    x0 = math.floor(start_x)
    y0 = math.floor(start_y)
    x1 = math.floor(end_x)
    y1 = math.floor(end_y)

    # 2. Compute Step Directions
    step_x = 1 if x1 > x0 else (-1 if x1 < x0 else 0)
    step_y = 1 if y1 > y0 else (-1 if y1 < y0 else 0)

    # 3. Compute Direction Deltas
    dx = end_x - start_x
    dy = end_y - start_y

    # 4. Compute T-Deltas
    t_delta_x = abs(1.0 / dx) if dx != 0.0 else INF
    t_delta_y = abs(1.0 / dy) if dy != 0.0 else INF

    # 5. Compute Next Intersection Coordinates
    next_x = math.floor(start_x) + 1 if step_x > 0 else math.floor(start_x)
    next_y = math.floor(start_y) + 1 if step_y > 0 else math.floor(start_y)

    # 6. Compute Initial T-Max Values
    t_max_x = abs((next_x - start_x) / dx) if dx != 0.0 else INF
    t_max_y = abs((next_y - start_y) / dy) if dy != 0.0 else INF

    # 7. Initialize Loop State
    max_t = 1.0
    current_x = x0
    current_y = y0

    # 8. Traversal Loop
    while True:
        visitor.visit(current_x, current_y)
        
        t = min(t_max_x, t_max_y)
        
        if t > max_t:
            break
        
        if t_max_x < t_max_y:
            t_max_x += t_delta_x
            current_x += step_x
        else:
            t_max_y += t_delta_y
            current_y += step_y

    # 9. Final Cell Check
    if current_x != x1 or current_y != y1:
        visitor.visit(x1, y1)
```

**Edge Cases and Pitfalls:**
- **Zero Direction Component:** If $dx=0$ or $dy=0$, the corresponding t-delta and t-max are set to infinity to prevent division by zero and ensure the loop only steps in the non-zero direction [S2, unofficial].
- **End Cell Inclusion:** The loop may terminate before visiting the final cell $(x_1, y_1)$ if the segment ends exactly at a boundary or if the t-max logic skips the last step. The explicit check at the end ensures the end cell is always visited [S2, unofficial].

## Subtopics

### [Amanatides–Woo grid traversal (fixed-point Q24.8 variant)](5006-amanatides-woo-grid-traversal-fixed-point-q24-8-variant.md)
Implements the algorithm using 32-bit signed integers with 8 fractional bits (Q24.8) to improve performance and avoid floating-point precision issues. Choose it when targeting fixed-point hardware or uniform grids where integer arithmetic is preferred.

### [Compiled Per-Ray Triangle Grid Walk](5198-compiled-per-ray-triangle-grid-walk.md)
Compiles the per-ray traversal logic into a tight loop using Numba, eliminating global ray state array overhead. Choose it when traversing sparse triangle meshes per-ray rather than uniform grids.

### [2.5D Grid Traversal with Z-interpolation](13672-2-5d-grid-traversal-with-z-interpolation.md)
Extends the 2D DDA to calculate Z-coordinates for culling in 2.5D height-field representations. Choose it when traversing terrain or height-field data where Z-overlap must be checked per cell.

### [Amanatides-Woo Grid Picking](15338-amanatides-woo-grid-picking.md)
Applies the traversal logic to a logical height map for exact, deterministic picking in 2D grid-based environments. Choose it when replacing expensive raycasting against detailed terrain meshes with fast, exact grid-based picking.

## Operating characteristics
- **Time Complexity:** The traversal time is proportional to the number of cells intersected by the segment [S2, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space, as only scalar variables are used [S2, unofficial].

## Background
The algorithm is implemented in the FastLED library for grid traversal tasks [S2, unofficial].

## Sources
[S1] Fast axis aligned cell traversal algorithm, https://stackoverflow.com/questions/13700625, fetched 2026-10-25 (unofficial)
[S2] FastLED: fl/traverse_grid.h Source File, https://fastled.io/docs/d9/d42/traverse__grid_8h_source.html, fetched 2026-10-30 (unofficial)
