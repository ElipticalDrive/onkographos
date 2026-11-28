# GPU SPH Implementation via Distribution and Bucket Space

Also called: Bucket Space Texture SPH, Distribution-based SPH, GPU SPH

## Overview
This method implements Smoothed Particle Hydrodynamics (SPH) on the GPU by inverting the standard gathering computation into a distribution process, effectively treating particle interactions as 3D splatting [S1]. It solves the parallelization bottleneck of neighbor searches by mapping particles to a 3D grid (bucket space) and accumulating contributions via texture storage, enabling adaptive sampling (splitting and merging) directly within the rendering pipeline [S1]. Unlike gathering methods that require explicit neighbor lists, this approach leverages the GPU's rasterization and fragment accumulation capabilities to handle density and force calculations in parallel [S1].

## How it works

### Inputs and Outputs
*   **Inputs:** Particle attributes (position, velocity, density, mass) stored in vertex buffers; smoothing length $h$; time step $\Delta t$; bucket space resolution.
*   **Outputs:** Updated particle attributes in vertex buffers; rendered fluid visualization.

### Data Structures
The implementation relies on specific GPU memory structures to facilitate the distribution and merging processes [S1]:

1.  **Bucket Space:** A 3D grid covering the simulation domain. Each particle is mapped to the closest voxel. The physical attributes (position, velocity, density, mass) are stored in the closest voxel. Crucially, the *original* particle attributes are stored, not voxelized averages, to avoid discretization errors in the mapping process [S1].
2.  **Slice Space Texture:** The 3D bucket space grid is interpreted as a stack of 2D slices along the Y-axis. These slices are combined into a single large 2D texture to allow efficient GPU texture sampling and storage [S1].
3.  **Vertex Buffers ($V_1, V_2$):** Two dynamic vertex buffers are used for double-buffering. $V_1$ stores the current set of particles. After adaptive sampling (splitting/merging) in the geometry shader, the new particle set is written to $V_2$ via the stream-output stage. The buffers are swapped for the next time step [S1].
4.  **Textures ($TB_1, TB_2$):**
    *   $TB_1$: Accumulation buffer used during the mapping and distribution phases.
    *   $TB_2$: Stores the normalized results after the merging pass [S1].
5.  **Texture $TID$:** A primitive ID texture. It stores the unique primitive ID for each particle. This is used to identify and discard obsolete particles after merging [S1].

### Simulation Stages
The simulation proceeds in four passes per time step [S1]:

#### 1. Distributing Particles
Each particle is rasterized as one texel in the bucket slice space texture $TB_1$.
*   **Mapping:** Particles are mapped to their closest voxel in the bucket space.
*   **Accumulation:** If multiple particles map to the same texel, their attributes are accumulated using mass as the weight.
    *   Mass accumulation: $M_{voxel} = \sum m_i$
    *   Attribute accumulation (e.g., density $\rho$, velocity $\mathbf{v}$): $A_{voxel} = \sum m_i A_i$
*   **ID Recording:** The unique primitive ID of each particle is computed and recorded in texture $TID$. The system-supplied values `SV_PrimitiveID` or `SV_VertexID` (from Direct3D 10) are used as unique IDs [S1].

#### 2. Merging Particles
This is an image processing pass to normalize the attributes of merged particles and output results to texture $TB_2$.
*   **Normalization:** The accumulated attributes are divided by the accumulated mass.
    *   Normalized Density: $\rho_{new} = \frac{\sum m_i \rho_i}{\sum m_i}$
    *   Normalized Velocity: $\mathbf{v}_{new} = \frac{\sum m_i \mathbf{v}_i}{\sum m_i}$
*   The result is written to $TB_2$ [S1].

#### 3. 3D Splatting (Interaction)
The 3D splat is decomposed into a set of 2D splats in the geometry shader.
*   **Decomposition:** A 3D spherical splat centered at particle position $\mathbf{p}$ with smoothing length $h$ is intersected with the 2D slices of the bucket space. This forms multiple 2D splats with varying radii [S1].
*   **Contribution:** The contribution from the particle to its neighbors (covered by the 2D splats) is computed and accumulated at the corresponding texels of the neighbors in the accumulation buffer. This replaces the standard gathering loop [S1].

#### 4. Updating and Splitting Particles
This pass updates particle attributes and performs adaptive sampling.
*   **Update:** Attributes are computed in the vertex shader based on values stored in the textures.
*   **Obsolete Particle Removal:** Particles are compared against the ID recorded in $TID$. The depth test function is set to pass all fragments so only the last written fragment is kept for a single texel in $TID$. Particles whose ID does not match the value in $TID$ are discarded as obsolete [S1].
*   **Splitting:** Performed in the geometry shader at the end of the time step, after attributes are updated. Particles are split if they exceed a mass threshold. The output is directed to vertex buffer $V_2$ via the stream-output stage [S1].

### Adaptive Sampling Parameters

#### Splitting Threshold
Splitting is performed if a particle's mass $m$ is greater than a threshold $\varepsilon m$. The threshold is derived to ensure that sub-particles do not map to the same voxel, violating the resolution limit. The condition is:

$$ \alpha \sigma \left( \frac{\varepsilon_m}{\rho} \right)^{1/3} \geq \sqrt{3}s $$

Where:
*   $\varepsilon_m$: Minimum mass for splitting.
*   $\rho$: Density.
*   $s$: Smoothing length (or related spatial scale).
*   $\alpha, \sigma$: Constants related to the grid resolution and particle distribution [S1].

Rearranging for the threshold:

$$ \varepsilon_m \geq \rho \left( \frac{\sqrt{9/8s}}{\alpha \sigma} \right)^3 $$

