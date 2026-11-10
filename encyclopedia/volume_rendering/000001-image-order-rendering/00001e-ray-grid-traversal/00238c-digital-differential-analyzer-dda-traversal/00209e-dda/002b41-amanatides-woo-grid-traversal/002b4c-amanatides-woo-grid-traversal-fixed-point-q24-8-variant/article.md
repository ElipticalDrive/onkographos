# Amanatides–Woo grid traversal (fixed-point Q24.8 variant)
Also called: traverseGridSegment32

## Overview
This method implements the Amanatides–Woo grid traversal algorithm using 32-bit signed integers with 8 fractional bits (Q24.8 fixed-point arithmetic) to determine the sequence of grid cells intersected by a line segment [S1, unofficial]. It builds on [Amanatides–Woo grid traversal](5003-amanatides-woo-grid-traversal.md) by replacing floating-point operations with integer arithmetic to improve performance and avoid floating-point precision issues on specific hardware, while maintaining the same logical structure of stepping through cells based on t-max values [G]. Unlike [Compiled Per-Ray Triangle Grid Walk](5198-compiled-per-ray-triangle-grid-walk.md), which optimizes for sparse triangle meshes, this variant targets uniform grid traversal on fixed-point hardware [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `start` (`vec2f`), `end` (`vec2f`) defining the segment endpoints in world/grid coordinates [S1, unofficial].
- Input: `visitor` (`GridVisitor`), an object with a `visit(x, y)` method called for each intersected cell [S1, unofficial].
- Output: Side effects via `visitor.visit()` calls for each cell $(x, y)$ in the traversal path [S1, unofficial].

**Data Structures:**
- `i32`: 32-bit signed integer [S1, unofficial].
- `u32`: 32-bit unsigned integer [S1, unofficial].
- `FP_SHIFT`: Constant integer 8 [S1, unofficial].
- `FP_ONE`: Constant integer $2^8 = 256$ [S1, unofficial].

**Parameters:**
- `FP_SHIFT = 8` [S1, unofficial]
- `FP_ONE = 256` [S1, unofficial]

**Steps:**

1.  **Convert to Fixed-Point:**
    Convert the floating-point start and end coordinates to Q24.8 fixed-point representation:
    $$
    \begin{aligned}
    \text{startX\_fp} &= \text{start}.x \times \text{FP\_ONE} \\
    \text{startY\_fp} &= \text{start}.y \times \text{FP\_ONE} \\
    \text{endX\_fp} &= \text{end}.x \times \text{FP\_ONE} \\
    \text{endY\_fp} &= \text{end}.y \times \text{FP\_ONE}
    \end{aligned}
    $$
    Cast results to `i32` [S1, unofficial].

2.  **Compute Initial Cell Coordinates:**
    Determine the starting and ending grid cell indices by right-shifting the fixed-point coordinates:
    $$
    \begin{aligned}
    x_0 &= \text{startX\_fp} \gg \text{FP\_SHIFT} \\
    y_0 &= \text{startY\_fp} \gg \text{FP\_SHIFT} \\
    x_1 &= \text{endX\_fp} \gg \text{FP\_SHIFT} \\
    y_1 &= \text{endY\_fp} \gg \text{FP\_SHIFT}
    \end{aligned}
    $$
    [S1, unofficial]

3.  **Compute Step Directions:**
    Determine the direction of traversal for each axis:
    $$
    \begin{aligned}
    \text{stepX} &= \begin{cases} 1 & \text{if } x_1 > x_0 \\ -1 & \text{if } x_1 < x_0 \\ 0 & \text{otherwise} \end{cases} \\
    \text{stepY} &= \begin{cases} 1 & \text{if } y_1 > y_0 \\ -1 & \text{if } y_1 < y_0 \\ 0 & \text{otherwise} \end{cases}
    \end{aligned}
    $$
    [S1, unofficial]

4.  **Compute Fixed-Point Deltas:**
    Calculate the displacement in fixed-point units:
    $$
    \begin{aligned}
    \Delta X_{\text{fp}} &= \text{endX\_fp} - \text{startX\_fp} \\
    \Delta Y_{\text{fp}} &= \text{endY\_fp} - \text{startY\_fp}
    \end{aligned}
    $$
    [S1, unofficial]

5.  **Compute Fixed-Point t-Deltas:**
    Calculate the inverse step size in fixed-point units [S1, unofficial]. To prevent overflow during division, the numerator is scaled by $2^{\text{FP\_SHIFT}}$ using 64-bit arithmetic before division by the signed 32-bit delta [S1, unofficial]. The result is cast to `u32`. If the delta is zero, the t-delta is set to the maximum unsigned 32-bit integer [S1, unofficial].
    $$
    \begin{aligned}
    \text{absDeltaX\_fp} &= \begin{cases} \left\lfloor \frac{|\text{FP\_ONE} \ll \text{FP\_SHIFT}|}{|\Delta X_{\text{fp}}|} \right\rfloor & \text{if } \Delta X_{\text{fp}} \neq 0 \\ \text{max}(u32) & \text{otherwise} \end{cases} \\
    \text{absDeltaY\_fp} &= \begin{cases} \left\lfloor \frac{|\text{FP\_ONE} \ll \text{FP\_SHIFT}|}{|\Delta Y_{\text{fp}}|} \right\rfloor & \text{if } \Delta Y_{\text{fp}} \neq 0 \\ \text{max}(u32) & \text{otherwise} \end{cases}
    \end{aligned}
    $$
    Note: The division uses `fl::i64(FP_ONE) << FP_SHIFT` to ensure 64-bit precision for the numerator [S1, unofficial].

6.  **Compute Next Intersection Coordinates:**
    Determine the fixed-point coordinate of the next grid boundary crossing for each axis:
    $$
    \begin{aligned}
    \text{nextX\_fp} &= \begin{cases} (x_0 + 1) \ll \text{FP\_SHIFT} & \text{if } \text{stepX} > 0 \\ x_0 \ll \text{FP\_SHIFT} & \text{otherwise} \end{cases} \\
    \text{nextY\_fp} &= \begin{cases} (y_0 + 1) \ll \text{FP\_SHIFT} & \text{if } \text{stepY} > 0 \\ y_0 \ll \text{FP\_SHIFT} & \text{otherwise} \end{cases}
    \end{aligned}
    $$
    [S1, unofficial]

7.  **Compute Initial t-Max Values:**
    Calculate the fixed-point distance to the first boundary crossing [S1, unofficial]. This is computed as the absolute distance to the next boundary multiplied by the inverse step size, then right-shifted to normalize [S1, unofficial]. If the delta is zero, t-max is set to the maximum unsigned 32-bit integer [S1, unofficial].
    $$
    \begin{aligned}
    \text{tMaxX\_fp} &= \begin{cases} \left\lfloor \frac{|\text{nextX\_fp} - \text{startX\_fp}| \times \text{absDeltaX\_fp}}{\text{FP\_ONE}} \right\rfloor & \text{if } \Delta X_{\text{fp}} \neq 0 \\ \text{max}(u32) & \text{otherwise} \end{cases} \\
    \text{tMaxY\_fp} &= \begin{cases} \left\lfloor \frac{|\text{nextY\_fp} - \text{startY\_fp}| \times \text{absDeltaY\_fp}}{\text{FP\_ONE}} \right\rfloor & \text{if } \Delta Y_{\text{fp}} \neq 0 \\ \text{max}(u32) & \text{otherwise} \end{cases}
    \end{aligned}
    $$
    Implementation detail: The multiplication is performed using 64-bit arithmetic (`fl::i64(...) * ...`) before the right shift to avoid overflow [S1, unofficial].

8.  **Initialize Loop State:**
    $$
    \begin{aligned}
    \text{maxT\_fp} &= \text{FP\_ONE} \\
    \text{currentX} &= x_0 \\
    \text{currentY} &= y_0
    \end{aligned}
    $$
    [S1, unofficial]

9.  **Traverse Cells:**
    Loop indefinitely:
    a. Call `visitor.visit(currentX, currentY)` [S1, unofficial].
    b. Determine the next crossing time:
       $$ t_{\text{fp}} = \min(\text{tMaxX\_fp}, \text{tMaxY\_fp}) $$
    c. If $t_{\text{fp}} > \text{maxT\_fp}$, break the loop [S1, unofficial].
    d. If $\text{tMaxX\_fp} < \text{tMaxY\_fp}$:
       $$
       \begin{aligned}
       \text{tMaxX\_fp} &+= \text{absDeltaX\_fp} \\
       \text{currentX} &+= \text{stepX}
       \end{aligned}
       $$
       Else:
       $$
       \begin{aligned}
       \text{tMaxY\_fp} &+= \text{absDeltaY\_fp} \\
       \text{currentY} &+= \text{stepY}
       \end{aligned}
       $$
       [S1, unofficial]

10. **Ensure End Cell Visit:**
    After the loop, if `currentX != x1` or `currentY != y1`, explicitly call `visitor.visit(x1, y1)` to ensure the final cell is included [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverseGridSegment32(start_x, start_y, end_x, end_y, visitor):
    FP_SHIFT = 8
    FP_ONE = 256  # 1 << 8
    
    # 1. Convert to fixed-point
    startX_fp = int(start_x * FP_ONE)
    startY_fp = int(start_y * FP_ONE)
    endX_fp = int(end_x * FP_ONE)
    endY_fp = int(end_y * FP_ONE)
    
    # 2. Initial cell coordinates
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
    
    # 5. Fixed-point t-deltas
    # Use 64-bit arithmetic for numerator to avoid overflow
    numerator = FP_ONE << FP_SHIFT  # 256 << 8 = 65536
    
    if deltaX_fp != 0:
        absDeltaX_fp = abs(numerator // deltaX_fp)
    else:
        absDeltaX_fp = 0xFFFFFFFF  # max u32
        
    if deltaY_fp != 0:
        absDeltaY_fp = abs(numerator // deltaY_fp)
    else:
        absDeltaY_fp = 0xFFFFFFFF  # max u32
        
    # 6. Next intersection coordinates
    if stepX > 0:
        nextX_fp = (x0 + 1) << FP_SHIFT
    else:
        nextX_fp = x0 << FP_SHIFT
        
    if stepY > 0:
        nextY_fp = (y0 + 1) << FP_SHIFT
    else:
        nextY_fp = y0 << FP_SHIFT
        
    # 7. Initial t-max values
    # Use 64-bit arithmetic for multiplication
    if deltaX_fp != 0:
        distX = abs(nextX_fp - startX_fp)
        tMaxX_fp = (distX * absDeltaX_fp) >> FP_SHIFT
    else:
        tMaxX_fp = 0xFFFFFFFF  # max u32
        
    if deltaY_fp != 0:
        distY = abs(nextY_fp - startY_fp)
        tMaxY_fp = (distY * absDeltaY_fp) >> FP_SHIFT
    else:
        tMaxY_fp = 0xFFFFFFFF  # max u32
        
    # 8. Initialize loop state
    maxT_fp = FP_ONE
    currentX = x0
    currentY = y0
    
    # 9. Traverse
    while True:
        visitor.visit(currentX, currentY)
        
        t_fp = tMaxX_fp if tMaxX_fp < tMaxY_fp else tMaxY_fp
        
        if t_fp > maxT_fp:
            break
            
        if tMaxX_fp < tMaxY_fp:
            tMaxX_fp += absDeltaX_fp
            currentX += stepX
        else:
            tMaxY_fp += absDeltaY_fp
            currentY += stepY
            
    # 10. Ensure end cell visit
    if currentX != x1 or currentY != y1:
        visitor.visit(x1, y1)
```

**Edge Cases and Pitfalls:**
- **Zero Delta:** If `deltaX_fp` or `deltaY_fp` is zero, the corresponding `absDelta` and `tMax` are set to `max(u32)` to prevent division by zero and effectively disable stepping on that axis [S1, unofficial].
- **End Cell Inclusion:** The loop condition `t_fp > maxT_fp` may terminate before the final cell `(x1, y1)` is visited [S1, unofficial]. An explicit check and visit after the loop is required [S1, unofficial].
- **Macro Expansion:** On platforms like Arduino, the `max` macro can interfere with function calls [S1, unofficial]. The source uses `(fl::numeric_limits<u32>::max)()` to prevent this [S1, unofficial].
- **Overflow:** The calculation of `absDelta` and `tMax` uses 64-bit intermediate arithmetic (`i64`) to prevent overflow when multiplying or dividing fixed-point values [S1, unofficial].

## Operating characteristics
Not given in the sources. [S1, unofficial]

## Background
This specific implementation is part of the FastLED library, which provides utilities for LED matrix manipulation, including grid traversal for effects that span multiple LEDs [S1, unofficial]. The algorithm is based on the classic Amanatides–Woo ray traversal method, adapted for fixed-point arithmetic [G].

## Sources
[S1] FastLED: fl/traverse_grid.h Source File, https://fastled.io/docs/d9/d42/traverse__grid_8h_source.html, fetched 2026-10-30 (unofficial)
