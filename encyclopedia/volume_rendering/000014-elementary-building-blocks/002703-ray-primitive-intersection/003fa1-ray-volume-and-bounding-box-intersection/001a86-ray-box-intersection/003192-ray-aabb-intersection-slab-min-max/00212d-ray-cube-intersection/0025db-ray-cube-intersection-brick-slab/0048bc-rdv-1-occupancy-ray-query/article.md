# RDV-1 Occupancy Ray Query

Also called: occupancy-only ray query, toroidal brick buffer ray march

## Overview
The RDV-1 Occupancy Ray Query is an occupancy-only ray tracing method designed for a toroidal brick buffer voxel grid. It traces a ray through the voxel space to locate the first occupied voxel, returning the voxel coordinates, surface normal, distance, and step counts [S1, unofficial]. It differs from its parent [Ray-Cube Intersection (Brick/Slab)](3620-ray-cube-intersection-brick-slab.md) by operating across the entire grid rather than a single brick, and from [In-Brick DDA for Occlusion](3623-in-brick-dda-for-occlusion.md) by supporting arbitrary ray origins and distances outside the brick boundary.

## How it works
**Inputs and Outputs**
*   **Input:** A ray defined by origin, direction, and maximum distance.
*   **Output:** A `BbvResult` struct containing the hit voxel coordinates, a hit flag, the surface normal, the distance to the hit, the number of voxel steps taken, and the number of brick lookups performed.

**Data Structures**
The method relies on global constants and specific struct layouts:

*   **Global Constants:**
    *   `gMinimumBrick` (`int3`): Minimum brick coordinate in the grid.
    *   `gGridDimensions` (`int3`): Dimensions of the brick grid.
    *   `gToroidalOffset` (`int3`): Offset applied for toroidal wrapping of brick indices.
    *   `gMaximumSteps` (`uint`): Maximum number of voxel steps allowed per ray trace.
    *   `gRayCount` (`uint`): Total number of rays dispatched to the kernel.

*   **Input Ray Struct (`BbvRay`):**
    ```cpp
    struct BbvRay {
        float3 origin;
        float maxDistance;
        float3 direction;
        uint rayId;
    };
    ```

*   **Output Result Struct (`BbvResult`):**
    ```cpp
    struct BbvResult {
        int3 voxel;
        uint hit;
        int3 normal;
        uint padding;
        float distance;
        uint voxelSteps;
        uint brickLookups;
        uint padding2;
    };
    ```

*   **Brick Buffer Layout:**
    Each brick occupies exactly 64 bytes. The occupancy data uses DVE's authoritative indexing scheme: `x + 8*y + 64*z`. This differs from InstantRDV's subbrick/Morton wire format [S1, unofficial].

**Algorithm Steps**

1.  **Validation:** Compute the magnitude of the ray direction. If `magnitude <= 0.0f` or `maxDistance < 0.0f`, return an empty `BbvResult` and exit.
2.  **Normalization and Bounds:** Normalize the direction. Compute the grid bounds:
    $$ \text{gridMin} = \text{float3}(gMinimumBrick) \times 8.0f $$
    $$ \text{gridMax} = \text{float3}(gMinimumBrick + gGridDimensions) \times 8.0f $$
    Initialize `entry = 0.0f`, `exit = ray.maxDistance`, and `normal = int3(0,0,0)`.
3.  **Axis Intersection:** For each axis ($x, y, z$):
    *   If `direction[axis] == 0.0f`:
        *   Check if `origin[axis]` is outside the range `[gridMin[axis], gridMax[axis])`.
        *   If outside, return an empty result. Otherwise, continue to the next axis.
    *   Compute $t_0$ and $t_1$ intersections for the slab.
    *   Update `entry`: If $\min(t_0, t_1) > \text{entry}$, set `entry = min(t0, t1)` and set `normal[axis]` to $-1$ or $1$ based on the sign of `direction[axis]`.
    *   Update `exit`: Set `exit = max(t0, t1)`.
4.  **Entry Validation and Offset:**
    *   If `entry > exit`, return an empty result.
    *   Compute the sample offset to avoid precision issues at the ray origin:
        $$ \text{sampleT} = \begin{cases} \text{entry} + \max(|\text{entry}| \times 1.1920929e-7f, 1.0e-5f) & \text{if } \text{entry} > 0.0f \\ 0.0f & \text{otherwise} \end{cases} $$
    *   Clamp the sample position to the grid bounds:
        $$ p = \text{clamp}(\text{ray.origin} + \text{direction} \times \text{sampleT}, \text{gridMin}, \text{gridMax} - 1.0e-4f) $$
    *   Set the initial voxel:
        $$ \text{voxel} = \text{int3}(\lfloor p \rfloor) $$
