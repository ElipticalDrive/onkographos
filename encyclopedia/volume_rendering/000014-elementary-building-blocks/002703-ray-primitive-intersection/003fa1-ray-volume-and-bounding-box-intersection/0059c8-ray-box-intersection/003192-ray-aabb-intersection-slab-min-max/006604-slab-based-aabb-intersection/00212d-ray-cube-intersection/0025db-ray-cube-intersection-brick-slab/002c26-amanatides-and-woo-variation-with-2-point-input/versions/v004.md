# Amanatides and Woo Variation with 2-Point Input

Also called: 2-Point Voxel Traversal, Amanatides and Woo ray marching, Improved Amanatides and Woo, Real Line Voxelization

## Overview
This method is a voxel traversal algorithm that takes two endpoints (start and end points) instead of a ray origin and direction vector. It is designed to be more precise than standard face-to-face traversal, specifically handling edge cases such as zero coordinate deltas, exact edge hits, negative traversal, and non-0-based block centers [S1, unofficial]. It builds on the foundational Amanatides and Woo DDA logic but adapts the initialization and termination conditions for finite line segments [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** `pointA` (start point, `Vector3`), `pointB` (end point, `Vector3`).
*   **Output:** An iterable sequence of voxel coordinates (`IEnumerable<Vector3>` or list of `Vector3`) representing the voxels intersected by the line segment.

**Data Structures:**
*   `x, y, z`: Current voxel integer coordinates.
*   `tMaxX, tMaxY, tMaxZ`: Distance along the line to the next voxel boundary in each axis.
*   `tDeltaX, tDeltaY, tDeltaZ`: Distance along the line between consecutive voxel boundaries in each axis.
*   `stepX, stepY, stepZ`: Direction signs (+1 or -1) for traversal in each axis.
*   `lineLength`: Total Euclidean length of the segment `pointB - pointA`.

**Formulas and Definitions:**
*   Line equation: $p = p_0 + vt$, where $p_0$ is start position, $v$ is normalized direction, and $t$ is scaled distance [S2, unofficial].
*   `Signum(d)`: Returns 1 if $d > 0$, -1 if $d < 0$, and 0 if $d == 0$.
*   `Mod(v, m)`: Returns $(v \% m + m) \% m$ to ensure positive modulo [S1, unofficial].
*   `Ceil(s)`: Returns 1 if $s == 0$, else returns $\lceil s \rceil$ [S1, unofficial].
*   `Intbound(s, ds)`: Calculates the distance to the next integer boundary [S1, unofficial]:
    *   If $ds < 0$ and $s$ is an integer, return 0.
    *   Else, $s = \text{Mod}(s, 1)$.
    *   If $ds > 0$, return $(\text{Ceil}(s) - s) / |ds|$.
    *   Else, return $(s - \text{Floor}(s)) / |ds|$.
*   `Precision5(val)`: Rounds float to 5 decimal places using integer arithmetic: `((int)(100000 * val)) / 100000.0f`. Used to mitigate floating-point precision issues in comparisons [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Round start point to nearest integer grid: $x = \text{round}(pointA.X)$, $y = \text{round}(pointA.Y)$, $z = \text{round}(pointA.Z)$.
    *   Compute $\delta = pointB - pointA$.
    *   Compute normalized direction $\delta_{normal} = \delta / \|\delta\|$.
    *   Extract components: $dx = \delta_{normal}.X$, $dy = \delta_{normal}.Y$, $dz = \delta_{normal}.Z$.
    *   Compute steps: $stepX = \text{Signum}(dx)$, $stepY = \text{Signum}(dy)$, $stepZ = \text{Signum}(dz)$.
    *   Compute initial distances to next boundary. Note: The source uses `pointA - 0.5f` in the `Intbound` call, accounting for .5f-based block centers [S1, unofficial]:
        *   $tMaxX = \text{Intbound}(pointA.X - 0.5f, dx)$
        *   $tMaxY = \text{Intbound}(pointA.Y - 0.5f, dy)$
        *   $tMaxZ = \text{Intbound}(pointA.Z - 0.5f, dz)$
    *   Compute step deltas:
        *   $tDeltaX = stepX / dx$
        *   $tDeltaY = stepY / dy$
        *   $tDeltaZ = stepZ / dz$
    *   Compute total length: $lineLength = \|\delta\|$.
    *   Initialize result list with starting voxel $(x, y, z)$.

2.  **Step Functions:**
    *   `StepX()`: If $tMaxX \ge lineLength$, return `true` (end of line). Else, $x \leftarrow x + stepX$, $tMaxX \leftarrow tMaxX + tDeltaX$, return `false`.
    *   `StepY()`: Analogous to `StepX()` using $y, tMaxY, tDeltaY$.
    *   `StepZ()`: Analogous to `StepX()` using $z, tMaxZ, tDeltaZ$.

3.  **Iteration:**
    The iteration logic depends on which axes have non-zero steps.

    *   **Case 1: 3D Traversal ($stepX \neq 0, stepY \neq 0, stepZ \neq 0$)**
        Loop while true:
        1.  Compare precision-rounded values: $tX = \text{Precision5}(tMaxX)$, $tY = \text{Precision5}(tMaxY)$, $tZ = \text{Precision5}(tMaxZ)$.
        2.  Determine smallest $t$:
            *   If $tX < tY$:
                *   If $tX < tZ$: call `StepX()`.
                *   Else if $tZ < tX$: call `StepZ()`.
                *   Else ($tZ == tX$): call `StepZ() || StepX()` (advance in both).
            *   Else if $tY < tX$:
                *   If $tY < tZ$: call `StepY()`.
                *   Else if $tZ < tY$: call `StepZ()`.
                *   Else ($tZ == tY$): call `StepY() || StepZ()` (advance in both).
            *   Else ($tX == tY$):
                *   If $tY < tZ$: call `StepX() || StepY()` (advance in both).
                *   Else if $tZ < tY$: call `StepZ()`.
                *   Else ($tZ == tY$): call `StepX() || StepY() || StepZ()` (advance in all three).
        3.  Add current $(x, y, z)$ to result list.
        4.  If any step function returned `true`, break loop.

    *   **Case 2: 2D Traversal (One step is zero)**
        Separate loops handle cases where one or two steps are zero.
        *   Example: If $stepX \neq 0, stepY \neq 0, stepZ == 0$:
            Loop while true:
            1.  Compare $tY = \text{Precision5}(tMaxY)$ and $tZ = \text{Precision5}(tMaxZ)$? No, $stepZ=0$ implies no movement in Z, but $tMaxZ$ still exists. However, the source code snippet for 2D cases (implied by `stepX == 0 && stepY != 0 && stepZ != 0`) compares relevant `tMax` values.
            2.  If $stepZ == 0$, the logic simplifies to comparing only the active axes. If $stepX \neq 0, stepY \neq 0, stepZ == 0$:
                *   Compare $tX$ and $tY$.
                *   If $tX < tY$: `StepX()`.
                *   Else if $tY < tX$: `StepY()`.
                *   Else: `StepX() || StepY()`.
            3.  Add current $(x, y, z)$ to result.
            4.  Break if end reached.

    *   **Case 3: 1D Traversal (Two steps are zero)**
        *   Example: If $stepX \neq 0, stepY == 0, stepZ == 0$:
            Loop while true:
            1.  Call `StepX()`.
            2.  Add current $(x, y, z)$ to result.
            3.  Break if end reached.

    *   **Termination:**
        The iteration terminates when a step function returns `true`, which occurs when the current $tMax$ exceeds or equals `lineLength` [S1, unofficial]. Alternatively, in the RLV formulation, termination occurs when the sum of all $t_{min}$ components exceeds the length of the line [S2, unofficial]. The implementation in [S1, unofficial] uses the `tMax >= lineLength` check within the step functions.

**Pitfalls and Edge Cases:**
*   **Zero Delta:** If a coordinate delta is zero (straight line parallel to an axis), the corresponding step is 0, and the algorithm must handle the reduced dimensionality (2D or 1D loops) [S1, unofficial].
*   **Exact Edge Hits:** When $tMax$ values are equal, the algorithm advances in multiple axes simultaneously to avoid missing voxels [S1, unofficial].
*   **Negative Traversal:** The `Signum` function handles negative directions correctly [S1, unofficial].
*   **Block Centers:** The algorithm accounts for blocks with .5f-based centers by subtracting 0.5f in the `Intbound` calculation [S1, unofficial].
*   **Precision:** Floating-point errors in comparisons are mitigated by `Precision5` [S1, unofficial].

## Subtopics

### Adjacency-Based Ray Traversal
This variant uses the ray-volume impact point (from an initial intersection check) as the first ray-voxel intersection point (entry point) [S3, unofficial]. It calculates the point where the ray emerges from the current voxel (exit point) and tests the data between these points against user-specified target values [S3, unofficial]. It repeats this process until the ray emerges from the data volume or available light is depleted [S3, unofficial]. Unlike Levy's uniform resampling, which requires infinite samples to guarantee finding all surfaces or suffers from aliasing if steps are too large, this method guarantees traversal through every voxel the ray intersects [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels intersected by the line segment. Each step involves constant-time arithmetic and comparisons.
*   **Memory Complexity:** Linear with respect to the number of voxels intersected, as it stores the list of visited voxels.
*   **Strengths:** High precision compared to face-to-face traversal; handles edge cases like zero deltas and exact boundary hits robustly; works with finite line segments rather than infinite rays [S1, unofficial].
*   **Limitations:** Potentially slower than standard Amanatides and Woo due to the overhead of handling 2-point input and precision rounding [S1, unofficial]. Requires careful handling of degenerate cases (zero steps) [S1, unofficial].
*   **When to use:** When precise voxelization of a finite line segment is required, especially in contexts where edge cases (zero deltas, exact hits) are common, such as in voxel games or specific scientific visualizations [S1, unofficial].
*   **When not to use:** When performance is critical and approximate traversal is acceptable, or when working with infinite rays where standard Amanatides and Woo is sufficient.

## Background
The algorithm is a variation of the Amanatides and Woo ray marching algorithm, originally proposed for voxelizing lines in a uniform grid [S2, unofficial]. The specific 2-point input variation was developed to address precision issues and edge cases in practical implementations, such as those found in voxel-based games [S1, unofficial]. The underlying concept of Real Line Voxelization (RLV) is discussed in literature comparing optimal scanline voxelization algorithms [S2, unofficial]. The adjacency-based variant is described in the context of volumetric rendering techniques for aerodynamic flow field data [S3, unofficial].

## Sources
[S1] Cast ray to select block in voxel game, https://gamedev.stackexchange.com/questions/47362, fetched 2026-10-25 (unofficial)
[S2] PDF A Comparison of Optimal Scanline Voxelization Algorithms, https://liu.diva-portal.org/smash/get/diva2:1459330/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
[S3] DTIC ADA202661: Volumetric Rendering Techniques for the Display of Three-Dimensional Aerodynamic Flow Field Data, https://archive.org/details/DTIC_ADA202661, fetched 2026-10-26 (unofficial)
