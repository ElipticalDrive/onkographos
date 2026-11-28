# PBF Displacement Calculation with Grid-Based Neighbor Search
## Overview
PBF Displacement Calculation with Grid-Based Neighbor Search is a particle-based fluid simulation step that calculates particle displacements to enforce incompressibility via density constraints [S1, unofficial]. It operates by using a spatial hash grid to identify neighboring particles within a search radius and applying separation forces to push particles apart if they are too close [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Input:** `positionBufferIN` (array of `vec4<f32>`), where `.rgb` is the particle position and `.a` encodes the radius/identifier [S1, unofficial].
*   **Output:** `positionBufferOUT` (array of `vec4<f32>`, read-write), storing the updated positions [S1, unofficial].
*   **Auxiliary Input:** `indicesBuffer` (array of `u32`), storing particle indices for the current voxel [S1, unofficial].
*   **Uniforms:** `uResolution` (grid resolution), `uSearchRadius` (implied by separation logic), `separation` (target separation distance), `maxParticlesPerVoxel` (max particles per voxel) [S1, unofficial].

**Data Structures:**
*   **Grid Position:** `vec3<i32>` derived from `floor(particlePosition.rgb)` [S1, unofficial].
*   **Neighbor Offsets:** A static array of 27 `vec3<i32>` values representing the $3 \times 3 \times 3$ neighborhood (offsets from $(-1,-1,-1)$ to $(1,1,1)$) [S1, unofficial].
*   **Delta Position:** `vec3<f32>` initialized to `(0., 0., 0.)` for each particle [S1, unofficial].

**Algorithm Steps:**
1.  Initialize `deltaPosition` to `vec3<f32>(0.)` for the current particle [S1, unofficial].
2.  Read `particlePosition` from `positionBufferIN` [S1, unofficial].
3.  Extract the radius identifier from `particlePosition.a` and compute `gridPosition` as `vec3<i32>(floor(particlePosition.rgb))` [S1, unofficial].
4.  Define the 27 neighbor voxel offsets [S1, unofficial].
5.  Loop over all 27 neighbor voxels ($i$ from 0 to 26):
    a. Calculate `neighborsVoxel = gridPosition + offsets[i]` [S1, unofficial].
    b. Calculate `voxelIndex = neighborsVoxel.x + neighborsVoxel.y * uResolution + neighborsVoxel.z * uResolution * uResolution` [S1, unofficial].
    c. Loop over all particles $j$ stored in the neighbor voxel using `indicesBuffer` (up to `maxParticlesPerVoxel`) [S1, unofficial].
    d. For each neighbor particle $j$, retrieve its position from `positionBufferIN` [S1, unofficial].
    e. Call `addToSum(particlePosition, neighborParticlePosition)` [S1, unofficial].
6.  Apply the accumulated `deltaPosition` to the particle's position in `positionBufferOUT` [S1, unofficial].

**Helper Function: `addToSum`**
Given `currentPos` and `neighborPos`:
1.  Calculate `distanceVector = currentPos.rgb - neighborPos.rgb` [S1, unofficial].
2.  Calculate `r = length(distanceVector)` [S1, unofficial].
3.  Calculate `r1 = floor(currentPos.a) / 4.0` [S1, unofficial].
4.  Calculate `r2 = floor(neighborPos.a) / 4.0` [S1, unofficial].
5.  Calculate `separation = 0.5 * (r1 + r2)` [S1, unofficial].
6.  If `r > 0` and `r < separation`:
    a. Calculate `normalizedDistance = normalize(distanceVector)` [S1, unofficial].
    b. Update `deltaPosition`:
       $$ \Delta p \leftarrow \Delta p - (r - \text{separation}) \cdot \text{normalizedDistance} \cdot \frac{r_2}{r_1 + r_2} $$
       [S1, unofficial].

**Parameters:**
*   `uResolution`: Grid resolution [S1, unofficial].
*   `separation`: Target separation distance [S1, unofficial].
*   `maxParticlesPerVoxel`: Maximum number of particles stored per voxel for neighbor lookup [S1, unofficial].
*   `uSearchRadius`: Implied by the separation logic but not explicitly used in the loop condition in the source [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Distance:** The condition `r > 0` prevents division by zero during normalization [S1, unofficial].
*   **Radius Encoding:** The radius is encoded in the `.a` component and scaled by division by 4.0 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def vec3(x, y, z):
    return (x, y, z)

def vec4(x, y, z, w):
    return (x, y, z, w)

def length(v):
    return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)

def normalize(v):
    l = length(v)
    if l == 0.0:
        return vec3(0.0, 0.0, 0.0)
    return vec3(v[0]/l, v[1]/l, v[2]/l)

def calculate_displacements(position_buffer_in, position_buffer_out, indices_buffer, u_resolution, max_particles_per_voxel):
    num_particles = len(position_buffer_in)
    for particle_index in range(num_particles):
        particle_position = position_buffer_in[particle_index]
        delta_position = vec3(0.0, 0.0, 0.0)

        grid_position = vec3(math.floor(particle_position[0]), math.floor(particle_position[1]), math.floor(particle_position[2]))

        offsets = [
            vec3(-1,-1,-1), vec3(-1,-1,0), vec3(-1,-1,1),
            vec3(-1,0,-1),  vec3(-1,0,0),  vec3(-1,0,1),
            vec3(-1,1,-1),  vec3(-1,1,0),  vec3(-1,1,1),
            vec3(0,-1,-1),  vec3(0,-1,0),  vec3(0,-1,1),
            vec3(0,0,-1),   vec3(0,0,0),   vec3(0,0,1),
            vec3(0,1,-1),   vec3(0,1,0),   vec3(0,1,1),
            vec3(1,-1,-1),  vec3(1,-1,0),  vec3(1,-1,1),
            vec3(1,0,-1),   vec3(1,0,0),   vec3(1,0,1),
            vec3(1,1,-1),   vec3(1,1,0),   vec3(1,1,1)
        ]

        for i in range(27):
            neighbors_voxel = vec3(
                grid_position[0] + offsets[i][0],
                grid_position[1] + offsets[i][1],
                grid_position[2] + offsets[i][2]
            )
            voxel_index = neighbors_voxel[0] + neighbors_voxel[1] * u_resolution + neighbors_voxel[2] * u_resolution * u_resolution

            for j in range(max_particles_per_voxel):
                neighbor_index = indices_buffer[voxel_index * max_particles_per_voxel + j]
                neighbor_position = position_buffer_in[neighbor_index]
                delta_position = add_to_sum(particle_position, neighbor_position, delta_position)

        position_buffer_out[particle_index] = vec4(
            particle_position[0] + delta_position[0],
            particle_position[1] + delta_position[1],
            particle_position[2] + delta_position[2],
            particle_position[3]
        )

def add_to_sum(current_pos, neighbor_pos, current_delta):
    distance_vector = vec3(current_pos[0] - neighbor_pos[0], current_pos[1] - neighbor_pos[1], current_pos[2] - neighbor_pos[2])
    r = length(distance_vector)

    r1 = math.floor(current_pos[3]) / 4.0
    r2 = math.floor(neighbor_pos[3]) / 4.0

    separation = 0.5 * (r1 + r2)

    if r > 0 and r < separation:
        normalized_distance = normalize(distance_vector)
        factor = -(r - separation) * (r2 / (r1 + r2))
        delta = vec3(
            factor * normalized_distance[0],
            factor * normalized_distance[1],
            factor * normalized_distance[2]
        )
        current_delta = vec3(
            current_delta[0] + delta[0],
            current_delta[1] + delta[1],
            current_delta[2] + delta[2]
        )

    return current_delta
```
## Subtopics

### [Local Short-Range Repulsion](15114-local-short-range-repulsion.md)
Local Short-Range Repulsion is a particle distribution optimization step used in adaptive voxel-based volume rendering to resolve particle clustering and improve surface extraction quality [15114-local-short-range-repulsion.md]. It operates by applying short-range repulsive forces between neighboring particles to spread them out, ensuring a more uniform sampling density [15114-local-short-range-repulsion.md]. It differs from the parent PBF Displacement Calculation which enforces incompressibility via density constraints, this method focuses on geometric distribution regularity through tangential force projection [G].
#### Distance Constraint (Repulsion)
**Definition:** A constraint to promote uniform particle distribution by pushing apart particles closer than a distance threshold $d_0$ [S2].
**Formula:** The distance constraint is defined as:
$$ D_{ij} = \min\{0, \|p_i - p_j\| - d_0\} $$
where $d_0$ is the distance threshold [S2].

## Operating characteristics
*   **Strengths:** Efficient neighbor search using spatial hashing; enforces incompressibility effectively for fluid-like behavior [S1, unofficial].

## Background
PBF (Particle-Based Fluid) simulation is a method for simulating fluid dynamics using particles [S1, unofficial]. The specific displacement calculation described here originates from a WebGPU-based voxel cone tracing implementation [S1, unofficial]. The distance constraint formula for repulsion is associated with Gaussian Splashing, a method for unified particles for versatile motion synthesis and rendering [S2].

## Sources
[S1] HectorArellanoDev/WebGPUVoxelConeTracing, https://github.com/HectorArellanoDev/WebGPUVoxelConeTracing, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