5.  **Ray Marching Loop:**
    *   Initialize `t = entry`.
    *   Loop up to $\min(gMaximumSteps, 65536u)$ iterations while $t \le exit$:
        *   Increment `voxelSteps`.
        *   If `voxel` is outside grid bounds, break.
        *   Call `IsOccupied(voxel, result.brickLookups)`.
        *   If occupied:
            *   Set `result.hit = 1`.
            *   Set `result.voxel = voxel`.
            *   Set `result.normal = normal`.
            *   Set `result.distance = t`.
            *   Break the loop.
        *   Compute next boundary crossings:
            $$ \text{nextT} = \text{float3}(\text{NextBoundary}(\text{ray.origin.x}, \text{direction.x}, \text{voxel.x}), \dots) $$
        *   Find the axis with the minimum `nextT`. Let this value be `next`.
        *   If `next > exit` or `next + 1.0e-5f < t`, break.
        *   Update $t = \max(t, \text{next})$.
        *   Advance `voxel[axis]` by $\delta$ ($1$ or $-1$).
        *   Update `normal` for the new axis.

**Helper Functions**

*   **IsOccupied:**
    Determines if a voxel is occupied using the brick buffer.
    1.  Calculate logical position: $\text{logical} = \text{key} - gMinimumBrick$.
    2.  Calculate physical index with toroidal wrapping:
        $$ \text{physical} = \text{logical} + gToroidalOffset $$
        $$ \text{index} = \text{physical.x} + gGridDimensions.x \times (\text{physical.y} + gGridDimensions.y \times \text{physical.z}) $$
    3.  Calculate local voxel position within the brick:
        $$ \text{local} = \text{voxel} - \text{key} \times 8 $$
    4.  Calculate bit position:
        $$ \text{bit} = \text{local.x} + 8 \times \text{local.y} + 64 \times \text{local.z} $$
    5.  Load the 64-byte brick and test the bit:
        $$ \text{word} = gBbvBricks.\text{Load}(\text{index} \times 64u + (\text{bit} \gg 5u) \times 4u) $$
        $$ \text{return } (\text{word} \& (1u \ll (\text{bit} \& 31u))) \neq 0u $$

*   **NextBoundary:**
    Computes the $t$ value where the ray crosses the next voxel boundary along an axis.
    $$ \text{NextBoundary}(o, d, v) = \begin{cases} \frac{v + 1 - o}{d} & \text{if } d > 0 \\ \frac{v - o}{d} & \text{if } d < 0 \end{cases} $$
    *(Note: Standard Amanatides-Woo slab intersection logic applies here, adapted for the specific boundary checks in the source.)*

**Parameters**
*   `gMinimumBrick`: Not given in the sources.
*   `gGridDimensions`: Not given in the sources.
*   `gToroidalOffset`: Not given in the sources.
*   `gMaximumSteps`: Not given in the sources.
*   `gRayCount`: Not given in the sources.

**Edge Cases and Pitfalls**
*   **Zero Direction Components:** If a direction component is zero, explicit bounds checking is required for that axis. Without it, the ray may incorrectly pass through the grid or cause division by zero [S1, unofficial].
*   **Precision at Origin:** The entry point offset uses `max(abs(entry) * 1.1920929e-7f, 1.0e-5f)` to avoid self-intersection or precision issues when the ray starts inside or very close to a voxel [S1, unofficial].
*   **Infinite Loops:** The step advancement includes a check `next + 1.0e-5f < t` to prevent infinite loops or stuck steps due to floating-point rounding errors at voxel boundaries [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels traversed, capped by `gMaximumSteps` and 65536.
*   **Memory Complexity:** Constant, dependent on the brick buffer size which is determined by `gGridDimensions`.
*   **Strengths:** Efficient for sparse grids due to occupancy-only checks; toroidal wrapping simplifies boundary handling for certain grid topologies.
*   **Limitations:** Not used by production render passes yet; limited to occupancy queries (no shading or density integration) [S1, unofficial].
*   **When to Use:** When only the first hit in a voxel grid is needed, and the grid uses the DVE brick buffer format.
*   **When Not to Use:** When density values, normals from geometry, or full ray-marching integration are required.

## Background
The RDV-1 Occupancy Ray Query is part of the Destructible Voxel Engine (DVE) implementation. It is an independent kernel, opt-in, and was not used by production render passes at the time of the source documentation [S1, unofficial]. It builds on the principles of ray-box intersection and Amanatides-Woo DDA, adapted for a toroidal brick buffer layout.

## Sources
[S1] HiroSakuraba/destructible-voxel-engine, https://github.com/HiroSakuraba/destructible-voxel-engine, fetched 2026-10-28 (unofficial)