#### Merging Strategy
Merging is applied to "inner" particles to reduce particle count.
*   **Mapping Rule:** Inner particles are mapped to the closest voxel with **even coordinates** in the bucket space, rather than the absolute closest voxel. This groups nearby inner particles into the same voxel for merging [S1].
*   **Collision Handling:** Sub-particles from different parent particles or non-inner particles may map to the same voxel, exceeding the resolution limit. This is rare due to the sensitivity of pressure forces to density changes. The strategy is to merge them into a single particle [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structures
# V1, V2: Vertex Buffers (swapped each step)
# TB1, TB2: 2D Textures (Bucket Space Slice Space)
# TID: 2D Texture (Primitive IDs)
# bucket_size: 3D grid dimensions
# voxel_size: size of one voxel in bucket space

def simulate_step(V1, V2, TB1, TB2, TID, bucket_size, voxel_size):
    # Pass 1: Distributing Particles
    # Rasterize particles from V1 into TB1 and TID
    for particle in V1:
        # Map particle position to closest voxel coordinates
        voxel_coords = get_closest_voxel(particle.position, bucket_size, voxel_size)
        
        # Accumulate attributes in TB1 using mass weighting
        # TB1[voxel_coords].mass += particle.mass
        # TB1[voxel_coords].density_weighted += particle.mass * particle.density
        # TB1[voxel_coords].velocity_weighted += particle.mass * particle.velocity
        
        # Record unique ID in TID
        # Use SV_PrimitiveID or SV_VertexID as unique ID
        TID[voxel_coords] = particle.primitive_id

    # Pass 2: Merging Particles
    # Normalize accumulated attributes in TB1 and write to TB2
    for voxel in TB1:
        if voxel.mass > 0:
            TB2[voxel].density = voxel.density_weighted / voxel.mass
            TB2[voxel].velocity = voxel.velocity_weighted / voxel.mass
            TB2[voxel].mass = voxel.mass
        else:
            TB2[voxel].mass = 0

    # Pass 3: 3D Splatting (Interaction)
    # Decompose 3D splat into 2D splats and accumulate contributions
    for particle in V1:
        # Decompose spherical influence region into 2D slices
        slices = decompose_3d_splat_to_2d(particle.position, particle.smoothing_length, bucket_size)
        
        for slice in slices:
            # Compute contribution to neighbors covered by this 2D slice
            # Accumulate force/density contributions into TB1 or a separate interaction buffer
            accumulate_contribution(particle, slice, TB1)

    # Pass 4: Updating and Splitting
    V2.clear()
    
    # Read back updated attributes from TB2/TB1 to V2
    # Discard obsolete particles by comparing ID with TID
    for particle in V1:
        voxel_coords = get_closest_voxel(particle.position, bucket_size, voxel_size)
        stored_id = TID[voxel_coords]
        
        if particle.primitive_id != stored_id:
            continue # Discard obsolete particle
        
        # Update attributes from textures
        particle.density = TB2[voxel_coords].density
        particle.velocity = TB2[voxel_coords].velocity
        particle.mass = TB2[voxel_coords].mass
        
        # Splitting Check
        # If mass > epsilon_m, split into sub-particles
        if particle.mass > calculate_splitting_threshold(particle.density):
            sub_particles = split_particle(particle)
            for sub in sub_particles:
                V2.add(sub)
        else:
            V2.add(particle)
            
    # Swap buffers for next step
    V1, V2 = V2, V1
```

## Operating characteristics

*   **Time Complexity:** The method avoids $O(N^2)$ neighbor searches. The distribution and splatting steps are proportional to the number of particles $N$ and the number of voxels covered by the smoothing kernel. The merging step is proportional to the number of voxels. Overall complexity is closer to $O(N)$ with a small constant factor dependent on kernel support and grid resolution [S1].
*   **Memory Complexity:** Requires storage for two vertex buffers, two accumulation textures ($TB_1, TB_2$), and one ID texture ($TID$). The size of textures is determined by the bucket space resolution, which must be sufficient to cover the simulation domain [S1].
*   **Strengths:**
    *   Highly parallelizable on GPU due to distribution-based approach.
    *   Adaptive sampling reduces particle count in dense regions while maintaining resolution in sparse regions.
    *   No explicit neighbor list construction required.
*   **Limitations:**
    *   Resolution limited by bucket space grid size. If particles are too close, they may map to the same voxel, requiring merging (which can introduce artifacts if not handled carefully) [S1].
    *   3D splatting must be decomposed into 2D slices, adding complexity to the geometry shader [S1].
    *   Precision issues may arise from floating-point accumulation in textures [G].
*   **When to Use:** Suitable for real-time fluid simulation on GPUs where adaptive resolution is beneficial and explicit neighbor search is a bottleneck.
*   **When Not to Use:** When extremely high precision in particle positions is required without grid quantization, or when the simulation domain is very large relative to particle count (sparse), making the bucket space texture memory inefficient [S1].

## Background

This method was introduced in the paper "Adaptive Sampling and Rendering of Fluids on the GPU" (2008) [S1]. It addresses the limitations of standard SPH on GPUs, particularly the inefficiency of gathering-based neighbor searches. By converting the computation to a distribution process and utilizing a bucket space texture data structure, it enables efficient adaptive sampling (splitting and merging) directly within the graphics pipeline. The approach builds on techniques from point-based graphics for 3D splatting decomposition [S1].

## Sources

[S1] Adaptive Sampling and Rendering of Fluids on the GPU, https://doi.org/10.2312/vg/vg-pbg08/137-146, fetched 2026-10-26
