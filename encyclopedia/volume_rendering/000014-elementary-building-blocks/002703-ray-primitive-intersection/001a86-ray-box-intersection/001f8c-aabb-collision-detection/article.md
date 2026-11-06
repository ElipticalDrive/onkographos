# AABB Collision Detection
Also called: Axis-Aligned Bounding Box Collision, Axis-Aligned Bounding Box collision with axis-separated movement, Axis-Aligned Collision Sliding, Axis-Separated AABB Collision, Sliding Collision

## Overview
AABB Collision Detection is a movement resolution method for voxel worlds that prevents entities (players, cameras, NPCs) from passing through solid blocks by separating movement into individual axis checks [S1, S2, S3, unofficial]. It applies displacement along one axis, tests for intersection with the voxel grid, and commits or cancels that movement before proceeding to the next axis, enabling sliding along walls [S1, S3, unofficial].

## How it works
**Inputs:**
- Entity position $P = (x, y, z)$, typically defined as the center of its AABB [S2, unofficial].
- Entity half-dimensions $h = (h_x, h_y, h_z)$ [S2, S3, unofficial].
- Intended displacement vector $V = (v_x, v_y, v_z)$ [S2, S3, unofficial].
- Voxel world data structure (e.g., 2D/3D array `worldMap`) mapping integer coordinates to block types [S1, S3, unofficial].

**Outputs:**
- Updated entity position $P'$ that respects collision constraints [S2, S3, unofficial].

**Data Structures:**
- **Voxel Grid:** A discrete array mapping integer coordinates to block types [S1, S3, unofficial].
- **AABB:** Defined by center $P$ and half-dimensions $h$. For a camera, typical dimensions are $(0.8, 1.8, 0.8)$ [S2, unofficial]. For a player, typical dimensions are Width $0.3$, Height $1.7$ [S3, unofficial].

**Steps:**
1. **Calculate Target Position:** Compute the hypothetical new position if no collisions occurred: $P_{target} = P + V$ [S2, unofficial].
2. **Compute Delta:** Calculate the displacement vector $\Delta = P_{target} - P$ [S2, unofficial].
3. **Axis-Separated Resolution:** Iterate through axes (typically X, then Z, then Y to handle gravity/jumping) [S3, unofficial]:
    a. **Apply Axis Movement:** Create a tentative position $P_{tentative}$ by adding only the current axis component of $\Delta$ to the current safe position $P_{safe}$ [S2, S3, unofficial].
    b. **Define AABB Bounds:** Calculate the voxel grid indices covered by the entity's AABB at $P_{tentative}$ [S2, unofficial]:
       $$min_i = \lfloor P_{tentative, i} - h_i \rfloor$$
       $$max_i = \lceil P_{tentative, i} + h_i \rceil$$
       where $i$ is the axis index (0, 1, 2) [S2, unofficial].
    c. **Collision Check:**
       - *Method A (Voxel Sweep):* Iterate through all voxels in the box defined by $min$ and $max$. If any voxel is occupied (BlockID > 0 [S3, unofficial] or not none [S2, unofficial]), a collision is detected.
       - *Method B (Point Sampling):* Sample specific points on the entity's boundary (e.g., Feet, Waist, Head) against the voxel grid. If any point intersects a solid voxel, a collision is detected [S3, unofficial].
    d. **Resolve:** If collision detected, discard the movement for this axis ($P_{safe}$ remains unchanged). If no collision, update $P_{safe}$ with the new coordinate for this axis [S2, S3, unofficial].
