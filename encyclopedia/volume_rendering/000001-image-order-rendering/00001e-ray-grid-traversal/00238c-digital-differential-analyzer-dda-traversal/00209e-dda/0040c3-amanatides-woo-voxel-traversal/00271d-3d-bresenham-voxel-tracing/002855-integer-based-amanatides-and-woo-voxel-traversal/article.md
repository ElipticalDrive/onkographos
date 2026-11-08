# Integer-based Amanatides and Woo Voxel Traversal

Also called: Exact integer grid traversal, Integer-based Fast Grid Traversal, Integer-based ray marching

## Overview
This method implements the Amanatides and Woo voxel traversal algorithm using scaled integer arithmetic to determine the sequence of grid cells intersected by a ray, ensuring exactness and stability by avoiding floating-point operations [S1, unofficial]. It sets itself apart from its parent [3D Bresenham Voxel Tracing](3949-3d-bresenham-voxel-tracing.md) by explicitly calculating boundary crossing distances via scaled cross-products rather than incremental error accumulation, and from its sibling [Voxel Traversal with Rounding Error Correction](6207-voxel-traversal-with-rounding-error-correction.md) by relying on 64-bit integer precision to maintain exactness without post-hoc correction steps [S1, unofficial].

## How it works
**Inputs:**
- `startX, startY, startZ`: Integer grid coordinates of the starting voxel.
- `originX, originY, originZ`: Ray origin in world space.
- `vx, vy, vz`: Ray direction vector components, scaled to integers.
- `cellSize`: Width of a single voxel in world units.
- `justOutX, justOutY, justOutZ`: Predefined boundary values indicating the grid exit for each axis.

**Outputs:**
- The first object hit in the voxel grid, or `NULL` if the ray exits the grid without hitting an object.

**Data Structures:**
- `X, Y, Z`: Current voxel coordinates (32-bit signed integers).
- `stepX, stepY, stepZ`: Direction indicators (+1 or -1) for each axis.
- `iDeltaX, iDeltaY, iDeltaZ`: Scaled delta distances to the next boundary crossing for each axis (64-bit signed integers, `long long`).
- `iMaxX, iMaxY, iMaxZ`: Scaled distances to the *first* boundary crossing for each axis (64-bit signed integers, `long long`).

**Parameters:**
- Typical values for `cellSize` depend on the scene scale. `vx, vy, vz` must be scaled consistently with `origin` and `cellSize`.
- The scaling factor $S$ is implicitly $|v_x v_y v_z|$. The algorithm avoids division by multiplying the standard floating-point $t_{max}$ formula by this factor.

**Steps:**

1.  **Initialization Phase:**
    1.  Set current voxel coordinates: $X = startX$, $Y = startY$, $Z = startZ$.
    2.  Determine step directions:
        $$ stepX = \begin{cases} 1 & \text{if } v_x \ge 0 \\ -1 & \text{otherwise} \end{cases} $$
        (Repeat for $stepY$ and $stepZ$).
    3.  Calculate the world-space coordinate of the next grid boundary for each axis:
        $$ nextBoundaryX = \begin{cases} (X + 1) \cdot cellSize & \text{if } stepX > 0 \\ X \cdot cellSize & \text{if } stepX \le 0 \end{cases} $$
        (Repeat for $Y$ and $Z$).
    4.  Compute scaled delta distances ($iDelta$) for each axis. These represent the change in scaled distance when crossing one voxel:
        $$ iDeltaX = cellSize \cdot |v_y \cdot v_z| $$
        $$ iDeltaY = cellSize \cdot |v_x \cdot v_z| $$
        $$ iDeltaZ = cellSize \cdot |v_x \cdot v_y| $$
        *Note: Use 64-bit integers (`long long`) to prevent overflow during multiplication.*
    5.  Initialize scaled max distances ($iMax$) to the first boundary crossing. The standard floating-point formula $t_{max} = \frac{boundary - origin}{v}$ is scaled by multiplying by $|v_x v_y v_z|$.
        $$ iMaxX = (nextBoundaryX - originX) \cdot v_y \cdot v_z $$
        If $v_x < 0$, negate $iMaxX$ to correct for the direction of division.
        If $v_x = 0$, set $iMaxX = LLONG\_MAX$ (parallel ray).
        (Repeat for $iMaxY$ and $iMaxZ$).

