# Entity Collision & Movement

Also called: Entity Physics, Voxel Entity Physics, Player Movement & Collision, Entity Movement with Sub-stepping and Collision Resolution

## Overview
Entity Collision & Movement is a discrete-time physics solver for voxel-based environments that handles collision detection, response, step-height climbing, and crouching for dynamic hitboxes against static voxel chunks and other dynamic solid hitboxes [S2, unofficial]. It differs from simple grid checks by integrating gravity, ground detection, and axis-aligned movement with wall-sliding collision detection against voxel blocks [S3, unofficial]. It builds on [Voxel Collision Test](5774-voxel-collision-test.md) by extending simple occupancy checks into a full state-based simulation loop [G].

## How it works
**Inputs and Outputs:**
- **Inputs:** Entity position, velocity, input direction (yaw, forward/strafe vectors), time delta ($dt$), and a collection of static voxel chunks and dynamic hitboxes [S2, unofficial].
- **Outputs:** Updated entity position, velocity, and grounded state [S2, unofficial].

**Data Structures:**
The core state is stored in a `Hitbox` struct [S2, unofficial]:
- `position`: `glm::vec3` (current world position)
- `velocity`: `glm::vec3` (current velocity)
- `halfsize`: `glm::vec3` (half-extents of the AABB)
- `grounded`: `bool` (is currently on ground)
- `prevGrounded`: `bool` (state from previous substep)
- `prevVelocity`: `glm::vec3`
- `prevPosition`: `glm::vec3`
- `groundMaterial`: `string`
- `groundVelocity`: `glm::vec3`
- `gravityScale`: `float`
- `stepHeight`: `float` (maximum climbable step height)
- `elasticity`: `float` (coefficient of restitution)
- `type`: `BodyType` enum (STATIC, DYNAMIC, etc.)
- `crouching`: `bool`

An `AABB` struct is used for collision checks [S2, unofficial]:
- `b`: `glm::vec3` (max bounds)
- Methods: `scale(glm::vec3)`, `translated(glm::vec3)`, `intersects(AABB)`, `min()`, `max()`.

Voxel access uses `blockAt(goal, chunks)` [S4, unofficial]:
- Computes chunk indices: $x_{chunk} = \lfloor x / \text{CHUNK\_SIZE} \rfloor$, $z_{chunk} = \lfloor z / \text{CHUNK\_SIZE} \rfloor$ [S4, unofficial].
- Accesses `chunk->blocks[localX][localY][localZ]` [S4, unofficial].
- Returns `255` for air, otherwise the block ID [S4, unofficial].

**Parameters:**
- `E`: Small epsilon for floating-point comparisons and AABB scaling (e.g., $E \times 2.0f$) [S2, unofficial].
- `MOVE_SPEED`: 10 [S4, unofficial].
- `MAX_FIX`: Maximum distance for position correction [S2, unofficial].
- `JUMP_POWER`: 10 [S4, unofficial].
- `GRAVITY`: 30 (downward acceleration per second) [S4, unofficial].
- `ENTITY_HEIGHT`: 2 (vertical size in blocks) [S4, unofficial].
- `substeps`: Number of physics substeps per frame (e.g., 8) [S2, unofficial].
- `maxDeltaTime`: Maximum time step size, set to 0.01 seconds [S3, unofficial].
- `maxVelocityNormPerStep`: Maximum allowed velocity magnitude per sub-step, set to 0.1 [S3, unofficial].
- `epsilon`: Precision threshold for integer checks, set to $10^{-5}$ [S3, unofficial].

**Algorithm Steps:**

1. **Initialization & Sub-stepping Setup:**
   - Clear `groundMaterial`.
   - Set `prevGrounded = grounded`.
   - Reset `grounded = false`.
   - Store `prevVelocity = velocity` and `prevPosition = position` [S2, unofficial].
   - Calculate substep delta: $dt_{sub} = dt / \text{substeps}$ [S2, unofficial].

