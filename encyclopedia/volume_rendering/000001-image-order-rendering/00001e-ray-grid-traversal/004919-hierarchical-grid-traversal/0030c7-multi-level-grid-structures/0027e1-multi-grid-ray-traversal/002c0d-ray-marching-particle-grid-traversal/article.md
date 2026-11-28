# Ray Marching Particle Grid Traversal

Also called: Hybrid Ray Marching Particle Grid Traversal, Row-Range Particle Lookup

## Overview
Ray Marching Particle Grid Traversal is a hybrid ray-grid traversal technique that traverses a voxel grid to find particle hits, using a low-resolution grid for culling and a high-resolution grid for precise hit testing [S1, unofficial]. It optimizes particle sphere tests by reading contiguous rows of cells as single ranges of sorted particles, reducing memory loads [S2, unofficial]. Unlike standard uniform grid traversal which may visit empty voxels, this method skips large portions of the scene by switching between resolution levels based on occupancy flags [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $pos_0$ and direction $rayDirection$ (normalized) [S1, unofficial].
- Particle position buffer $particlesPositionBuffer$: array<vec4f>, stores particle positions [S1, unofficial].
- Particle index buffer $particlesIndicesBuffer$: array<i32>, stores particle IDs [S1, unofficial].
- Low-resolution occupancy grid $gridBuffer$: array<i32>, stores occupancy flags (1 or 0) for low-res grid cells [S1, unofficial].
- Scene resolution $resolution$ and inner radius $INNER\_RADIUS$ [S1, unofficial].

**Outputs:**
- Hit color, hit position, hit velocity, and boolean success flag [S1, unofficial].

**Data Structures:**
1. $particlesPositionBuffer$: array<vec4f>, stores particle positions [S1, unofficial].
2. $particlesIndicesBuffer$: array<i32>, stores particle IDs [S1, unofficial].
3. $gridBuffer$: array<i32>, stores occupancy flags (1 or 0) for low-res grid cells [S1, unofficial].

**Parameters:**
- $MAX\_ITERATIONS = 400$ [S1, unofficial].
- $GRID\_RATIO = 8$ [S1, unofficial].
- $RADIUS = 0.19$ (Squared radius threshold for hit check) [S1, unofficial].
- $EPSILON = 1.e-10$ [S1, unofficial].
- $acne = 0.05$ [S1, unofficial].

**Algorithm Steps:**

1. **Initialization:**
   Calculate the initial step direction and side distances for slab crossing [S1, unofficial].
   Let $mapPos = floor(pos_0)$.
   Let $deltaDist = rayDirection / |rayDirection|$ (normalized direction components) [G].
   Let $rayStep = 1.0$ (voxel step size) [G].
   $$sideDist = (rayStep \cdot (mapPos - pos_0 + 0.5) + 0.5) \cdot deltaDist$$ [S1, unofficial].
   Set $highResolution = true$.
   Set $d = \infty$ (closest hit distance).
   Set $index = -1000$ (hit particle ID).

2. **Ray Marching Loop:**
   For $i$ from 0 to $MAX\_ITERATIONS$:

   a. **Determine Step Direction:**
      Calculate the sign of the step for each axis using the `step` function (returns 1 if $a < b$, else 0) [S1, unofficial].
      $$stepForward = step(sideDist.xyz, sideDist.yxy) \cdot step(sideDist.xyz, sideDist.zzx)$$ [S1, unofficial].
      *Note: This logic determines the next voxel to enter based on which boundary is closest.*

   b. **Update Position and Distances:**
      $$sideDist += stepForward \cdot deltaDist$$ [S1, unofficial].
      $$pos += stepForward \cdot rayStep$$ [S1, unofficial].
      $$mapPos = floor(pos)$$ [S1, unofficial].

   c. **Calculate Distance to Next Boundary:**
      $$t = \min(sideDist.x, \min(sideDist.y, sideDist.z))$$ [S1, unofficial].

   d. **Border Check:**
      Check if the ray is outside the inner volume [S1, unofficial].
      $$borders = length(pos.yz / resolution - vec2(0.5)) > INNER\_RADIUS$$ [S1, unofficial].
      Also check x-bound conditions (implementation specific, typically checking if $pos.x$ is out of bounds) [S1, unofficial].
      If $borders$ is true, continue to next iteration (skip hit testing) [S1, unofficial].

   e. **Grid Level Selection:**
      Calculate the low-resolution grid index [S1, unofficial]:
      $$lowGrid = mapPos / GRID\_RATIO \quad (\text{integer division})$$ [S1, unofficial].
      $$indexGrid = lowGrid.x + lowRes \cdot lowGrid.y + lowRes^2 \cdot lowGrid.z$$ [S1, unofficial].
      where $lowRes = resolution / GRID\_RATIO$.

   f. **Occupancy Check:**
      $$resolution = (gridBuffer[indexGrid] == 1)$$ [S1, unofficial].
      
      *Case 1: High-Res Mode, No Occupancy*
      If $highResolution$ is true and $resolution$ is false:
      Switch to low-res mode [S1, unofficial].
      $$pos0\_l = (pos0\_h + t \cdot rayDirection) / GRID\_RATIO$$ [S1, unofficial].
      $$pos = pos0\_l$$ [S1, unofficial].
      $$mapPos = floor(pos)$$ [S1, unofficial].
      Recalculate $sideDist$ for the low-res grid [S1, unofficial].
      Set $highResolution = false$ [S1, unofficial].
      Continue to next iteration.

      *Case 2: High-Res Mode, Occupancy Found*
      If $highResolution$ is true and $resolution$ is true:
      Perform particle hit test in the current high-res voxel [S1, unofficial].
      Calculate linear index in the particle index buffer [S1, unofficial]:
      $$index1D = mapPos.x + res \cdot mapPos.y + res^2 \cdot mapPos.z$$ [S1, unofficial].
      where $res$ is the high-resolution grid dimension.
      
      Iterate $z$ from 0 to 7 (assuming up to 8 particles per voxel slot):
      $$particleId = 8 \cdot index1D + z$$ [S1, unofficial].
      $$voxelData = particlesIndicesBuffer[particleId]$$ [S1, unofficial].
      
      If $voxelData == 0$, break loop (no more particles in this slot) [S1, unofficial].
      
      Retrieve particle center [S1, unofficial]:
      $$center = particlesPositionBuffer[voxelData].rgb$$ [S1, unofficial].
      
      Check if particle is within valid bounds (e.g., $center.x > 0.1 \cdot resolution$) [S1, unofficial].
      
      Perform sphere intersection test [S1, unofficial]:
      Calculate distance $l$ from ray to particle center.
      If $checkHit(initPos, center, rayDirection)$ returns true AND $l < d$:
      $$index = voxelData$$ [S1, unofficial].
      $$d = l$$ [S1, unofficial].

      *Case 3: Low-Res Mode, Occupancy Found*
      If $highResolution$ is false and $resolution$ is true:
      Switch back to high-res mode [S1, unofficial].
      $$pos0\_h = GRID\_RATIO \cdot (pos0\_l + 0.9999 \cdot t \cdot rayDirection)$$ [S1, unofficial].
      $$pos = pos0\_h$$ [S1, unofficial].
      $$mapPos = floor(pos)$$ [S1, unofficial].
      Recalculate $sideDist$ for the high-res grid [S1, unofficial].
      Set $highResolution = true$ [S1, unofficial].
      Continue to next iteration.

   g. **Update State:**
      $$highResolution = resolution$$ [S1, unofficial].

3. **Result:**
   If $index > -1000$:
   Set $hitColor$, $hitPosition$, $hitVelocity$ from $particlesPositionBuffer[index]$ and other buffers [S1, unofficial].
   Return true.
   Else:
   Return false.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_particle_grid(pos0, ray_direction, particles_position_buffer, particles_indices_buffer, grid_buffer, resolution, inner_radius):
    MAX_ITERATIONS = 400
    GRID_RATIO = 8
    RADIUS = 0.19
    EPSILON = 1e-10
    acne = 0.05
    
    pos = pos0
    pos0_h = pos0
    pos0_l = pos0
    map_pos = floor(pos)
    delta_dist = ray_direction / length(ray_direction)
    ray_step = 1.0
    side_dist = (ray_step * (map_pos - pos + 0.5) + 0.5) * delta_dist
    high_resolution = True
    d = float('inf')
    index = -1000
    low_res = resolution // GRID_RATIO
    
    for i in range(MAX_ITERATIONS):
        # stepForward = step(sideDist.xyz, sideDist.yxy) * step(sideDist.xyz, sideDist.zzx)
        step_x = step(side_dist.x, side_dist.y) * step(side_dist.x, side_dist.z)
        step_y = step(side_dist.y, side_dist.z) * step(side_dist.y, side_dist.x)
        step_z = step(side_dist.z, side_dist.x) * step(side_dist.z, side_dist.y)
        step_forward = vec3(step_x, step_y, step_z)
        
        side_dist += step_forward * delta_dist
        pos += step_forward * ray_step
        map_pos = floor(pos)
        
        t = min(side_dist.x, min(side_dist.y, side_dist.z))
        
        border_check_yz = length(pos.yz / resolution - vec2(0.5)) > inner_radius
        border_check_x = (pos.x < 0.0) or (pos.x > resolution)
        
        if border_check_yz or border_check_x:
            continue
            
        low_grid = map_pos // GRID_RATIO
        index_grid = low_grid.x + low_res * low_grid.y + (low_res * low_res) * low_grid.z
        resolution_flag = (grid_buffer[index_grid] == 1)
        
        if high_resolution:
            if not resolution_flag:
                pos0_l = (pos0_h + t * ray_direction) / GRID_RATIO
                pos = pos0_l
                map_pos = floor(pos)
                side_dist = (ray_step * (map_pos - pos + 0.5) + 0.5) * delta_dist
                high_resolution = False
                continue
            else:
                res = resolution
                index1d = map_pos.x + res * map_pos.y + (res * res) * map_pos.z
                
                for z in range(8):
                    particle_id = 8 * index1d + z
                    voxel_data = particles_indices_buffer[particle_id]
                    
                    if voxel_data == 0:
                        break
                    
                    center = particles_position_buffer[voxel_data].xyz
                    
                    if center.x > 0.1 * resolution:
                        l = distance_from_ray_to_sphere(pos0, ray_direction, center, particles_position_buffer[voxel_data].w)
                        
                        if l < d:
                            index = voxel_data
                            d = l
                
                if index > -1000:
                    return True, index, d
        else:
            if resolution_flag:
                pos0_h = GRID_RATIO * (pos0_l + 0.9999 * t * ray_direction)
                pos = pos0_h
                map_pos = floor(pos)
                side_dist = (ray_step * (map_pos - pos + 0.5) + 0.5) * delta_dist
                high_resolution = True
                continue
        
        high_resolution = resolution_flag
        
    return False, -1, float('inf')

def distance_from_ray_to_sphere(ray_origin, ray_direction, sphere_center, radius):
    oc = ray_origin - sphere_center
    t_ca = dot(oc, ray_direction)
    d2 = dot(oc, oc) - t_ca * t_ca
    if d2 < radius * radius:
        return sqrt(d2)
    return float('inf')

def step(edge, x):
    return 1.0 if edge < x else 0.0
```

**Edge Cases and Pitfalls:**
- **Squared Radius:** The `checkHit` function uses squared radius ($RADIUS$) to avoid square root calculations. The distance check must be squared as well ($d = l^2 - t_{CA}^2$) [S1, unofficial].
- **Empty Voxels:** If $particlesIndicesBuffer$ contains 0s, the loop breaks early, which is efficient but requires careful initialization of the buffer [S1, unofficial].

## Subtopics

### [2D Projection 1D Lookup Particle Cache](8958-2d-projection-1d-lookup-particle-cache.md)
The 2D Projection 1D Lookup Particle Cache is a spatial data structure that optimizes field value calculations in volume rendering by iterating only over particles within a fixed cutoff radius $R_c$ from a sample point. It solves the inefficiency of retrieving influencing particles for isosurface extraction (e.g., Marching Slices) by combining a 2D projection with a 1D search along the z-axis. Unlike 3D lookup grid approaches (such as those by Wyvill et al., Müller et al., and Triquet) which return approximately 6.5 times more particles due to cubic bounding volumes requiring expensive floating-point distance culling, this method returns exactly the particles within the spherical radius $R_c$ using only integer comparisons.

### [Particle Rendering via Grid Bucketing](13245-particle-rendering-via-grid-bucketing.md)
Particle Rendering via Grid Bucketing renders particles as spheres by mapping them to a spatial grid to accelerate ray-sphere intersection tests. It differs from [Ray Marching Particle Grid Traversal](5182-ray-marching-particle-grid-traversal.md) by explicitly iterating over particle lists stored in grid buckets rather than marching through a voxel grid to find hits, and from [2D Projection 1D Lookup Particle Cache](8958-2d-projection-1d-lookup-particle-cache.md) by relying on 3D spatial hashing rather than 2D screen-space caching.

## Operating characteristics
- **Measured Performance:** Reading contiguous rows of cells as single ranges of sorted particles reduced memory loads by 14% compared to individual cell access [S2, unofficial]. The method achieved a rendering time of 4.1 ms for a lightbox scene, compared to 4.7 ms for a wall-drawing approach [S2, unofficial].
- **Strengths:** Efficient culling of empty space via low-resolution occupancy flags; reduced memory bandwidth usage due to contiguous memory access patterns; avoids square root calculations for sphere tests by using squared radius thresholds [S1, unofficial][S2, unofficial].
- **Limitations:** Performance degrades if the scene is densely packed with many particles per voxel, as the inner loop iterates up to 8 particles per cell [S1, unofficial]. Switching between resolution levels adds computational overhead [S1, unofficial].

## Background
Ray Marching Particle Grid Traversal was developed as part of a WebGPU-based voxel cone tracing and particle rendering system. The technique was documented in an unofficial GitHub repository by Hector Arellano Dev, which provides a complete implementation of the algorithm in WebGPU shaders [S1, unofficial]. The optimization of reading contiguous rows of cells as ranges was further discussed in a pull request by Siroko for the "kansei" project, which highlighted performance gains in ray-traced lightbox rendering [S2, unofficial]. The method is distinct from traditional grid-based volume rendering as it focuses on discrete particle spheres rather than continuous scalar fields [S1, unofficial].

## Sources
[S1] HectorArellanoDev/WebGPUVoxelConeTracing, https://github.com/HectorArellanoDev/WebGPUVoxelConeTracing, fetched 2026-10-26 (unofficial)
[S2] voxel-gi-particles: ray-traced lightbox about twice as fast, same image, https://github.com/Siroko/kansei/pull/114, fetched 2026-10-26 (unofficial)
