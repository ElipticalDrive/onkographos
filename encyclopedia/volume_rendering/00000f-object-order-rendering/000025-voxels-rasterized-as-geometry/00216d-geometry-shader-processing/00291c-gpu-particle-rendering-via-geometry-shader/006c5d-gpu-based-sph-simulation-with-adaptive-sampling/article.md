# GPU-based SPH Simulation with Adaptive Sampling

Also called: Adaptive SPH on GPU

## Overview
This method implements Smoothed Particle Hydrodynamics (SPH) on the GPU by inverting the standard gathering computation into a distribution process, effectively treating particle interactions as 3D splatting [S1]. It solves the parallelization bottleneck of neighbor searches by mapping particles to a 3D grid (bucket space) and accumulating contributions via texture storage, enabling adaptive sampling (splitting and merging) directly within the rendering pipeline [S1]. Unlike gathering methods that require explicit neighbor lists, this approach leverages the GPU's rasterization and fragment accumulation capabilities to handle density and force calculations in parallel [S1]. It builds on [GPU Particle Rendering via Geometry Shader](4444-gpu-particle-rendering-via-geometry-shader.md) to generate the necessary geometry for splatting.

## How it works
The simulation proceeds through a four-pass rendering algorithm [S1].

**Inputs and Outputs:**
*   **Input:** Particle attributes (position, velocity, mass, primitive ID) stored in a vertex buffer.
*   **Output:** Updated particle attributes in a vertex buffer for the next frame.
*   **Textures:**
    *   $TB_1$: Bucket slice space texture. Each particle is rasterized as one texel. Attributes are accumulated by mass.
    *   $TID$: Texture storing the unique primitive ID of each particle.
    *   $TB_2$: Output texture for normalized attributes of merged particles.

**Data Structures:**
*   **Bucket Space:** A 3D grid representing the simulation domain. The resolution is fine to minimize the threshold $\epsilon \cdot m$ for adaptive sampling [S1].
*   **Primitive ID:** A unique identifier for each particle, used to track validity during merging and splitting.

**Steps:**

1.  **Distributing Particles:**
    Each particle is rasterized as one texel in the bucket slice space texture $TB_1$ [S1]. If multiple particles map to the same texel, their attributes are accumulated using their mass as the accumulation weight [S1]. Simultaneously, the unique primitive ID of each particle is computed and recorded in texture $TID$ [S1].

2.  **Merging Particles:**
    An image processing pass normalizes the attributes of merged particles. The results are output to texture $TB_2$ [S1]. This step handles particles that have been combined due to proximity or density criteria.

3.  **3D Splatting:**
    The 3D splat is decomposed into a set of 2D splats in the geometry shader [S1]. The contribution from each particle to its neighbors, covered by the set of 2D splats, is computed and accumulated at the corresponding texels of those neighbors [S1]. This replaces the traditional neighbor search loop with parallel rasterization accumulation.

4.  **Updating and Splitting Particles:**
    This pass computes particle attributes in the vertex shader based on values stored in the textures [S1].
    *   **Discarding Obsolete Particles:** Particles that no longer exist or have been merged/split are discarded by comparing their primitive ID to the value recorded in $TID$ [S1].
    *   **Splitting:** In the geometry shader, particles satisfying specific splitting criteria are split into four sub-particles [S1].
    *   **Output:** The remaining original particles and the newly split sub-particles are streamed out into a vertex buffer [S1].

