# Volumetric Particle Injection

Also called: particle voxelization, particles as volumes

## Overview
Volumetric Particle Injection is a real-time technique that injects discrete volumetric particles (smoke, steam, energy discharges) as density into a frustum-aligned froxel grid, allowing them to be lit by the same scattering pass as global fog [S1, unofficial]. It solves the problem of visible seams and lighting inconsistencies that arise when particles are rendered as decoupled billboards or separate volume layers with independent lighting models [S1, unofficial]. By unifying particle density with the global fog volume, it ensures consistent light scattering and shadowing across the entire participating medium [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A list of particle emitters, each defined by position, footprint size, and extinction coefficient.
*   **Output:** A froxel grid volume (typically `R32_UINT` during injection, converted to `R16_FLOAT` for rendering) containing accumulated density/extinction values.

**Data Structures**
*   **Froxel Grid:** A 3D grid of voxels. The injection target is a `uint` volume to support atomic operations.
*   **Particle Footprint:** Defined as a 2x2x2 cube (for point or trilinear sampling) or a sphere, proportional to the footprint in froxels rather than screen area [S1, unofficial].
*   **Shadow Map:** A separate cascaded clip map (3 levels, camera-following, with world-space snapping) is used for out-of-view shadow casters, as clip-space volume alone is insufficient [S1, unofficial].
*   **Refinement:** Sparse per-froxel refinement (brick or small octree) is attached only to froxels holding particles to handle high-detail effects like fluid surfaces or electricity arcs [S1, unofficial].

**Steps**
1.  **Initialize Volume:** Allocate a `uint` volume for particle density.
2.  **Iterate Emitters:** For each particle emitter:
    a.  Determine the footprint (2x2x2 cube or sphere) in froxel coordinates.
    b.  Calculate the normalized extinction value. Extinction 1.0 is mapped to 2048u in the uint volume [S1, unofficial].
    c.  **Normalize by Volume:** To prevent overshadowing in large voxel cascades where particles write more often into the same voxels, divide the extinction contribution by the volume of the footprint or the voxel cell size [S1, unofficial].
    d.  **Atomic Write:** For each froxel intersected by the footprint, use atomic addition (e.g., `InterlockedAdd`) to accumulate the normalized extinction value. This prevents multiple emitters from overwriting each other's data [S1, unofficial].
3.  **Convert Format:** After all particles are injected, convert the `uint` volume to `f16` (R16_FLOAT) for the rendering pass [S1, unofficial].
4.  **Raymarching:** During the volumetric rendering pass, march through the froxel grid. Cap the number of samples inside a froxel to a fixed set number to ensure dense cells do not cost more than sparse ones [S1, unofficial].
5.  **Shadow Casting:** Use the separate cascaded clip map for shadow calculations for out-of-view casters [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume froxel_grid is a 3D array of uint32
# emitter_list contains objects with: position, footprint_radius, extinction_coeff

def inject_particles(froxel_grid, emitter_list):
    for emitter in emitter_list:
        # Define footprint in froxel coordinates
        # Footprint is a 2x2x2 cube or sphere proportional to froxel size
        footprint_froxels = get_footprint_froxels(emitter.position, emitter.footprint_radius)
        
        # Normalize extinction by volume to prevent overshadowing
        # Extinction 1.0 maps to 2048u
        base_value = 2048
        normalized_extinction = emitter.extinction_coeff * base_value
        
        # Distribute by volume (simplified: divide by number of froxels in footprint)
        # Note: Exact normalization formula depends on specific volume definition
        value_per_froxel = normalized_extinction / len(footprint_froxels)
        
        for froxel in footprint_froxels:
            # Use atomic add to prevent overwrites from multiple emitters
            atomic_add(froxel_grid, froxel, value_per_froxel)

def render_volumetric_froxel(froxel_grid, ray):
    # Convert uint grid to f16 if not already done
    density_grid = convert_uint_to_f16(froxel_grid)
    
    # Raymarch with capped samples
    max_samples = 16 # Example cap
    samples = 0
    transmittance = 1.0
    
    while samples < max_samples:
        # March along ray
        pos = ray.origin + ray.direction * samples * step_size
        density = sample_density(density_grid, pos)
        
        # Beer-Lambert law for extinction
        extinction = density * step_size
        transmittance *= exp(-extinction)
        
        samples += 1
        
        if transmittance < epsilon:
            break
            
    return transmittance
```

**Parameters**
*   **Extinction Mapping:** Extinction 1.0 is mapped to 2048u in the uint volume before conversion to f16 [S1, unofficial].
*   **Footprint Size:** 2x2x2 cube for point or trilinear sampling, or a sphere [S1, unofficial].
*   **Cascaded Clip Map Levels:** 3 levels, camera-following, with world-space snapping [S1, unofficial].
*   **Sample Cap:** Fixed number of samples per froxel (value not specified in sources) [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Atomicity:** Without atomics, multiple emitters writing to the same froxel will overwrite each other's data [S1, unofficial].
*   **Overshadowing:** Large voxel cascades cause overshadowing if extinction is not normalized by volume, as particles write more often into the same voxels [S1, unofficial].
*   **Shadow Casting:** Clip-space volume alone is insufficient for out-of-view shadow casters; a separate cascaded clip map is required [S1, unofficial].
*   **Performance:** Dense cells must not cost more than sparse ones; this is achieved by capping the number of samples in the raymarch [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Injection cost is proportional to the number of particles and the footprint size in froxels, not screen area [S1, unofficial].
*   **Memory Complexity:** Memory usage is dominated by the froxel grid size and the number of particle emitters. The constraint is scattering-pass bandwidth, not footprint [S1, unofficial].
*   **Measured Performance:** On PS4 (Frostbite 2015), injecting 1k particles costs 0.03 ms (default) or 0.25 ms (high quality) [S1, unofficial]. Volumetric shadow map with 32³ raymarch costs 0.04 ms for spot lights and 0.14 ms for point lights [S1, unofficial].
*   **Scaling:** Cost scales with the number of particles and the complexity of the footprint. The fixed sample cap ensures that dense cells do not disproportionately increase cost [S1, unofficial].
*   **Strengths:** Nearly free injection cost; consistent lighting with global fog; avoids visible seams [S1, unofficial].
*   **Limitations:** Not suitable for high-fidelity 3D fluid simulation (which is unaffordable for real-time games except as a single hero effect) [S1, unofficial].
*   **When to Use:** For 95% of cases where volumetric particles are needed for effects like smoke, steam, or energy discharges [S1, unofficial].
*   **When Not to Use:** When high-fidelity 3D fluid simulation is required (use baked VDB/3D textures or offline rendering instead) [S1, unofficial].

## Background
Volumetric Particle Injection was shipped in Frostbite in 2015, referred to as "particle voxelization" [S1, unofficial]. It is also documented by Unreal Engine as a feature for Volumetric Fog and implemented in Godot [S1, unofficial]. Red Dead Redemption 2 (RDR2) uses this technique, keeping up to 32 volumes in the cluster grid with Z-binning [S1, unofficial]. The technique was developed to address the limitations of decoupled particle rendering, which caused lighting inconsistencies and visible seams [S1, unofficial].

## Sources
[S1] feat(render): volumetric particles in the froxel grid, https://github.com/lobinuxsoft/kooch/issues/1235, fetched 2026-10-26 (unofficial)
