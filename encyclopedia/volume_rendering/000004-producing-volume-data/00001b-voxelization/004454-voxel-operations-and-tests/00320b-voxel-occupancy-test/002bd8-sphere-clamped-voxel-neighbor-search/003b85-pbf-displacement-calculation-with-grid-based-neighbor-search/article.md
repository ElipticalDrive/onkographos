# PBF Displacement Calculation with Grid-Based Neighbor Search
## Overview
PBF Displacement Calculation with Grid-Based Neighbor Search is a Particle-Based Fluid (PBF) simulation step that calculates particle displacements to enforce incompressibility via density constraints [S1, unofficial]. It relies on a spatial hash grid to efficiently locate neighboring particles within a search radius and applies separation forces to push overlapping particles apart [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Input:** `positionBufferIN` (array of `vec4<f32>`, read-only), where `.rgb` is position and `.a` encodes particle radius/identifier [S1, unofficial].
*   **Output:** `positionBufferOUT` (array of `vec4<f32>`, read-write), storing updated positions [S1, unofficial].
*   **Auxiliary Input:** `indicesBuffer` (array of `u32`, read-only), storing particle indices for the current voxel [S1, unofficial].
*   **Uniforms:** `uResolution` (grid resolution), `maxParticlesPerVoxel` (max particles per voxel), `separation` (target separation distance, though the source logic derives separation from particle radii), `currentFrame`, `vertical`, and `uSearchRadius` (not explicitly used in loop condition but implied by separation logic) [S1, unofficial].

**Data Structures:**
*   **Particle Position:** `vec4<f32>`. The radius $r_i$ is extracted from the alpha channel: $r_i = \lfloor \text{position}.a \rfloor / 4$ [S1, unofficial].
*   **Grid Indexing:** Particles are mapped to a 3D grid. The grid position is `gridPosition = vec3<i32>(floor(particlePosition.rgb))` [S1, unofficial].
*   **Neighbor Search:** A 3x3x3 offset array of 27 `vec3<i32>` values covers the current voxel and all adjacent voxels (faces, edges, corners) [S1, unofficial]. Offsets range from $(-1,-1,-1)$ to $(1,1,1)$ [S1, unofficial].

**Algorithm Steps:**
1.  Initialize a global displacement vector `deltaPosition` to `vec3<f32>(0.)` for the current particle [S1, unofficial].
2.  For the current particle index, read its position `p` from `positionBufferIN` [S1, unofficial].
3.  Extract the particle's radius $r_1$ from `p.a`: $r_1 = \lfloor p.a \rfloor / 4$ [S1, unofficial].
4.  Compute the grid coordinate `gridPosition` using `floor(p.rgb)` [S1, unofficial].
5.  Iterate over all 27 neighbor voxels defined by the 3x3x3 offset array [S1, unofficial].
    a.  Calculate `neighborsVoxel` = `gridPosition` + `offsets[i]` [S1, unofficial].
    b.  Calculate the linear `voxelIndex` in the hash grid: `voxelIndex` = `neighborsVoxel.x` + `neighborsVoxel.y` * `uResolution` + `neighborsVoxel.z` * `uResolution` * `uResolution` [S1, unofficial].
    c.  Retrieve the list of particle indices stored in `indicesBuffer` for this `voxelIndex` [S1, unofficial].
    d.  Iterate through each neighbor particle index `j` in the voxel (up to `maxParticlesPerVoxel`) [S1, unofficial].
    e.  Retrieve the neighbor's position `p_j` from `positionBufferIN` [S1, unofficial].
    f.  Extract the neighbor's radius $r_2$: $r_2 = \lfloor p_j.a \rfloor / 4$ [S1, unofficial].
    g.  Calculate the distance vector $\vec{d} = p.rgb - p_j.rgb$ [S1, unofficial].
    h.  Calculate the distance $r = \|\vec{d}\|$ [S1, unofficial].
    i.  Calculate the target separation distance $s = 0.5 \times (r_1 + r_2)$ [S1, unofficial].
    j.  If $r > 0$ and $r < s$, apply the separation force:
        $$ \text{deltaPosition} \ -= \ (r - s) \times \frac{\vec{d}}{r} \times \frac{r_2}{r_1 + r_2} $$
6.  After checking all neighbors, update the particle's position in `positionBufferOUT` by adding `deltaPosition` to the original position [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_displacements(particle_idx, position_buffer_in, position_buffer_out, indices_buffer, u_resolution, max_particles_per_voxel):
    # 1. Initialize delta position
    delta_position = vec3(0.0, 0.0, 0.0)
    
    # 2. Read current particle data
    p_vec4 = position_buffer_in[particle_idx]
    p_pos = p_vec4.rgb
    p_radius_alpha = p_vec4.a
    
    # Extract radius r1
    r1 = floor(p_radius_alpha) / 4.0
    
    # 3. Compute grid position
    grid_pos = vec3(floor(p_pos.x), floor(p_pos.y), floor(p_pos.z))
    
    # 4. Define 3x3x3 offsets
    offsets = []
    for x in [-1, 0, 1]:
        for y in [-1, 0, 1]:
            for z in [-1, 0, 1]:
                offsets.append(vec3(x, y, z))
                
    # 5. Iterate over neighbor voxels
    for offset in offsets:
        neighbors_voxel = grid_pos + offset
        
        # Calculate linear voxel index
        voxel_index = neighbors_voxel.x + neighbors_voxel.y * u_resolution + neighbors_voxel.z * u_resolution * u_resolution
        
        # 6. Iterate over particles in the voxel
        base_idx = voxel_index * max_particles_per_voxel
        for k in range(max_particles_per_voxel):
            neighbor_particle_idx = indices_buffer[base_idx + k]
            
            p_j_vec4 = position_buffer_in[neighbor_particle_idx]
            p_j_pos = p_j_vec4.rgb
            p_j_radius_alpha = p_j_vec4.a
            
            # Extract radius r2
            r2 = floor(p_j_radius_alpha) / 4.0
            
            # Calculate distance vector and magnitude
            dist_vec = p_pos - p_j_pos
            r = length(dist_vec)
            
            # Calculate target separation
            separation = 0.5 * (r1 + r2)
            
            # Apply separation force if within range
            if r > 0.0 and r < separation:
                normalized_dist = dist_vec / r
                weight = r2 / (r1 + r2)
                delta_position -= (r - separation) * normalized_dist * weight
                
    # 7. Apply displacement
    new_pos = p_pos + delta_position
    position_buffer_out[particle_idx] = vec4(new_pos, p_radius_alpha)
```

**Parameters and Edge Cases:**
*   **Radius Extraction:** The radius is derived from the alpha channel using `floor(alpha) / 4`, implying the alpha channel stores an integer value representing $4 \times$ radius [S1, unofficial].
*   **Singularity:** The distance $r$ is checked against `0.0` to avoid division by zero when normalizing the distance vector [S1, unofficial].
*   **Boundary Conditions:** The grid search is limited to the 3x3x3 neighborhood, and particles near the domain boundary naturally have fewer neighbors [S1, unofficial].
*   **Precision:** Floating-point precision in `floor` operations and radius extraction may cause slight jitter if particle positions are not stable [G].

## Subtopics

### Distance Constraint (Repulsion)
A constraint to promote uniform particle distribution by pushing apart particles closer than a distance threshold $d_0$ [S2]. The distance constraint is calculated as $D_{ij} = \min\{0, \|p_i - p_j\| - d_0\}$ [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K)$, where $N$ is the number of particles and $K$ is the average number of neighbors found within the 3x3x3 voxel neighborhood [G]. The constant factor is small due to the fixed 27-voxel search [G].
*   **Memory Complexity:** $O(N)$ for position buffers and $O(V)$ for the voxel index buffers, where $V$ is the number of voxels [G].
*   **Strengths:** Efficient neighbor search using spatial hashing and simple separation force logic suitable for real-time simulation on GPU [G].
*   **Limitations:** The separation force is a heuristic and may not perfectly conserve mass or momentum compared to more rigorous PBF formulations [G]. The radius extraction method is specific to the implementation and may not generalize to other PBF variants [G].
*   **When to use:** Real-time fluid simulation where performance is critical and visual plausibility is acceptable [G].
*   **When not to use:** High-fidelity scientific simulation requiring strict incompressibility constraints [G].

## Background
The method implements a PBF simulation step using a spatial hash grid for neighbor lookup and separation forces [S1, unofficial]. It operates as a compute shader or parallel loop step within a larger voxel-based rendering or simulation pipeline [S1, unofficial].

## Sources
[S1] HectorArellanoDev/WebGPUVoxelConeTracing, https://github.com/HectorArellanoDev/WebGPUVoxelConeTracing, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
