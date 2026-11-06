# Amanatides and Woo Variation with 2-Point Input

Also called: 2-Point Voxel Traversal, Improved Amanatides and Woo

## Overview
This method traverses a voxel grid by accepting two explicit 3D points (start and end) rather than a ray origin and direction vector [S1, unofficial]. It is designed to handle edge cases such as zero coordinate deltas, exact edge hits, negative traversal, and non-0-based block centers with higher precision than standard face-to-face traversal [S1, unofficial]. It builds on the Amanatides and Woo DDA logic but adapts the initialization and comparison logic to operate directly on point pairs [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `pointA` (start position, `Vector3`), `pointB` (end position, `Vector3`) [S1, unofficial].
- Output: An iterable sequence of `Vector3` coordinates representing the integer voxel centers (or grid indices) intersected by the line segment [S1, unofficial].

**Data Structures:**
- `x, y, z`: Current voxel coordinates (floats, initialized to rounded `pointA`) [S1, unofficial].
- `tMaxX, tMaxY, tMaxZ`: Distance along the segment to the next voxel boundary in each axis [S1, unofficial].
- `tDeltaX, tDeltaY, tDeltaZ`: Distance between consecutive voxel boundaries in each axis [S1, unofficial].
- `stepX, stepY, stepZ`: Direction signs (+1 or -1) for traversal [S1, unofficial].
- `lineLength`: Total Euclidean length of the segment `pointB - pointA` [S1, unofficial].

**Formulas and Definitions:**
- `delta = pointB - pointA` [S1, unofficial].
- `deltaNormal = delta / |delta|` (Normalized direction vector) [S1, unofficial].
- `dx, dy, dz`: Components of `deltaNormal` [S1, unofficial].
- `Signum(v)`: Returns 1 if $v > 0$, -1 if $v < 0$, and 0 if $v == 0$ [S1, unofficial].
- `Intbound(s, ds)`: Calculates the distance to the next integer boundary [S1, unofficial].
  - If $ds < 0$ and $s$ is an integer, return 0 [S1, unofficial].
  - Else, let $s' = s \pmod 1$ (using `Mod(v, m) = (v \% m + m) \% m`) [S1, unofficial].
  - If $ds > 0$, return $(\lceil s' \rceil - s') / |ds|$ [S1, unofficial]. Note: $\lceil s' \rceil$ returns 1 if $s' == 0$, otherwise standard ceiling [S1, unofficial].
  - If $ds < 0$, return $(s' - \lfloor s' \rfloor) / |ds|$ [S1, unofficial].
- `Precision5(val)`: Rounds `val` to 5 decimal places using integer arithmetic: `((int)(100000 * val)) / 100000.0f`. This mitigates floating-point precision issues during comparisons [S1, unofficial].

**Step Functions:**
Each step function checks if the current `tMax` exceeds `lineLength` [S1, unofficial]. If so, traversal ends (returns `true`) [S1, unofficial]. Otherwise, it increments the voxel coordinate by `step` and adds `tDelta` to `tMax` (returns `false`) [S1, unofficial].
- `StepX()`: If `tMaxX >= lineLength` return `true`. Else `x += stepX`, `tMaxX += tDeltaX`, return `false` [S1, unofficial].
- `StepY()`: Analogous for Y [S1, unofficial].
- `StepZ()`: Analogous for Z [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:**
   - Set `x = round(pointA.X)`, `y = round(pointA.Y)`, `z = round(pointA.Z)` [S1, unofficial].
   - Compute `delta`, `deltaNormal`, `dx, dy, dz` [S1, unofficial].
   - Compute `stepX, stepY, stepZ` using `Signum` [S1, unofficial].
   - Compute `tMaxX, tMaxY, tMaxZ` using `Intbound(pointA.X - 0.5f, deltaNormal.x)`, etc. [S1, unofficial].
   - Compute `tDeltaX = stepX / dx` (handle division by zero if `dx=0` by setting `tDeltaX` to infinity or skipping, though the source implies `dx` is non-zero when `stepX` is non-zero) [S1, unofficial].
   - Compute `lineLength = |delta|` [S1, unofficial].
   - Initialize result list with `(x, y, z)` [S1, unofficial].

2. **Iteration:**
   The iteration logic depends on which steps are non-zero [S1, unofficial]. The source provides specific logic for the 3D case (all steps non-zero) and mentions separate loops for 2D/1D cases [S1, unofficial].

   **3D Case (`stepX != 0 && stepY != 0 && stepZ != 0`):**
   Loop while `true`:
   - Compute `tX = Precision5(tMaxX)`, `tY = Precision5(tMaxY)`, `tZ = Precision5(tMaxZ)` [S1, unofficial].
   - Compare `tX, tY, tZ`:
     - If `tX < tY`:
       - If `tX < tZ`: `res = StepX()`
       - Else if `tZ < tX`: `res = StepZ()`
       - Else (`tZ == tX`): `res = StepZ() || StepX()` (Execute both, result is true if either ends traversal) [S1, unofficial].
     - Else if `tY < tX`:
       - If `tY < tZ`: `res = StepY()`
       - Else if `tZ < tY`: `res = StepZ()`
       - Else (`tZ == tY`): `res = StepY() || StepZ()` [S1, unofficial].
     - Else (`tX == tY`):
       - If `tY < tZ`: `res = StepX() || StepY()`
       - Else if `tZ < tY`: `res = StepZ()`
       - Else (`tZ == tY`): `res = StepX() || StepY() || StepZ()` [S1, unofficial].
   - Add current `(x, y, z)` to result list [S1, unofficial].
   - If `res` is `true`, break loop [S1, unofficial].

   **2D/1D Cases:**
   Separate `while` loops handle cases where one or two steps are zero [S1, unofficial]. The logic compares only the relevant `tMax` values [S1, unofficial]. For example, if `stepX != 0 && stepY == 0 && stepZ == 0`, only `StepX()` is called in a loop until it returns `true` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def signum(v):
    return 1 if v > 0 else (-1 if v < 0 else 0)

def mod(v, m):
    return (v % m + m) % m

def intbound(s, ds):
    if ds < 0 and s == math.floor(s):
        return 0.0
    s_mod = mod(s, 1.0)
    if ds > 0:
        ceil_s = 1.0 if s_mod == 0.0 else math.ceil(s_mod)
        return (ceil_s - s_mod) / abs(ds)
    else:
        return (s_mod - math.floor(s_mod)) / abs(ds)

def precision5(val):
    return float(int(100000 * val) / 100000)

def get_intersected_world_blocks(pointA, pointB):
    line = []
    x = round(pointA[0])
    y = round(pointA[1])
    z = round(pointA[2])
    line.append((x, y, z))

    dx_raw = pointB[0] - pointA[0]
    dy_raw = pointB[1] - pointA[1]
    dz_raw = pointB[2] - pointA[2]
    length = math.sqrt(dx_raw**2 + dy_raw**2 + dz_raw**2)
    if length == 0.0:
        return line

    dx = dx_raw / length
    dy = dy_raw / length
    dz = dz_raw / length

    stepX = signum(dx)
    stepY = signum(dy)
    stepZ = signum(dz)

    tMaxX = intbound(pointA[0] - 0.5, dx)
    tMaxY = intbound(pointA[1] - 0.5, dy)
    tMaxZ = intbound(pointA[2] - 0.5, dz)

    tDeltaX = stepX / dx if dx != 0.0 else 0.0
    tDeltaY = stepY / dy if dy != 0.0 else 0.0
    tDeltaZ = stepZ / dz if dz != 0.0 else 0.0

    def step_x():
        nonlocal x, tMaxX
        if tMaxX >= length:
            return True
        x += stepX
        tMaxX += tDeltaX
        return False

    def step_y():
        nonlocal y, tMaxY
        if tMaxY >= length:
            return True
        y += stepY
        tMaxY += tDeltaY
        return False

    def step_z():
        nonlocal z, tMaxZ
        if tMaxZ >= length:
            return True
        z += stepZ
        tMaxZ += tDeltaZ
        return False

    active_x = stepX != 0
    active_y = stepY != 0
    active_z = stepZ != 0

    while True:
        res = False
        if active_x and active_y and active_z:
            tX = precision5(tMaxX)
            tY = precision5(tMaxY)
            tZ = precision5(tMaxZ)
            if tX < tY:
                if tX < tZ:
                    res = step_x()
                elif tZ < tX:
                    res = step_z()
                else:
                    res = step_z() or step_x()
            elif tY < tX:
                if tY < tZ:
                    res = step_y()
                elif tZ < tY:
                    res = step_z()
                else:
                    res = step_y() or step_z()
            else:
                if tY < tZ:
                    res = step_x() or step_y()
                elif tZ < tY:
                    res = step_z()
                else:
                    res = step_x() or step_y() or step_z()
        elif active_x and not active_y and not active_z:
            res = step_x()
        elif not active_x and active_y and not active_z:
            res = step_y()
        elif not active_x and not active_y and active_z:
            res = step_z()
        elif active_x and active_y and not active_z:
            tX = precision5(tMaxX)
            tY = precision5(tMaxY)
            if tX < tY:
                res = step_x()
            elif tY < tX:
                res = step_y()
            else:
                res = step_x() or step_y()
        elif active_x and not active_y and active_z:
            tX = precision5(tMaxX)
            tZ = precision5(tMaxZ)
            if tX < tZ:
                res = step_x()
            elif tZ < tX:
                res = step_z()
            else:
                res = step_x() or step_z()
        elif not active_x and active_y and active_z:
            tY = precision5(tMaxY)
            tZ = precision5(tMaxZ)
            if tY < tZ:
                res = step_y()
            elif tZ < tY:
                res = step_z()
            else:
                res = step_y() or step_z()
        else:
            break

        line.append((x, y, z))
        if res:
            break

    return line
```

## Subtopics

### Adjacency-Based Ray Traversal
This variant uses the ray-volume impact point (from intersection check) as the first ray-voxel intersection point (entry point) [S2, unofficial]. It calculates the point where the ray emerges from the current voxel (exit point), tests the data between the entry and exit points against user-specified target values, and repeats traversal until the ray emerges from the data volume or available light is depleted [S2, unofficial]. Unlike Levy's uniform resampling which causes aliasing and misses interstitial surfaces, this method guarantees traversal through every voxel the ray intersects [S2, unofficial]. Uniform sampling requires infinite samples to guarantee finding all surfaces, or suffers from aliasing if steps are too large [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels traversed [S1, unofficial].
- **Memory Complexity:** Linear with respect to the number of voxels traversed, as all intersected voxels are stored in the output list [S1, unofficial].
- **Strengths:** Handles edge cases (zero deltas, exact edge hits, negative traversal) more robustly than standard implementations [S1, unofficial]. The `Precision5` helper reduces floating-point comparison errors [S1, unofficial].
- **Limitations:** The source notes it "might (and probably will) be slower" than standard face-to-face traversal due to the overhead of precision rounding and complex branching logic for equality cases [S1, unofficial].
- **When to use:** When high precision is required for voxel selection, especially with non-0-based block centers or when dealing with degenerate rays (zero length or axis-aligned) [S1, unofficial].

## Background
This method is described as a variation of the Amanatides and Woo algorithm, adapted for a voxel game context where blocks are centered at 0.5-based coordinates [S1, unofficial]. It was shared on GameDev Stack Exchange in 2013 to address specific edge cases not handled by existing code [S1, unofficial].

## Sources
[S1] Cast ray to select block in voxel game, https://gamedev.stackexchange.com/questions/47362, fetched 2026-10-25 (unofficial)
[S2] DTIC ADA202661: Volumetric Rendering Techniques for the Display of Three-Dimensional Aerodynamic Flow Field Data, https://archive.org/details/DTIC_ADA202661, fetched 2026-10-26 (unofficial)