2. **Gravity & Position Integration:**
   - Apply gravity to velocity: $v_y \leftarrow v_y - \text{GRAVITY} \times dt_{sub} \times \text{gravityScale}$ [S2, unofficial].
   - Update position: $p \leftarrow p + v \times dt_{sub} + \frac{1}{2} g \times \text{gravityScale} \times dt_{sub}^2$ [S2, unofficial].

3. **Ground Detection:**
   - Check block at $p - (0, \text{ENTITY\_HEIGHT}, 0)$ [S4, unofficial].
   - If block $\neq 255$:
     - Set `onGround = true`.
     - If $v_y < 0$, set $v_y = 0$.
     - Snap $p_y = \lceil p_y \rceil$ [S4, unofficial].

4. **Movement Calculation & Axis-Aligned Collision Resolution:**
   - Compute horizontal direction from yaw:
     - $\vec{front} = (\cos(\text{yaw}), 0, \sin(\text{yaw}))$ [S4, unofficial].
     - $\vec{right} = \text{normalize}(\vec{front} \times (0,1,0))$ [S4, unofficial].
   - Calculate change vector:
     - $\Delta \vec{p} = \vec{front} \times (-\text{MOVE\_SPEED} \times dt_{sub}) \times \text{move\_dir}_z + \vec{right} \times (\text{MOVE\_SPEED} \times dt_{sub}) \times \text{move\_dir}_x$ [S4, unofficial].
   - Angle wrapping: if $\text{yaw} \ge 2\pi$ or $\text{yaw} \le -2\pi$, reset to 0 [S1, unofficial].
   - Check collision at $p + \Delta \vec{p}$ [S4, unofficial].
   - If air (255), move there [S4, unofficial].
   - Else, zero out X component of $\Delta \vec{p}$ and recheck [S4, unofficial].
   - Else, zero out Z component of $\Delta \vec{p}$ and recheck [S4, unofficial].

5. **Step-Up Climbing & Ceiling Checks:**
   - If `stepHeight > 0` and $v_y \le 0$:
     - Check for obstacles ahead in movement direction by scaling AABB and iterating x/z coordinates [S2, unofficial].
     - If obstacle is climbable (height $\le$ `stepHeight`), adjust position and velocity to climb [S2, unofficial].
   - If $v_y > 0$, check for ceiling collisions against voxel chunks by iterating x/z coordinates and scaling the AABB [S2, unofficial].

6. **Crouching:**
   - If not crouching or not grounded, return [S2, unofficial].
   - Check ground contact at two axes (x and z) by checking obstacles at $y = p_y - \text{halfsize}_y - E$ [S2, unofficial].
   - If no obstacle found at an axis, reset position and velocity at that axis to initial values and zero velocity [S2, unofficial].
   - Force `grounded = true` after check [S2, unofficial].

7. **Dynamic Hitbox Interaction:**
   - Find nearby solid hitboxes using spatial culling based on movement budget and extents [S2, unofficial].
   - For each dynamic hitbox, call `calcCollisions` with `stepHeight` if previously grounded and `gravityScale > 0` [S2, unofficial].
   - Check ceiling collisions if $v_y > 0$ by iterating x/z coordinates and scaling AABB [S2, unofficial].

