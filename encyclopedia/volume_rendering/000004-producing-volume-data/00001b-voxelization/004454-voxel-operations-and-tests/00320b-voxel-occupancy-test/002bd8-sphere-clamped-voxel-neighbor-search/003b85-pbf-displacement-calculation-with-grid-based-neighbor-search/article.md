# PBF Displacement Calculation with Grid-Based Neighbor Search


## Overview
PBF Displacement Calculation with Grid-Based Neighbor Search is a particle-based fluid simulation step that calculates particle displacements to enforce incompressibility via density constraints [S1, unofficial]. It operates by using a spatial hash grid to identify neighboring particles within a search radius and applying separation forces to push particles apart if they are too close [S1, unofficial]. Unlike its parent method, Sphere-Clamped Voxel Neighbor Search, which focuses on efficient neighbor retrieval, this method performs the actual physical constraint resolution [S1, unofficial]. It differs from Local Short-Range Repulsion, which optimizes geometric distribution regularity through tangential force projection, by strictly enforcing volumetric incompressibility [15114-local-short-range-repulsion.md].

## How it works
**Inputs and Outputs:**
*   **Input:** `positionBufferIN` (array of `vec4<f32>`), where `.rgb` is the particle position and `.a` encodes the radius/identifier.
*   **Output:** `positionBufferOUT` (array of `vec4<f32>`, read-write), storing the updated positions.
*   **Auxiliary Input:** `indicesBuffer` (array of `u32`), storing particle indices for the current voxel.
*   **Uniforms:** `uResolution` (grid resolution), `uSearchRadius` (implied), `separation` (target separation distance), `maxParticlesPerVoxel` (max particles per voxel).

**Data Structures:**
*   **Grid Position:** `vec3<i32>` derived from `floor(particlePosition.rgb)`.
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
       $$ \Delta p = - (r - separation) \cdot \text{normalizedDistance} \cdot \frac{r2}{r1 + r2} $$
       [S1, unofficial].

