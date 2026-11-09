# AABB Collision Detection
Also called: Axis-Aligned Bounding Box Collision, Axis-Separated AABB Collision, Sliding Collision, Axis-Aligned Collision Sliding, AABB Collision with axis-separated movement

## Overview
AABB Collision Detection is a physics resolution method that prevents an Axis-Aligned Bounding Box (AABB) from penetrating solid voxel terrain by processing movement along each spatial axis independently [S1, S2, S3, unofficial]. It solves the problem of "sticking" or getting trapped inside walls by allowing the object to slide along surfaces rather than being completely halted by a single axis collision [S1, S3, unofficial]. Unlike continuous collision detection or swept-volume tests, this method uses discrete, axis-separated validation, which is computationally cheap but requires careful ordering to handle gravity and wall-sliding correctly [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Current AABB position (center or min/max), AABB dimensions (half-extents or full width/height/depth), velocity or intended displacement vector, and a voxel world data structure (grid) where `BlockID > 0` indicates solid terrain [S2, S3, unofficial].
*   **Output:** Updated AABB position that respects collision constraints, potentially reduced velocity if a collision occurred [S2, S3, unofficial].

**Data Structures:**
*   **Voxel World:** A 3D array or hash map mapping integer coordinates $(x, y, z)$ to block types. Solid blocks are identified by `BlockID > 0` [S3, unofficial].
*   **AABB:** Defined by center position $(c_x, c_y, c_z)$ and half-dimensions $(h_x, h_y, h_z)$ [S2, unofficial]. Alternatively defined by min/max bounds [S6310-aabb-representation-min-max.md].

**Parameters:**
*   **Camera/Player Dimensions:** Typical values include width 0.8, height 1.8, depth 0.8 (Minecraft-style) [S2, unofficial] or width 0.3, height 1.7 [S3, unofficial].
*   **Gravity:** $18.0$ units/s² [S3, unofficial].
*   **Jump Force:** $8.0$ units/s [S3, unofficial].
*   **Time Step:** Movement is often scaled by a time factor (e.g., `ms` or `dt`) [S1, unofficial].

**Algorithm Steps:**
1.  **Calculate Displacement:** Compute the intended movement vector $\Delta = \text{new\_position} - \text{current\_position}$ [S2, unofficial].
2.  **Axis Separation:** Iterate through each axis (typically X, then Z, then Y to handle gravity correctly) [S2, S3, unofficial].
3.  **Apply Axis Movement:** Tentatively update the position component for the current axis:
    $$ \text{target\_pos}[axis] = \text{current\_pos}[axis] + \Delta[axis] $$
    [S2, unofficial]
4.  **Define Query Volume:** Calculate the voxel grid bounds that the AABB covers at this tentative position.
    *   $\text{min\_voxel} = \lfloor \text{target\_pos} - \text{half\_dims} \rfloor$
    *   $\text{max\_voxel} = \lceil \text{target\_pos} + \text{half\_dims} \rceil$
    [S2, unofficial]
5.  **Collision Check:** Iterate through all voxel coordinates $(v_x, v_y, v_z)$ within the box defined by $\text{min\_voxel}$ and $\text{max\_voxel}$ [S2, S3, unofficial].
    *   *Optimization (S3):* Sample specific points (e.g., Feet, Waist, Head) around the player model (12 points total) instead of iterating every voxel [S3, unofficial].
    *   *Condition:* If any sampled point or voxel is solid (`BlockID > 0`), a collision is detected [S3, unofficial].
6.  **Resolution:**
    *   If collision detected: Revert the position component for this axis to its value before step 3. The movement along this axis is cancelled [S2, S3, unofficial].
    *   If no collision: Commit the tentative position for this axis [S2, S3, unofficial].
7.  **Repeat:** Proceed to the next axis.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def aabb_collision_step(camera_pos, camera_dims, velocity, world):
    """
    camera_pos: Vector3 (center of AABB)
    camera_dims: Vector3 (half-extents, e.g., [0.4, 0.9, 0.4])
    velocity: Vector3 (intended movement per frame)
    world: Function voxel(x, y, z) -> BlockID
    """
    # 1. Calculate delta
    delta = velocity
    
    # 2. Process axes independently (X, Z, Y order recommended for gravity)
    axes = ['x', 'z', 'y']
    
    for axis in axes:
        # 3. Apply tentative movement on current axis
        tentative_pos = camera_pos.copy()
        tentative_pos[axis] += delta[axis]
        
        # 4. Define voxel bounds for the AABB at tentative_pos
        # Calculate min and max voxel coordinates covered by the AABB
        min_voxel = floor(tentative_pos - camera_dims)
        max_voxel = ceil(tentative_pos + camera_dims)
        
        # 5. Check for collisions
        collided = False
        
        # Option A: Iterate all voxels in the bounding box [S2, unofficial]
        for x in range(min_voxel.x, max_voxel.x + 1):
            for y in range(min_voxel.y, max_voxel.y + 1):
                for z in range(min_voxel.z, max_voxel.z + 1):
                    if world.voxel(x, y, z) > 0: # Solid block
                        collided = True
                        break
                if collided: break
            if collided: break
            
        # Option B: Sample specific points [S3, unofficial]
        # points = get_sample_points(tentative_pos, camera_dims)
        # for p in points:
        #     if world.voxel(floor(p.x), floor(p.y), floor(p.z)) > 0:
        #         collided = True
        #         break
            
        # 6. Resolve
        if not collided:
            camera_pos[axis] = tentative_pos[axis]
        # If collided, axis remains at previous value (movement cancelled)
            
    return camera_pos
```

**Edge Cases and Pitfalls:**
*   **Stuck Bug:** Checking collision against the *current* cell (before moving) causes permanent sticking if the player starts inside a wall or due to floating-point drift. The check must be against the *next* cell (the target position) [S1, unofficial].
*   **Corner Penetration:** Simple axis checks might allow corners to clip into walls if not validated properly. A "BoxClear Corner Check" tests all four corners of the box on both axis moves to ensure no overlap occurs [S4, unofficial].
*   **Ordering:** The order of axis processing affects behavior. Processing Y last allows sliding along walls while still applying gravity/jumping correctly [S3, unofficial].

## Subtopics

### [AABB Voxel Intersection](6734-aabb-voxel-intersection.md)
Determines whether an AABB overlaps with any solid voxel in a voxel world by iterating through integer coordinates. Unlike hierarchical methods, it performs a direct brute-force check of every voxel coordinate within the bounding volume.

### BoxClear Corner Check
A collision check variant that tests all four corners of the player box on both axis moves to prevent overlap. It asserts that no corner ever enters a solid cell and that pulling away from the wall works from the pressed position, fixing issues where simple axis checks fail to prevent corner clipping [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the AABB's bounding box. For small characters, this is constant and very fast. Sampling optimizations (S3) reduce the constant factor significantly [S2, S3, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space, assuming the voxel world is accessed via random lookup [S2, unofficial].
*   **Strengths:** Simple to implement; allows natural sliding along walls; computationally inexpensive for small AABBs [S1, S3, unofficial].
*   **Limitations:** Discrete nature can cause tunneling at high speeds; corner clipping can occur without additional checks (BoxClear) [S4, unofficial]; performance degrades if the AABB is large relative to voxel size [S2, unofficial].
*   **When to Use:** Voxel-based games, simple platformers, real-time applications where performance is critical and high-speed tunneling is not a primary concern [S2, S3, unofficial].

## Background
Originating from early 3D raycasting engines and voxel games, this method was popularized by implementations in Wolfenstein 3D clones and Minecraft-like engines [S1, S4, unofficial]. It addresses the specific needs of grid-based worlds where continuous collision detection is overkill and discrete grid alignment simplifies spatial queries [S3, unofficial]. The "axis-separated" approach is a standard technique in 2D and 3D game physics to handle wall-sliding without complex rigid-body dynamics [S1, S3, unofficial].

## Sources
[S1] Как Я Заставил Esp8266 Рисовать 3d, Стрелять И Летать: Полный ..., https://habr.com/ru/articles/1097380/, fetched 2026-10-26 (unofficial)
[S2] Make yourself a Voxel Engine THIS weekend, https://daymare.net/blogs/voxel-engine-in-a-weekend/, fetched 2026-10-26 (unofficial)
[S3] GitHub - ernerdgn/voxel-engine: A voxel engine written from scratch in ..., https://github.com/ernerdgn/voxel-engine, fetched 2026-10-29 (unofficial)
[S4] A minimal Wolfenstein 3D raycasting engine, https://4rknova.com/blog/2026/07/06/wolfenstein-3d-raycasting, fetched 2026-10-24 (unofficial)
