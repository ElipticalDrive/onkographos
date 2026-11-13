# Extinction-based Shadow and Scattering Approximation

Also called: Cone approximation via SAT, Extinction SAT queries

## Overview
This method approximates directional shadows and light scattering in volume rendering by computing the extinction of light intensity for voxels using a 3D Summed Area Table (SAT) [S1, unofficial]. It solves the problem of efficiently calculating soft shadows and ambient occlusion (AO) without propagating illumination from outside the volume or requiring hundreds of explicit shadow ray samples [S1, unofficial]. Unlike half-angle slicing [Sibling: Half-angle Slicing], it handles any light type and multiple sources; unlike Ropinski et al.'s method [Sibling: Illumination Caching], it computes extinction directly within the volume, allowing light sources to be placed inside or on the border of the dataset without special handling [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scalar volume dataset $V$, transfer function $TF$, light source positions and types, camera parameters.
*   **Output:** Precomputed light cache texture containing shadow and AO/color bleeding terms, applied during raycasting.

**Data Structures:**
*   **3D Summed Area Table (SAT):** A 3D texture where each texel stores the cumulative sum of extinction coefficients along the three axes. The SAT is constructed from the volume data mapped via the transfer function [S1, unofficial].
*   **Light Cache Texture:** A 3D texture storing precomputed shadow and AO terms. One additional texture lookup per sample is performed during raycasting [S1, unofficial].

**Parameters:**
*   **Cone angle ($\Psi$):** Defines the angular spread of the shadow cone. Typical values: $1.0^\circ$ to $5.0^\circ$ [S1, unofficial].
*   **Sampling frequency:** Number of cuboids used for approximation. Typically a few dozen lookups per cone and voxel are sufficient [S1, unofficial].
*   **Growth rate:** Controls the change of sampling frequency over distance to adjust density [S1, unofficial].
*   **Weighting function ($h$):** Weights cuboids differently to strengthen or weaken shadow effects [S1, unofficial].