**Parameters:**
*   **Splitting Criteria:** Particles are split if they satisfy specific conditions checked in the geometry shader. The exact mathematical condition is not explicitly defined in the sources, but it is implied to be related to particle density or proximity to prevent clumping [S1].
*   **Precision:** The simulation can run at FP32 or FP16 precision. FP16 offers performance improvements at the cost of physical accuracy [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 1: Distributing Particles
def pass_distribute(particles, TB1, TID):
    # Initialize TB1 and TID
    for particle in particles:
        # Map particle position to bucket space texel coordinates
        texel_coords = map_to_bucket(particle.position)
        
        # Accumulate attributes by mass
        TB1[texel_coords].mass += particle.mass
        TB1[texel_coords].momentum += particle.velocity * particle.mass
        # Other attributes accumulated similarly
        
        # Record primitive ID
        TID[texel_coords] = particle.primitive_id

# Pass 2: Merging Particles
def pass_merge(TB1, TB2):
    # Normalize attributes for merged particles
    for texel in TB1:
        if texel has multiple particles:
            # Normalize mass and momentum
            TB2[texel].mass = texel.mass
            TB2[texel].velocity = texel.momentum / texel.mass
        else:
            TB2[texel] = texel

# Pass 3: 3D Splatting
def pass_splat(particles, TB2):
    # Decompose 3D splat into 2D splats in Geometry Shader
    for particle in particles:
        # Compute contribution to neighbors
        neighbors = get_neighbors_covered_by_splat(particle.position, particle.radius)
        for neighbor in neighbors:
            # Accumulate force/density contributions
            TB2[neighbor.texel].force += compute_splat_contribution(particle, neighbor)

# Pass 4: Updating and Splitting
def pass_update(particles, TB2, TID):
    new_particles = []
    for particle in particles:
        # Retrieve updated attributes from textures
        texel = map_to_bucket(particle.position)
        updated_attrs = TB2[texel]
        
        # Check if particle is obsolete
        if TID[texel] != particle.primitive_id:
            continue # Discard obsolete particle
        
        # Update attributes
        particle.velocity = updated_attrs.velocity
        particle.force = updated_attrs.force
        
        # Check splitting criteria
        if satisfies_splitting_criteria(particle):
            # Split into four sub-particles
            sub_particles = split_into_four(particle)
            new_particles.extend(sub_particles)
        else:
            new_particles.append(particle)
    
    return new_particles
```

## Subtopics

### [GPU SPH Implementation via Distribution and Bucket Space](21188-gpu-sph-implementation-via-distribution-and-bucket-space.md)
This method implements Smoothed Particle Hydrodynamics (SPH) on the GPU by inverting the standard gathering computation into a distribution process, effectively treating particle interactions as 3D splatting. It solves the parallelization bottleneck of neighbor searches by mapping particles to a 3D grid (bucket space) and accumulating contributions via texture storage, enabling adaptive sampling (splitting and merging) directly within the rendering pipeline. Unlike gathering methods that require explicit neighbor lists, this approach leverages the GPU's rasterization and fragment accumulation capabilities to handle density and force calculations in parallel.

## Operating characteristics
*   **Performance:** On an NVIDIA GeForce 8800GTX, with adaptive sampling enabled, the method achieves 49.01 sim+visualize steps/sec for ~7.5k average particles, 37.28 for ~14k, and 8.21 for ~52.7k [S1].
*   **Comparison:** Without adaptive sampling, the method is approximately 2.5 times faster than [TH07] [S1].
*   **Precision Trade-off:** Running the simulation at half-floating-point precision (FP16) improves performance at the cost of physical accuracy, though most particles behave similarly to FP32 [S1].
*   **Grid Resolution:** A fine 3D grid is preferred to represent the simulation domain because it results in a smaller threshold $\epsilon \cdot m$, allowing adaptive sampling to produce smaller particles [S1]. However, a fine grid introduces more empty voxels, which can impact performance [S1].

## Background
This method was introduced in the paper "Adaptive Sampling and Rendering of Fluids on the GPU" [S1]. It represents an approach to GPU-accelerated SPH simulation that leverages the graphics pipeline for physics computation, specifically using the geometry shader for 3D splatting and adaptive sampling techniques to manage particle density and resolution dynamically.

## Sources
[S1] Adaptive Sampling and Rendering of Fluids on the GPU, https://doi.org/10.2312/vg/vg-pbg08/137-146, fetched 2026-10-26
