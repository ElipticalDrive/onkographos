# Fixed-point Amanatides–Woo Grid Traversal
Also called: Amanatides–Woo grid traversal (fixed-point Q8.8 variant), Q8.8 Amanatides-Woo, traverseGridSegment16

## Overview
This method implements the Amanatides–Woo voxel traversal algorithm using 16-bit signed fixed-point arithmetic (Q8.8 format) to avoid floating-point operations entirely [S1, unofficial]. It solves the problem of efficiently determining which grid cells a line segment intersects in 2D space, serving as the core traversal loop for image-order volume rendering or ray casting [S1, unofficial]. It sets itself apart from its parent [Integer-based Amanatides and Woo Voxel Traversal](4246-integer-based-amanatides-and-woo-voxel-traversal.md) by using fractional bits to represent sub-cell distances, allowing for exact intersection calculations without the accumulation of rounding errors inherent in pure integer DDA, while maintaining the speed benefits of integer arithmetic [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `start` (vec2f), `end` (vec2f) defining the segment endpoints in floating-point grid coordinates.
- Output: A sequence of integer cell coordinates `(x, y)` passed to a `GridVisitor` callback via `visitor.visit(x, y)`.

**Data Structures:**
- `i16`: 16-bit signed integer.
- `u16`: 16-bit unsigned integer.
- `i32`: 32-bit signed integer (used for intermediate arithmetic to prevent overflow).
- Constants: `FP_SHIFT = 8`, `FP_ONE = 256` (representing $1.0$ in Q8.8).

**Parameters:**
- `FP_SHIFT`: The number of fractional bits. Value: 8 [S1, unofficial].
- `FP_ONE`: The fixed-point representation of 1.0. Value: $1 \ll 8 = 256$ [S1, unofficial].

**Steps:**
1.  **Convert to Fixed-Point:** Multiply floating-point coordinates by `FP_ONE` and cast to `i16`.
    $$
    \begin{aligned}
    startX_{fp} &= \text{static\_cast}<i16>(start.x \times FP\_ONE) \\
    startY_{fp} &= \text{static\_cast}<i16>(start.y \times FP\_ONE) \\
    endX_{fp} &= \text{static\_cast}<i16>(end.x \times FP\_ONE) \\
    endY_{fp} &= \text{static\_cast}<i16>(end.y \times FP\_ONE)
    \end{aligned}
    $$
2.  **Determine Initial and Final Cells:** Shift right by `FP_SHIFT` to get integer cell indices.
    $$
    \begin{aligned}
    x_0 &= startX_{fp} \gg FP\_SHIFT, \quad y_0 = startY_{fp} \gg FP\_SHIFT \\
    x_1 &= endX_{fp} \gg FP\_SHIFT, \quad y_1 = endY_{fp} \gg FP\_SHIFT
    \end{aligned}
    $$
3.  **Compute Step Directions:** Determine the direction of traversal for each axis.
    $$
    \begin{aligned}
    stepX &= \begin{cases} 1 & \text{if } x_1 > x_0 \\ -1 & \text{if } x_1 < x_0 \\ 0 & \text{otherwise} \end{cases} \\
    stepY &= \begin{cases} 1 & \text{if } y_1 > y_0 \\ -1 & \text{if } y_1 < y_0 \\ 0 & \text{otherwise} \end{cases}
    \end{aligned}
    $$
4.  **Compute Fixed-Point Deltas:** Calculate the total displacement in fixed-point.
    $$
    \begin{aligned}
    \Delta X_{fp} &= endX_{fp} - startX_{fp} \\
    \Delta Y_{fp} &= endY_{fp} - startY_{fp}
    \end{aligned}
    $$
5.  **Compute t-Deltas (Inverse Step Size):** Calculate the fixed-point distance along the ray to cross one cell boundary. To avoid overflow during division, the numerator is scaled by `FP_ONE` again (effectively dividing by $\Delta X_{fp}$ in Q16.16 space, then shifting back).
    $$
    \begin{aligned}
    absDeltaX_{fp} &= \begin{cases} \text{static\_cast}<u16>(| (i32(FP\_ONE) \ll FP\_SHIFT) / \Delta X_{fp} |) & \text{if } \Delta X_{fp} \neq 0 \\ \text{max}(u16) & \text{otherwise} \end{cases} \\
    absDeltaY_{fp} &= \begin{cases} \text{static\_cast}<u16>(| (i32(FP\_ONE) \ll FP\_SHIFT) / \Delta Y_{fp} |) & \text{if } \Delta Y_{fp} \neq 0 \\ \text{max}(u16) & \text{otherwise} \end{cases}
    \end{aligned}
    $$
6.  **Compute Next Intersection Coordinates:** Determine the fixed-point coordinate of the next grid boundary.
    $$
    \begin{aligned}
    nextX_{fp} &= \begin{cases} (x_0 + 1) \ll FP\_SHIFT & \text{if } stepX > 0 \\ x_0 \ll FP\_SHIFT & \text{otherwise} \end{cases} \\
    nextY_{fp} &= \begin{cases} (y_0 + 1) \ll FP\_SHIFT & \text{if } stepY > 0 \\ y_0 \ll FP\_SHIFT & \text{otherwise} \end{cases}
    \end{aligned}
    $$
7.  **Compute Initial t-Max:** Calculate the fixed-point distance from the start point to the first boundary intersection.
    $$
    \begin{aligned}
    tMaxX_{fp} &= \begin{cases} \text{static\_cast}<u16>(| i32(nextX_{fp} - startX_{fp}) | \times absDeltaX_{fp} \gg FP\_SHIFT) & \text{if } \Delta X_{fp} \neq 0 \\ \text{max}(u16) & \text{otherwise} \end{cases} \\
    tMaxY_{fp} &= \begin{cases} \text{static\_cast}<u16>(| i32(nextY_{fp} - startY_{fp}) | \times absDeltaY_{fp} \gg FP\_SHIFT) & \text{if } \Delta Y_{fp} \neq 0 \\ \text{max}(u16) & \text{otherwise} \end{cases}
    \end{aligned}
    $$
8.  **Initialize Loop State:** Set `maxT_fp = FP_ONE` (representing $t=1.0$) and `currentX = x0`, `currentY = y0` [S1, unofficial].
9.  **Traverse:** Iterate until the ray segment is fully traversed or the final cell is reached.
    - Visit current cell.
    - Select the smaller `tMax` (`t_fp`).
    - If `t_fp > maxT_fp`, break loop.
    - Increment the corresponding `tMax` by its delta and advance the cell coordinate.
10. **Final Cell Check:** Explicitly visit `(x1, y1)` if the loop terminated before reaching it [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverseGridSegment16(start, end, visitor):
    FP_SHIFT = 8
    FP_ONE = 1 << FP_SHIFT  # 256
    
    # 1. Convert to fixed-point Q8.8
    startX_fp = int(start.x * FP_ONE)
    startY_fp = int(start.y * FP_ONE)
    endX_fp = int(end.x * FP_ONE)
    endY_fp = int(end.y * FP_ONE)
    
    # 2. Initial and final cells
    x0 = startX_fp >> FP_SHIFT
    y0 = startY_fp >> FP_SHIFT
    x1 = endX_fp >> FP_SHIFT
    y1 = endY_fp >> FP_SHIFT
    
    # 3. Step directions
    stepX = 1 if x1 > x0 else (-1 if x1 < x0 else 0)
    stepY = 1 if y1 > y0 else (-1 if y1 < y0 else 0)
    
    # 4. Fixed-point deltas
    deltaX_fp = endX_fp - startX_fp
    deltaY_fp = endY_fp - startY_fp
    
    # 5. t-deltas (inverse step size)
    # Use i32 for numerator to prevent overflow: (FP_ONE << FP_SHIFT) is 256 << 8 = 65536
    if deltaX_fp != 0:
        absDeltaX_fp = abs((65536 // deltaX_fp))
    else:
        absDeltaX_fp = 65535  # max u16
        
    if deltaY_fp != 0:
        absDeltaY_fp = abs((65536 // deltaY_fp))
    else:
        absDeltaY_fp = 65535  # max u16
        
    # 6. Next intersection coordinates
    if stepX > 0:
        nextX_fp = (x0 + 1) << FP_SHIFT
    else:
        nextX_fp = x0 << FP_SHIFT
        
    if stepY > 0:
        nextY_fp = (y0 + 1) << FP_SHIFT
    else:
        nextY_fp = y0 << FP_SHIFT
        
    # 7. Initial t-max
    if deltaX_fp != 0:
        # abs(nextX_fp - startX_fp) * absDeltaX_fp >> FP_SHIFT
        distX = abs(nextX_fp - startX_fp)
        tMaxX_fp = (distX * absDeltaX_fp) >> FP_SHIFT
    else:
        tMaxX_fp = 65535  # max u16
        
    if deltaY_fp != 0:
        distY = abs(nextY_fp - startY_fp)
        tMaxY_fp = (distY * absDeltaY_fp) >> FP_SHIFT
    else:
        tMaxY_fp = 65535  # max u16
        
    maxT_fp = FP_ONE  # 256
    currentX = x0
    currentY = y0
    
    # 9. Traverse loop
    while True:
        visitor.visit(currentX, currentY)
        
        # Select smaller tMax
        t_fp = tMaxX_fp if tMaxX_fp < tMaxY_fp else tMaxY_fp
        
        if t_fp > maxT_fp:
            break
            
        if tMaxX_fp < tMaxY_fp:
            tMaxX_fp += absDeltaX_fp
            currentX += stepX
        else:
            tMaxY_fp += absDeltaY_fp
            currentY += stepY
            
    # 10. Ensure end cell is visited
    if currentX != x1 or currentY != y1:
        visitor.visit(x1, y1)
```

**Edge Cases and Pitfalls:**
- **Zero Delta:** If `deltaX_fp` or `deltaY_fp` is zero, the corresponding `absDelta` and `tMax` are set to `max(u16)` to effectively disable that axis from triggering a step, preventing division by zero [S1, unofficial].
- **End Cell Inclusion:** The loop condition `t_fp > maxT_fp` may terminate before the final cell `(x1, y1)` is visited due to fixed-point precision or boundary alignment. An explicit check and visit for `(x1, y1)` is required [S1, unofficial].
- **Macro Expansion:** On platforms like Arduino, `max` is a macro. Calls to `std::max` or similar must be wrapped in parentheses `(fl::numeric_limits<u16>::max)()` to prevent syntax errors [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of cells intersected by the segment. The loop runs once per cell.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Performance:** Significantly faster than floating-point variants on hardware lacking efficient FPU support, as it uses only integer shifts, adds, and multiplies [S1, unofficial].
- **Precision:** Q8.8 provides 8 bits of fractional precision. This is sufficient for most grid traversal tasks but may introduce slight inaccuracies in `tMax` accumulation over very long segments compared to 32-bit fixed-point or double precision [S1, unofficial].
- **Limitations:** Limited to 2D grids in this specific implementation (`traverseGridSegment16`). 3D traversal requires a separate implementation or extension. The 16-bit integer range limits the maximum coordinate values and segment lengths that can be represented without overflow [S1, unofficial].

## Background
This method is part of the FastLED library's geometry utilities, specifically defined in `fl/traverse_grid.h` [S1, unofficial]. It is a specialized fixed-point adaptation of the classic Amanatides and Woo voxel traversal algorithm, originally published in 1987 for ray tracing [S1, unofficial]. The Q8.8 variant is designed for embedded or resource-constrained environments where floating-point operations are expensive or unavailable, leveraging the speed of integer arithmetic while maintaining the exactness of the DDA approach [S1, unofficial].

## Sources
[S1] FastLED: fl/traverse_grid.h Source File, https://fastled.io/docs/d9/d42/traverse__grid_8h_source.html, fetched 2026-10-30 (unofficial)