2.  **Incremental Phase:**
    1.  Enter a loop. Compare $iMaxX$, $iMaxY$, and $iMaxZ$ to find the smallest value, indicating the next axis to cross.
    2.  If $iMaxX$ is the smallest:
        - Increment $X$ by $stepX$.
        - If $X == justOutX$, return `NULL` (ray exited grid).
        - Update $iMaxX = iMaxX + iDeltaX$.
    3.  Else if $iMaxY$ is the smallest:
        - Increment $Y$ by $stepY$.
        - If $Y == justOutY$, return `NULL`.
        - Update $iMaxY = iMaxY + iDeltaY$.
    4.  Else ($iMaxZ$ is the smallest):
        - Increment $Z$ by $stepZ$.
        - If $Z == justOutZ$, return `NULL`.
        - Update $iMaxZ = iMaxZ + iDeltaZ$.
    5.  Check the voxel list at the current $(X, Y, Z)$ coordinates for an object hit.
    6.  If an object is hit, return it. Otherwise, continue the loop.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def integer_amanatides_woo_traversal(startX, startY, startZ, originX, originY, originZ, vx, vy, vz, cellSize, justOutX, justOutY, justOutZ):
    # 1. Initialization Phase
    X = startX
    Y = startY
    Z = startZ

    stepX = 1 if vx >= 0 else -1
    stepY = 1 if vy >= 0 else -1
    stepZ = 1 if vz >= 0 else -1

    # Calculate distance to the first boundary in world units
    nextBoundaryX = (X + 1) * cellSize if stepX > 0 else X * cellSize
    nextBoundaryY = (Y + 1) * cellSize if stepY > 0 else Y * cellSize
    nextBoundaryZ = (Z + 1) * cellSize if stepZ > 0 else Z * cellSize

    # Compute scale factors (Absolute values of cross-products)
    # Use 64-bit integers to prevent overflow
    iDeltaX = cellSize * abs(vy * vz)
    iDeltaY = cellSize * abs(vx * vz)
    iDeltaZ = cellSize * abs(vx * vy)

    # Initialize iMax (Scaled t to next boundary)
    # Logic: (nextBoundary - origin) * (product of other two velocity components)
    # Correct sign for negative velocity components
    
    # For X
    iMaxX = (nextBoundaryX - originX) * vy * vz
    if vx < 0:
        iMaxX = -iMaxX
    if vx == 0:
        iMaxX = float('inf') # Representing LLONG_MAX conceptually

    # For Y
    iMaxY = (nextBoundaryY - originY) * vx * vz
    if vy < 0:
        iMaxY = -iMaxY
    if vy == 0:
        iMaxY = float('inf')

    # For Z
    iMaxZ = (nextBoundaryZ - originZ) * vx * vy
    if vz < 0:
        iMaxZ = -iMaxZ
    if vz == 0:
        iMaxZ = float('inf')

    # 2. Incremental Phase
    hit = None

    while hit is None:
        # Find the smallest iMax
        if iMaxX < iMaxY:
            if iMaxX < iMaxZ:
                # X is smallest
                X += stepX
                if X == justOutX:
                    return None
                iMaxX += iDeltaX
            else:
                # Z is smallest
                Z += stepZ
                if Z == justOutZ:
                    return None
                iMaxZ += iDeltaZ
        else:
            if iMaxY < iMaxZ:
                # Y is smallest
                Y += stepY
                if Y == justOutY:
                    return None
                iMaxY += iDeltaY
            else:
                # Z is smallest
                Z += stepZ
                if Z == justOutZ:
                    return None
                iMaxZ += iDeltaZ

        # Check voxel list at current (X, Y, Z)
        hit = check_voxel_object_list(X, Y, Z)

    return hit
```

**Edge Cases and Pitfalls:**
- **Parallel Rays:** If a velocity component is 0 (e.g., $v_x = 0$), the corresponding $iMax$ is set to infinity (or `LLONG_MAX`), effectively removing that axis from the minimum comparison. This prevents division by zero and correctly handles rays parallel to grid planes [S1, unofficial].
- **Integer Overflow:** The intermediate calculations for $iDelta$ and $iMax$ involve products of velocity components and coordinates. Using 64-bit integers (`long long`) is necessary to prevent overflow for large grids or high-precision direction vectors [S1, unofficial].
- **Grid Exit:** The loop terminates when the incremented coordinate equals the predefined `justOut` boundary for that axis. This must be checked immediately after incrementing [S1, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels traversed. Each iteration performs a constant number of integer comparisons and additions.
- **Memory Complexity:** $O(1)$ auxiliary space. The algorithm only stores a few integer variables.
- **Performance:** Significantly faster than floating-point Amanatides-Woo on hardware lacking efficient FPUs or where integer arithmetic is preferred for determinism. It avoids the cost of division and floating-point conversion.
- **Strengths:** Exactness (no rounding errors in traversal order), stability, and speed. Ideal for fixed-grid environments like voxel games or exact collision detection.
- **Limitations:** Requires careful scaling of input values to fit within 64-bit integer limits. Not suitable for dynamic grids where cell sizes vary per axis unless adapted. Precision is limited by the integer range; very small steps or very large grids may require higher precision types or fixed-point variants.

## Background
The original Amanatides and Woo algorithm (1987) is a standard method for voxel traversal using floating-point arithmetic. This integer-based variant was proposed in an unofficial GitHub issue discussion to provide a fast, exact, and stable alternative for applications like bullet hit detection in games, where floating-point precision issues could cause rays to pass through voxels incorrectly [S1, unofficial]. It mirrors the logic of the original paper but replaces floating-point operations with scaled integer arithmetic.

## Sources
[S1] Fast Grid Traversal Algorithm integer based by Gemini 3.0 Pro (verified in Delphi/translated seems to work ! ;)), https://github.com/OpenXcom/OpenXcom/issues/1483, fetched 2026-10-25 (unofficial)