8. **Edge Cases & Merging:**
   - If $p_y < 0$, teleport to $y=100$ and reset $v_y = 0$ [S4, unofficial].
   - Floating-point precision issues are mitigated by using epsilon $E$ in AABB scaling and distance checks [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def update_entity(entity, chunks, dynamic_hitboxes, dt, yaw, input_dir):
    # Constants from sources
    GRAVITY = 30.0
    MOVE_SPEED = 10.0
    ENTITY_HEIGHT = 2.0
    MAX_FIX = 0.1  # Not given exact value in sources [G]
    E = 1e-5       # Precision threshold [S3, unofficial]
    substeps = 8
    max_delta_time = 0.01
    max_velocity_norm_per_step = 0.1
    
    dt_sub = dt / substeps
    entity.prev_ground = entity.grounded
    entity.grounded = False
    entity.prev_velocity = entity.velocity
    entity.prev_position = entity.position
    
    remaining_dt = dt
    while remaining_dt > 0:
        local_dt = min(remaining_dt, max_delta_time)
        
        # Velocity scaling to respect max_velocity_norm_per_step [S3, unofficial]
        vel_norm = math.sqrt(entity.velocity.x**2 + entity.velocity.y**2 + entity.velocity.z**2)
        naive_step_vel_norm = remaining_dt * vel_norm
        if naive_step_vel_norm > max_velocity_norm_per_step:
            local_dt = remaining_dt * (max_velocity_norm_per_step / naive_step_vel_norm)
            
        # Apply gravity [S2, unofficial]
        entity.velocity.y -= GRAVITY * local_dt * entity.gravity_scale
        entity.position = (
            entity.position.x + entity.velocity.x * local_dt,
            entity.position.y + entity.velocity.y * local_dt + 0.5 * GRAVITY * entity.gravity_scale * local_dt**2,
            entity.position.z + entity.velocity.z * local_dt
        )
        
        # Ground detection [S4, unofficial]
        check_pos = (entity.position.x, entity.position.y - ENTITY_HEIGHT, entity.position.z)
        if block_at(check_pos, chunks) != 255:
            entity.grounded = True
            if entity.velocity.y < 0:
                entity.velocity.y = 0
            entity.position = (entity.position.x, math.ceil(entity.position.y), entity.position.z)
            
        # Movement calculation [S4, unofficial]
        front = (math.cos(yaw), 0, math.sin(yaw))
        right = normalize(cross(front, (0, 1, 0)))
        move_dir = normalize((input_dir.x, 0, input_dir.z))
        change = (
            front[0] * (-MOVE_SPEED * local_dt) * move_dir.z + right[0] * (MOVE_SPEED * local_dt) * move_dir.x,
            0,
            front[2] * (-MOVE_SPEED * local_dt) * move_dir.z + right[2] * (MOVE_SPEED * local_dt) * move_dir.x
        )
        
        # Axis-aligned collision resolution [S4, unofficial]
        goal = (entity.position.x + change[0], entity.position.y, entity.position.z + change[2])
        if block_at(goal, chunks) == 255:
            entity.position = goal
        else:
            change_x = change[0]
            change[0] = 0
            goal = (entity.position.x + change[0], entity.position.y, entity.position.z + change[2])
            if block_at(goal, chunks) == 255:
                entity.position = goal
            else:
                change[2] = 0
                goal = (entity.position.x + change[0], entity.position.y, entity.position.z + change[2])
                if block_at(goal, chunks) == 255:
                    entity.position = goal
                    
        # Step-up climbing [S2, unofficial]
        if entity.step_height > 0 and entity.velocity.y <= 0:
            # Sources do not provide full AABB iteration logic for step-up climbing [G]
            pass
            
        # Crouching [S2, unofficial]
        if entity.crouching and entity.grounded:
            # Sources do not provide full AABB check logic for crouching at y = pos.y - halfsize.y - E [G]
            pass
            entity.grounded = True
            
        remaining_dt -= local_dt
        
    # Edge cases [S4, unofficial]
    if entity.position.y < 0:
        entity.position = (entity.position.x, 100, entity.position.z)
        entity.velocity = (entity.velocity.x, 0, entity.velocity.z)
        
    # Angle wrapping [S1, unofficial]
    if yaw >= 2 * math.pi or yaw <= -2 * math.pi:
        yaw = 0
        
    return entity.position, entity.velocity, entity.grounded

def block_at(pos, chunks):
    x_chunk = math.floor(pos[0] / CHUNK_SIZE)
    z_chunk = math.floor(pos[2] / CHUNK_SIZE)
    chunk = get_chunk(x_chunk, z_chunk, chunks)
    if chunk is None: return 255
    local_x = pos[0] - x_chunk * CHUNK_SIZE
    local_y = pos[1]
    local_z = pos[2] - z_chunk * CHUNK_SIZE
    return chunk.blocks[local_x][local_y][local_z]

def normalize(v):
    length = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    if length == 0: return (0, 0, 0)
    return (v[0]/length, v[1]/length, v[2]/length)

def cross(a, b):
    return (
        a[1]*b[2] - a[2]*b[1],
        a[2]*b[0] - a[0]*b[2],
        a[0]*b[1] - a[1]*b[0]
    )
```

## Subtopics

### [Voxel Physics Engine](19653-voxel-physics-engine.md)
The Voxel Physics Engine is a rigid-body simulation system designed for voxel-based game environments, handling collisions between entities and solid or liquid voxel terrain. It distinguishes itself from general-purpose physics engines by relying on abstracted boolean test functions for voxel properties rather than a pre-computed spatial structure, making it lightweight and engine-agnostic. Unlike its predecessor `voxel-physical`, it implements more physically accurate behaviors including semi-implicit Euler integration, axis-dependent friction, and buoyancy. [Voxel Physics Engine](19653-voxel-physics-engine.md)

### Player Movement & Physics Integration
Computes player velocity and position updates using directional acceleration, ground/air friction, gravity, and discrete time steps [S5, unofficial]. It calculates forward and right vectors from yaw, builds a wish direction from WASD/Arrow keys, and normalizes it. Acceleration is scaled by `ACCEL` when grounded and `AIR_CONTROL` in air, with deceleration applied when stationary. Gravity is applied per frame, and jump impulses are handled via `JUMP_FORCE` [S5, unofficial]. Parameters include `WALK_SPEED = 6.0`, `RUN_SPEED = 12.0`, `GRAVITY = 30.0`, `JUMP_FORCE = 14.0`, `ACCEL = 40.0`, `DECEL = 30.0`, and `AIR_CONTROL = 0.15` [S5, unofficial].

## Operating characteristics
- **Time Complexity:** Spatial culling reduces collision checks from $O(N \times M)$ to $O(N \times K)$ where $K$ is the number of nearby solids per hitbox [S2, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Performance:** Depends on the number of substeps and nearby solids. Higher substeps improve stability but increase CPU cost [S2, unofficial].
- **Strengths:** Handles complex interactions like step-climbing, crouching, and dynamic hitboxes. Stable with discrete substeps [S2, unofficial].
- **Limitations:** Discrete nature can cause tunneling at high speeds. Requires careful tuning of `stepHeight` and `MAX_FIX` [S2, unofficial].
- **When to Use:** Not given in the sources.
- **When Not to Use:** Not given in the sources.

## Background
Originating from voxel game engines like VoxelCore and VoxelEngine, this method implements a discrete-time physics solver for voxel environments [S2, S4, unofficial]. It evolved from simple raycasting prototypes to full state-based simulation with substeps to handle gravity, ground detection, and collision response [S1, S2, S3, unofficial].

## Sources
[S1] Raycasting Prototype with DDA, https://github.com/GPMLAD/raycasting, fetched 2026-10-25 (unofficial)
[S2] MihailRis/voxelcore, https://github.com/MihailRis/voxelcore, fetched 2026-10-26 (unofficial)
[S3] aresrpg/aresrpg-engine, https://github.com/aresrpg/aresrpg-engine, fetched 2026-10-27 (unofficial)
[S4] GitHub Repo for VoxelEngine, https://github.com/TheSlabby/VoxelEngine, fetched 2026-10-29 (unofficial)
[S5] RainArcanjo/CubWrldJS, https://github.com/RainArcanjo/CubWrldJS, fetched 2026-10-27 (unofficial)