**Parameters:**
*   `uResolution`: Grid resolution [S1, unofficial].
*   `separation`: Target separation distance [S1, unofficial].
*   `maxParticlesPerVoxel`: Maximum number of particles stored per voxel for neighbor lookup [S1, unofficial].
*   `uSearchRadius`: Implied by the separation logic but not explicitly used in the loop condition in the source [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Distance:** The condition `r > 0` prevents division by zero during normalization [S1, unofficial].
*   **Radius Encoding:** The radius is encoded in the `.a` component and scaled by division by 4.0. Implementation must ensure this encoding matches the input data [S1, unofficial].
*   **Boundary Conditions:** The grid search assumes particles are within the bounds of the hash grid. Out-of-bounds voxel indices must be handled or clamped [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_displacements(position_buffer_in, position_buffer_out, indices_buffer, u_resolution, max_particles_per_voxel):
    # Assume particle_index is provided by the caller or looped over
    # For each particle:
    particle_position = position_buffer_in[particle_index]
    delta_position = vec3(0.0)
    
    # Extract grid position
    grid_position = vec3i(floor(particle_position.rgb))
    
    # Define 27 offsets
    offsets = [
        vec3i(-1,-1,-1), vec3i(-1,-1,0), vec3i(-1,-1,1),
        vec3i(-1,0,-1),  vec3i(-1,0,0),  vec3i(-1,0,1),
        vec3i(-1,1,-1),  vec3i(-1,1,0),  vec3i(-1,1,1),
        vec3i(0,-1,-1),  vec3i(0,-1,0),  vec3i(0,-1,1),
        vec3i(0,0,-1),   vec3i(0,0,0),   vec3i(0,0,1),
        vec3i(0,1,-1),   vec3i(0,1,0),   vec3i(0,1,1),
        vec3i(1,-1,-1),  vec3i(1,-1,0),  vec3i(1,-1,1),
        vec3i(1,0,-1),   vec3i(1,0,0),   vec3i(1,0,1),
        vec3i(1,1,-1),   vec3i(1,1,0),   vec3i(1,1,1)
    ]
    
    for i in range(27):
        neighbors_voxel = grid_position + offsets[i]
        
        # Calculate voxel index in the hash grid
        # Note: indices_buffer logic implies accessing particles stored for this voxel
        # The source snippet cuts off the exact indices_buffer access pattern, 
        # but standard logic iterates through stored indices for the voxel.
        # Assuming indices_buffer contains the list of particle indices for the current voxel context.
        # In a full implementation, one would look up the start/end index for neighbors_voxel 
        # in a hash map or sorted array. Here we assume the loop over 'j' iterates 
        # through the relevant neighbor particles.
        
        # Pseudocode approximation of the neighbor loop based on source description:
        # for j in range(max_particles_per_voxel):
        #     neighbor_index = indices_buffer[...] # Implementation detail omitted in source
        #     neighbor_position = position_buffer_in[neighbor_index]
        #     delta_position += add_to_sum(particle_position, neighbor_position)
        
        # Since the source explicitly describes the loop over particles in the neighbor voxel,
        # we represent the core logic of add_to_sum which is fully defined.
        pass 

    # Apply delta position
    position_buffer_out[particle_index] = particle_position + vec4(delta_position, particle_position.a)

def add_to_sum(current_pos, neighbor_pos):
    distance_vector = current_pos.rgb - neighbor_pos.rgb
    r = length(distance_vector)
    
    r1 = floor(current_pos.a) / 4.0
    r2 = floor(neighbor_pos.a)  / 4.0
    
    separation = 0.5 * (r1 + r2)
    
    delta = vec3(0.0)
    
    if r > 0 and r < separation:
        normalized_distance = normalize(distance_vector)
        # Formula from source: deltaPosition -= (r - separation) * normalize(distance) * r2 / (r1 + r2)
        delta = - (r - separation) * normalized_distance * (r2 / (r1 + r2))
        
    return delta
```

## Subtopics

### [Local Short-Range Repulsion](15114-local-short-range-repulsion.md)
Local Short-Range Repulsion is a particle distribution optimization step used in adaptive voxel-based volume rendering to resolve particle clustering and improve surface extraction quality [15114-local-short-range-repulsion.md]. It operates by applying short-range repulsive forces between neighboring particles to spread them out, ensuring a more uniform sampling density [15114-local-short-range-repulsion.md]. Unlike the parent PBF Displacement Calculation which enforces incompressibility via density constraints, this method focuses on geometric distribution regularity through tangential force projection [15114-local-short-range-repulsion.md].

#### Distance Constraint (Repulsion)
**Definition:** A constraint to promote uniform particle distribution by pushing apart particles closer than a distance threshold $d_0$ [S2].
**Formula:** The distance constraint is defined as:
$$ D_{ij} = \min\{0, \|p_i - p_j\| - d_0\} $$
where $d_0$ is the distance threshold [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K)$, where $N$ is the number of particles and $K$ is the average number of neighbors found within the $3 \times 3 \times 3$ voxel neighborhood. The grid search reduces neighbor lookup from $O(N^2)$ to near-linear time [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for particle storage and $O(R^3)$ for the spatial hash grid, where $R$ is the grid resolution [S1, unofficial].
*   **Strengths:** Efficient neighbor search using spatial hashing; enforces incompressibility effectively for fluid-like behavior [S1, unofficial].
*   **Limitations:** The $3 \times 3 \times 3$ grid search assumes particles are well-distributed within voxels; clustering may require multiple iterations or smaller voxel sizes [S1, unofficial]. The radius encoding in the `.a` component requires specific preprocessing [S1, unofficial].
*   **When to use:** When simulating incompressible fluids or dense particle systems requiring volume preservation [S1, unofficial].
*   **When not to use:** When long-range interactions are critical or when particle distribution regularity is more important than volume preservation (use Local Short-Range Repulsion instead) [15114-local-short-range-repulsion.md].

## Background
PBF (Particle-Based Fluid) simulation is a method for simulating fluid dynamics using particles [S1, unofficial]. The specific displacement calculation described here originates from a WebGPU-based voxel cone tracing implementation [S1, unofficial]. The distance constraint formula for repulsion is associated with Gaussian Splashing, a method for unified particles for versatile motion synthesis and rendering [S2].

## Sources
[S1] HectorArellanoDev/WebGPUVoxelConeTracing, https://github.com/HectorArellanoDev/WebGPUVoxelConeTracing, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