4. **Final Position:** The resulting $P_{safe}$ is the new entity position [S2, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def aabb_collision_detection(position, velocity, world, entity_dims):
    """
    position: tuple (x, y, z) - center of AABB
    velocity: tuple (vx, vy, vz) - intended movement
    world: function voxel(x, y, z) -> int (0 for air, >0 for solid)
    entity_dims: tuple (hx, hy, hz) - half-extents of AABB
    """
    # Step 1 & 2: Calculate target and delta
    target_x = position[0] + velocity[0]
    target_y = position[1] + velocity[1]
    target_z = position[2] + velocity[2]
    
    delta_x = target_x - position[0]
    delta_y = target_y - position[1]
    delta_z = target_z - position[2]
    
    # Current safe position starts as original position
    safe_x = position[0]
    safe_y = position[1]
    safe_z = position[2]
    
    # Step 3: Axis-Separated Resolution (Order: X, Z, Y)
    
    # X Axis
    tentative_x = safe_x + delta_x
    min_x = math.floor(tentative_x - entity_dims[0])
    max_x = math.ceil(tentative_x + entity_dims[0])
    min_y = math.floor(safe_y - entity_dims[1])
    max_y = math.ceil(safe_y + entity_dims[1])
    min_z = math.floor(safe_z - entity_dims[2])
    max_z = math.ceil(safe_z + entity_dims[2])
    
    collided_x = False
    for x in range(min_x, max_x + 1):
        for y in range(min_y, max_y + 1):
            for z in range(min_z, max_z + 1):
                if world(x, y, z) > 0:
                    collided_x = True
                    break
            if collided_x: break
        if collided_x: break
        
    if not collided_x:
        safe_x = tentative_x
        
    # Z Axis
    tentative_z = safe_z + delta_z
    min_x = math.floor(safe_x - entity_dims[0])
    max_x = math.ceil(safe_x + entity_dims[0])
    min_y = math.floor(safe_y - entity_dims[1])
    max_y = math.ceil(safe_y + entity_dims[1])
    min_z = math.floor(tentative_z - entity_dims[2])
    max_z = math.ceil(tentative_z + entity_dims[2])
    
    collided_z = False
    for x in range(min_x, max_x + 1):
        for y in range(min_y, max_y + 1):
            for z in range(min_z, max_z + 1):
                if world(x, y, z) > 0:
                    collided_z = True
                    break
            if collided_z: break
        if collided_z: break
        
    if not collided_z:
        safe_z = tentative_z
        
    # Y Axis (Gravity/Jump)
    tentative_y = safe_y + delta_y
    min_x = math.floor(safe_x - entity_dims[0])
    max_x = math.ceil(safe_x + entity_dims[0])
    min_y = math.floor(tentative_y - entity_dims[1])
    max_y = math.ceil(tentative_y + entity_dims[1])
    min_z = math.floor(safe_z - entity_dims[2])
    max_z = math.ceil(safe_z + entity_dims[2])
    
    collided_y = False
    for x in range(min_x, max_x + 1):
        for y in range(min_y, max_y + 1):
            for z in range(min_z, max_z + 1):
                if world(x, y, z) > 0:
                    collided_y = True
                    break
            if collided_y: break
        if collided_y: break
        
    if not collided_y:
        safe_y = tentative_y
        
    return (safe_x, safe_y, safe_z)
```

**Parameters:**
- **Gravity:** Typically $18.0$ units/s² [S3, unofficial].
- **Jump Force:** Typically $8.0$ units/s [S3, unofficial].
- **Player Dimensions:** Width $0.3$, Height $1.7$ [S3, unofficial].
- **Camera Dimensions:** $(0.8, 1.8, 0.8)$ [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Stuck Bug:** Checking collision against the *current* cell causes permanent sticking if the player starts inside a wall or moves into one [S1, unofficial]. The solution is to check the *next* cell (the target position) [S1, unofficial].
- **Corner Penetration:** Simple axis checks may allow corners to penetrate walls slightly if the step size is large, leading to "entombment" in doors or walls [S4, unofficial]. This can be mitigated by testing all four corners of the AABB on both axis moves [S4, unofficial].

## Subtopics

### [AABB Voxel Intersection](6734-aabb-voxel-intersection.md)
Checks if two voxel sprites intersect by computing the axis-aligned bounding box overlap and iterating through all voxel coordinates within that overlap [6734-aabb-voxel-intersection.md].

### BoxClear Corner Check
A variant that tests all four corners of the player box on both axis moves to prevent overlap [S4, unofficial]. It asserts that no corner ever enters a solid cell and ensures that pulling away from a wall works from the pressed position, solving the "entombment" bug where simple axis checks fail to escape narrow gaps [S4, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Allows sliding along walls [S1, S3, unofficial]. Prevents tunneling through thin walls if step sizes are small [S1, unofficial].
- **Limitations:** Can be computationally expensive if the AABB is large or covers many empty voxels [S2, unofficial]. Prone to "stuck" bugs if not implemented with next-cell checks [S1, unofficial]. May require corner checks for robustness in narrow passages [S4, unofficial].

## Background
AABB Collision Detection is used in voxel engines and raycasting games for player and camera movement resolution [S1, S2, S3, unofficial]. It originates from implementations requiring axis-separated movement checks to resolve continuous entity motion against discrete voxel geometry [S1, S3, unofficial].

## Sources
[S1] Как Я Заставил Esp8266 Рисовать 3d, Стрелять И Летать: Полный ..., https://habr.com/ru/articles/1097380/, fetched 2026-10-26 (unofficial)
[S2] Make yourself a Voxel Engine THIS weekend, https://daymare.net/blogs/voxel-engine-in-a-weekend/, fetched 2026-10-26 (unofficial)
[S3] GitHub - ernerdgn/voxel-engine: A voxel engine written from scratch in ..., https://github.com/ernerdgn/voxel-engine, fetched 2026-10-29 (unofficial)
[S4] A minimal Wolfenstein 3D raycasting engine, https://4rknova.com/blog/2026/07/06/wolfenstein-3d-raycasting, fetched 2026-10-24 (unofficial)