**Steps:**
1.  **Construct SAT:** Build a 3D SAT of extinction coefficients from the volume data. This is done once per transfer function change or light source movement [S1, unofficial].
2.  **Determine Primary Axis:** For each voxel and light source, define the primary axis for sampling as the coordinate axis with the smallest angle to the vector pointing to the light source [S1, unofficial].
3.  **Define Sampling Points:** Generate sampling points along the primary axis using the user-defined sampling frequency and growth rate [S1, unofficial].
4.  **Derive Cuboid Queries:** For each sampling point, derive axis-aligned cuboid queries based on the projection of the query cone onto the primary-secondary axis planes. This ensures the best possible coverage of the cone with axis-aligned lookups [S1, unofficial].
5.  **Perform SAT Lookups:** Use the SAT to sum extinction coefficients for the derived cuboids covering the cone. Apply the weighting function $h$ if used [S1, unofficial].
6.  **Store Results:** Store the computed shadow and ambient occlusion/color bleeding terms in the light cache texture [S1, unofficial].
7.  **Raycasting:** During the direct volume ray casting pass, apply illumination terms using one additional texture lookup per sample [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_extinction_sat(volume, transfer_function):
    """
    Constructs a 3D Summed Area Table (SAT) for extinction coefficients.
    volume: 3D array of scalar values.
    transfer_function: Function mapping scalar value to extinction coefficient.
    Returns: 3D SAT array.
    """
    # Map volume to extinction coefficients
    extinction = apply_transfer_function(volume, transfer_function)
    
    # Compute 3D SAT
    # SAT[x, y, z] = sum of extinction[i, j, k] for 0<=i<=x, 0<=j<=y, 0<=k<=z
    sat = compute_3d_cumulative_sum(extinction)
    return sat

def compute_shadow_term(sat, voxel_pos, light_dir, cone_angle, sampling_freq, growth_rate, weighting_func):
    """
    Computes the shadow term for a voxel using SAT queries.
    sat: 3D SAT array.
    voxel_pos: (x, y, z) position of the voxel.
    light_dir: Unit vector from voxel to light source.
    cone_angle: Angular spread of the shadow cone (Psi).
    sampling_freq: Base sampling frequency.
    growth_rate: Rate of change of sampling frequency over distance.
    weighting_func: Function to weight cuboids.
    Returns: Shadow term value.
    """
    # Determine primary axis (axis with smallest angle to light_dir)
    primary_axis = argmin_angle(light_dir)
    
    # Define sampling points along primary axis
    # Distance from voxel to light source
    dist = distance(voxel_pos, light_source_pos)
    
    # Sampling points depend on frequency and growth rate
    # This is a simplified representation; actual implementation requires precise geometric derivation
    sampling_points = []
    current_freq = sampling_freq
    for i in range(num_samples):
        # Calculate position along primary axis based on growth rate
        pos = voxel_pos + i * step_size * primary_axis_vector
        sampling_points.append(pos)
        # Update frequency based on growth rate
        current_freq *= (1 + growth_rate)
        
    shadow_sum = 0.0
    for point in sampling_points:
        # Derive cuboid query from point and cone projection
        cuboid = derive_cuboid(point, light_dir, cone_angle, primary_axis)
        
        # Perform SAT lookup for cuboid
        extinction_sum = sat_lookup(sat, cuboid)
        
        # Apply weighting function
        weight = weighting_func(point, dist)
        shadow_sum += weight * extinction_sum
        
    return shadow_sum

def raycasting_pass(volume, sat, light_cache, transfer_function, lighting_model):
    """
    Performs direct volume ray casting with extinction-based shading.
    """
    for each pixel:
        cast_ray()
        color = 0.0
        alpha = 0.0
        for each sample along ray:
            # Get base color and opacity from transfer function
            base_color, opacity = transfer_function(volume[sample_pos])
            
            # Get precomputed shadow/AO term from light cache
            shadow_term = light_cache[sample_pos]
            
            # Compute lighting term (e.g., Phong) modulated by shadow
            # Note: The exact integration with the parent's Phong model is not detailed in [S1, unofficial]
            # but [S1, unofficial] states one additional texture lookup is sufficient.
            # Assuming shadow_term modulates the diffuse/specular components.
            light_intensity = lighting_model(sample_pos, shadow_term)
            
            shaded_color = base_color * light_intensity
            
            # Front-to-back compositing
            color += shaded_color * alpha * (1 - alpha)
            alpha += opacity * (1 - alpha)
            
        return color
```

**Edge Cases and Pitfalls:**
*   Using very few cone samples (e.g., 12) results in clearly visible cuboid artifacts [S1, unofficial].
*   The SAT size can be set at a fraction of the volume dataset size, but lower sizes result in more blurred shadows [S1, unofficial].
*   The method requires recomputation of the SAT and AO/shadow terms when the transfer function changes or light sources move relative to the dataset [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** SAT construction time scales with volume size. For a $256^3$ volume, SAT construction takes 0.311s; for $64^3$, it takes 0.029s [S1, unofficial]. AO/color bleeding computation for a $256^3$ texture with 3 shells takes 0.0307s, compared to 3.0818s for explicit sampling of radius 7 [S1, unofficial]. Shadow computation for a $64^3$ texture with 50 samples takes 0.0079s/frame [S1, unofficial].
*   **Memory Complexity:** Requires storage for the 3D SAT and the light cache texture. The SAT size can be smaller than the volume dataset [S1, unofficial].
*   **Performance:** Interactive performance is achieved. The method is significantly faster than explicit shadow ray sampling (hundreds of samples) [S1, unofficial].
*   **Strengths:** Fast, flexible, handles any light type and multiple sources, allows light sources inside the volume, low cost for soft shadows [S1, unofficial].
*   **Limitations:** Requires recomputation on TF or light source changes, potential cuboid artifacts with low sampling [S1, unofficial].

## Background
This method was introduced in the paper "Extinction-based Shading and Illumination in GPU Volume Ray-Casting" by UZH researchers [S1, unofficial]. It was developed to provide interactive volume rendering with dynamic transfer functions and light sources, addressing the limitations of previous methods like half-angle slicing and Ropinski et al.'s approach [S1, unofficial]. It is used in GPU ray-casters for real-time visualization [S1, unofficial].

## Sources
[S1] Extinction-based Shading and Illumination in GPU Volume Ray-Casting, https://www.ifi.uzh.ch/dam/jcr:ffffffff-9e7f-2af5-ffff-ffffcf9259bb/FastShadows.pdf, fetched 2026-10-30 (unofficial)
