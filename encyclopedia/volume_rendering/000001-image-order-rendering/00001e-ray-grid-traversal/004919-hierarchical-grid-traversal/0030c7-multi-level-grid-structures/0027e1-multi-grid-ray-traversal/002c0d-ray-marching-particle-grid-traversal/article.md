# Ray Marching Particle Grid Traversal

Also called: Hybrid Ray Marching Particle Grid Traversal, Row-Range Particle Lookup

## Overview
Ray Marching Particle Grid Traversal is a hybrid ray-grid traversal technique that uses a low-resolution grid for coarse culling and a high-resolution grid for precise particle hit testing [S1, unofficial]. It solves the inefficiency of checking every particle in a voxel by leveraging spatial coherence: the low-res grid quickly identifies occupied regions, while the high-res grid stores sorted particle indices to allow contiguous memory access during sphere intersection tests [S1, unofficial][S2, unofficial]. Unlike standard uniform grid traversal which may visit empty voxels, this method skips large portions of the scene by switching between resolution levels based on occupancy flags [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $pos_0$ and direction $rayDirection$ (normalized).
- Particle position buffer $particlesPositionBuffer$: array of $vec4f$, where $xyz$ is position and $w$ is radius [S2, unofficial].
- Particle index buffer $particlesIndicesBuffer$: array of $i32$, storing particle IDs mapped to high-resolution voxel slots [S1, unofficial].
- Low-resolution occupancy grid $gridBuffer$: array of $i32$, storing 1 (occupied) or 0 (empty) [S1, unofficial].
- Scene resolution $resolution$ and inner radius $INNER\_RADIUS$ [S1, unofficial].

**Outputs:**
- Hit color, hit position, hit velocity, and boolean success flag [S1, unofficial].

**Data Structures:**
1.  **$particlesPositionBuffer$**: $N \times 4$ floats. $xyz$ = center, $w$ = radius [S2, unofficial].
2.  **$particlesIndicesBuffer$**: $M$ integers. Maps high-res voxel coordinates $(x,y,z)$ to a linear index $i = x + res \cdot y + res^2 \cdot z$. Each voxel slot stores up to 8 particle IDs (batched) [S1, unofficial].
3.  **$gridBuffer$**: $L^3$ integers. $L = N / GRID\_RATIO$. Value is 1 if the corresponding low-res voxel contains any particles, 0 otherwise [S1, unofficial].

**Parameters:**
- $MAX\_ITERATIONS = 400$ [S1, unofficial].
- $GRID\_RATIO = 8$ [S1, unofficial].
- $RADIUS = 0.19$ (Squared radius threshold for hit check) [S1, unofficial].
- $EPSILON = 1.e-10$ [S1, unofficial].
- $acne = 0.05$ [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    Calculate the initial step direction and side distances for slab crossing.
    Let $mapPos = floor(pos_0)$.
    Let $deltaDist = rayDirection / |rayDirection|$ (normalized direction components).
    Let $rayStep$ be the initial step size (often 1.0 in voxel units or derived from resolution).
    $$sideDist = (rayStep \cdot (mapPos - pos_0 + 0.5) + 0.5) \cdot deltaDist$$
    Set $highResolution = true$.
    Set $d = \infty$ (closest hit distance).
    Set $index = -1000$ (hit particle ID).

2.  **Ray Marching Loop:**
    For $i$ from 0 to $MAX\_ITERATIONS$:

    a.  **Determine Step Direction:**
        Calculate the sign of the step for each axis using the `step` function (returns 1 if $a < b$, else 0).
        $$stepForward = step(sideDist.xyz, sideDist.yxy) \cdot step(sideDist.xyz, sideDist.zzx)$$
        *Note: This logic determines the next voxel to enter based on which boundary is closest.*

    b.  **Update Position and Distances:**
        $$sideDist += stepForward \cdot deltaDist$$
        $$pos += stepForward \cdot rayStep$$
        $$mapPos = floor(pos)$$

    c.  **Calculate Distance to Next Boundary:**
        $$t = \min(sideDist.x, \min(sideDist.y, sideDist.z))$$

    d.  **Border Check:**
        Check if the ray is outside the inner volume.
        $$borders = length(pos.yz / resolution - vec2(0.5)) > INNER\_RADIUS$$
        Also check x-bound conditions (implementation specific, typically checking if $pos.x$ is out of bounds).
        If $borders$ is true, continue to next iteration (skip hit testing) [S1, unofficial].

    e.  **Grid Level Selection:**
        Calculate the low-resolution grid index:
        $$lowGrid = mapPos / GRID\_RATIO \quad (\text{integer division})$$
        $$indexGrid = lowGrid.x + lowRes \cdot lowGrid.y + lowRes^2 \cdot lowGrid.z$$
        where $lowRes = N / GRID\_RATIO$.

    f.  **Occupancy Check:**
        $$resolution = (gridBuffer[indexGrid] == 1)$$
        
        *Case 1: High-Res Mode, No Occupancy*
        If $highResolution$ is true and $resolution$ is false:
        Switch to low-res mode.
        $$pos0\_l = (pos0\_h + t \cdot rayDirection) / GRID\_RATIO$$
        $$pos = pos0\_l$$
        $$mapPos = floor(pos)$$
        Recalculate $sideDist$ for the low-res grid.
        Set $highResolution = false$.
        Continue to next iteration.

        *Case 2: High-Res Mode, Occupancy Found*
        If $highResolution$ is true and $resolution$ is true:
        Perform particle hit test in the current high-res voxel.
        Calculate linear index in the particle index buffer:
        $$index1D = mapPos.x + res \cdot mapPos.y + res^2 \cdot mapPos.z$$
        where $res$ is the high-resolution grid dimension.
        
        Iterate $z$ from 0 to 7 (assuming up to 8 particles per voxel slot):
        $$particleId = 8 \cdot index1D + z$$
        $$voxelData = particlesIndicesBuffer[particleId]$$
        
        If $voxelData == 0$, break loop (no more particles in this slot) [S1, unofficial].
        
        Retrieve particle center:
        $$center = particlesPositionBuffer[voxelData].xyz$$
        
        Check if particle is within valid bounds (e.g., $center.x > 0.1 \cdot resolution$) [S1, unofficial].
        
        Perform sphere intersection test:
        Calculate distance $l$ from ray to particle center.
        If $checkHit(pos_0, center, rayDirection)$ returns true AND $l < d$:
        $$index = voxelData$$
        $$d = l$$

        *Case 3: Low-Res Mode, Occupancy Found*
        If $highResolution$ is false and $resolution$ is true:
        Switch back to high-res mode.
        $$pos0\_h = GRID\_RATIO \cdot (pos0\_l + 0.9999 \cdot t \cdot rayDirection)$$
        $$pos = pos0\_h$$
        $$mapPos = floor(pos)$$
        Recalculate $sideDist$ for the high-res grid.
        Set $highResolution = true$.
        Continue to next iteration.

    g.  **Update State:**
        $$highResolution = resolution$$

3.  **Result:**
    If $index > -1000$:
    Set $hitColor$, $hitPosition$, $hitVelocity$ from $particlesPositionBuffer[index]$ and other buffers.
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
    
    # Initial setup
    pos = pos0
    map_pos = floor(pos)
    delta_dist = ray_direction / length(ray_direction) # Normalize
    ray_step = 1.0 # Assuming unit voxel size for step calculation
    
    # Calculate initial side distances
    # sideDist = (rayStep * (mapPos - pos + 0.5) + 0.5) * deltaDist
    side_dist = (ray_step * (map_pos - pos + 0.5) + 0.5) * delta_dist
    
    high_resolution = True
    d = float('inf')
    index = -1000
    
    # Low-res grid dimensions
    low_res = resolution // GRID_RATIO
    
    for i in range(MAX_ITERATIONS):
        # Determine step direction
        # step(a, b) returns 1.0 if a < b else 0.0
        step_forward = step(side_dist.x, side_dist.y) * step(side_dist.x, side_dist.z) * \
                       step(side_dist.y, side_dist.x) * step(side_dist.y, side_dist.z) * \
                       step(side_dist.z, side_dist.x) * step(side_dist.z, side_dist.y)
        
        # Update side distances and position
        side_dist += step_forward * delta_dist
        pos += step_forward * ray_step
        map_pos = floor(pos)
        
        # Distance to next voxel boundary
        t = min(side_dist.x, min(side_dist.y, side_dist.z))
        
        # Border conditions
        # Check yz plane distance from center relative to resolution
        border_check_yz = length(pos.yz / resolution - vec2(0.5)) > inner_radius
        # Check x bounds (example logic, exact bounds depend on scene)
        border_check_x = (pos.x < 0.0) or (pos.x > resolution)
        
        if border_check_yz or border_check_x:
            continue
            
        # Determine low-res grid index
        low_grid = map_pos // GRID_RATIO
        index_grid = low_grid.x + low_res * low_grid.y + (low_res * low_res) * low_grid.z
        
        # Check occupancy
        resolution_flag = (grid_buffer[index_grid] == 1)
        
        if high_resolution:
            if not resolution_flag:
                # Switch to low-res mode
                pos0_l = (pos0 + t * ray_direction) / GRID_RATIO
                pos = pos0_l
                map_pos = floor(pos)
                # Recalculate sideDist for low-res grid
                side_dist = (ray_step * (map_pos - pos + 0.5) + 0.5) * delta_dist
                high_resolution = False
                continue
            else:
                # High-res hit test
                res = resolution
                index1d = map_pos.x + res * map_pos.y + (res * res) * map_pos.z
                
                hit_found = False
                for z in range(8):
                    particle_id = 8 * index1d + z
                    voxel_data = particles_indices_buffer[particle_id]
                    
                    if voxel_data == 0:
                        break
                    
                    center = particles_position_buffer[voxel_data].xyz
                    
                    # Basic bounds check
                    if center.x > 0.1 * resolution:
                        # Sphere intersection check
                        # checkHit function calculates distance from ray to sphere center
                        # Returns true if intersection occurs within radius
                        l = distance_from_ray_to_sphere(pos0, ray_direction, center, particles_position_buffer[voxel_data].w)
                        
                        if l < d:
                            index = voxel_data
                            d = l
                            hit_found = True
                
                if hit_found:
                    return True, index, d
        else:
            if resolution_flag:
                # Switch back to high-res mode
                pos0_h = GRID_RATIO * (pos0 + 0.9999 * t * ray_direction)
                pos = pos0_h
                map_pos = floor(pos)
                # Recalculate sideDist for high-res grid
                side_dist = (ray_step * (map_pos - pos + 0.5) + 0.5) * delta_dist
                high_resolution = True
                continue
        
        high_resolution = resolution_flag
        
    return False, -1, float('inf')

def distance_from_ray_to_sphere(ray_origin, ray_direction, sphere_center, radius):
    # Vector from ray origin to sphere center
    oc = ray_origin - sphere_center
    # Project oc onto ray direction
    t_ca = dot(oc, ray_direction)
    # Squared distance from ray to center
    d2 = dot(oc, oc) - t_ca * t_ca
    # Check if distance is less than squared radius
    if d2 < radius * radius:
        return sqrt(d2)
    return float('inf')
```

**Edge Cases and Pitfalls:**
- **Precision:** The check for borders uses floating-point comparisons. Small errors in $pos$ calculation can cause jitter or missed hits near boundaries [S1, unofficial].
- **Squared Radius:** The `checkHit` function uses squared radius ($RADIUS$) to avoid square root calculations. The distance check must be squared as well ($d = l^2 - t_{CA}^2$) [S1, unofficial].
- **Empty Voxels:** If $particlesIndicesBuffer$ contains 0s, the loop breaks early, which is efficient but requires careful initialization of the buffer [S1, unofficial].
- **Resolution Mismatch:** The $GRID\_RATIO$ must be an integer divisor of the high-resolution grid dimensions to ensure correct mapping [S1, unofficial].

## Subtopics

### [2D Projection 1D Lookup Particle Cache](8958-2d-projection-1d-lookup-particle-cache.md)
The 2D Projection 1D Lookup Particle Cache is a spatial data structure that optimizes field value calculations in volume rendering by iterating only over particles within a fixed cutoff radius $R_c$ from a sample point. It solves the inefficiency of retrieving influencing particles for isosurface extraction (e.g., Marching Slices) by combining a 2D projection with a 1D search along the z-axis. Unlike 3D lookup grid approaches (such as those by Wyvill et al., Müller et al., and Triquet) which return approximately 6.5 times more particles due to cubic bounding volumes requiring expensive floating-point distance culling, this method returns exactly the particles within the spherical radius $R_c$ using only integer comparisons.

### Particle Rendering via Grid Bucketing
Particle Rendering via Grid Bucketing renders a particle (sphere) by sampling grid cells within the particle's bounding box and checking if grid points fall within the particle's radius [S3, unofficial]. It calculates the grid index of the particle center, retrieves the update pattern (bucket) for that index, and iterates over each point in the bucket. If the squared distance from the particle center to the point is less than or equal to the squared radius, the point is added to the render list with the particle's object ID, display parameters, position, normal, dithered color, and clipping behavior [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of ray steps, but with a significantly reduced constant factor due to culling. The low-res grid allows skipping large empty regions. The high-res grid access is $O(1)$ per voxel, and the particle loop is bounded by the maximum number of particles per voxel (8) [S1, unofficial][S2, unofficial].
- **Memory Complexity:** $O(V_{low} + V_{high} \cdot P_{max})$ where $V_{low}$ is the number of low-res voxels, $V_{high}$ is the number of high-res voxels, and $P_{max}$ is the max particles per voxel (8) [S1, unofficial].
- **Measured Performance:** In a test case, using row-range particle lookup (reading contiguous rows of cells as single ranges) reduced memory loads by 14% compared to individual cell access [S2, unofficial]. The method achieved a rendering time of 4.1 ms for a lightbox scene, compared to 4.7 ms for a wall-drawing approach [S2, unofficial].
- **Strengths:** Efficient culling of empty space; reduced memory bandwidth usage due to contiguous memory access patterns; avoids square root calculations for sphere tests [S1, unofficial][S2, unofficial].
- **Limitations:** Requires pre-computation of the occupancy grid and particle index buffer; performance degrades if the scene is densely packed (many particles per voxel); switching between resolution levels adds overhead [S1, unofficial].
- **When to Use:** Suitable for particle-based volume rendering where particles are sparse and spatially coherent. Ideal for real-time applications on GPUs where memory bandwidth is a bottleneck [S1, unofficial][S2, unofficial].
- **When Not to Use:** Not suitable for dense, continuous volume data (e.g., medical CT scans) where trilinear interpolation is required. Not ideal for scenes with highly irregular particle distributions that do not align well with the grid [S1, unofficial].

## Background
Ray Marching Particle Grid Traversal was developed as part of a WebGPU-based voxel cone tracing and particle rendering system. The technique was documented in an unofficial GitHub repository by Hector Arellano Dev, which provides a complete implementation of the algorithm in WebGPU shaders [S1, unofficial]. The optimization of reading contiguous rows of cells as ranges was further discussed in a pull request by Siroko for the "kansei" project, which highlighted performance gains in ray-traced lightbox rendering [S2, unofficial]. The method is distinct from traditional grid-based volume rendering as it focuses on discrete particle spheres rather than continuous scalar fields [S1, unofficial].

## Sources
[S1] HectorArellanoDev/WebGPUVoxelConeTracing, https://github.com/HectorArellanoDev/WebGPUVoxelConeTracing, fetched 2026-10-26 (unofficial)
[S2] voxel-gi-particles: ray-traced lightbox about twice as fast, same image, https://github.com/Siroko/kansei/pull/114, fetched 2026-10-26 (unofficial)
[S3] quertzuiop/Volumetric-display, https://github.com/quertzuiop/Volumetric-display, fetched 2026-10-26 (unofficial)
