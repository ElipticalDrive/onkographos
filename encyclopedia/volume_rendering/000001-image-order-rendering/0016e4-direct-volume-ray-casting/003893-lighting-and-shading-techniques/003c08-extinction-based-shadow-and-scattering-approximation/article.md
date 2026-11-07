# Extinction-based Shadow and Scattering Approximation

Also called: Cone approximation via SAT, Extinction SAT queries

## Overview
This method approximates directional shadows and light scattering in volume rendering by computing the extinction of light intensity for voxels using a 3D Summed Area Table (SAT) [S1, unofficial]. It solves the problem of efficiently calculating soft shadows and ambient occlusion (AO) without propagating illumination from outside the volume or requiring expensive per-sample shadow rays [S1, unofficial]. Unlike [Direct Volume Rendering with Phong Shading](3686-direct-volume-rendering-with-phong-shading.md) which relies on local gradients, or [Illumination Caching](7548-illumination-caching.md) which may use different pre-computation strategies, this technique specifically models the attenuation of light through the volume by covering the light cone with axis-aligned cuboids [S1, unofficial]. It is distinct from [Kniss et al.'s half-angle slicing](https://www.ifi.uzh.ch/dam/jcr:ffffffff-9e7f-2af5-ffff-ffffcf9259bb/FastShadows.pdf) by handling any light type and multiple sources, and from [Ropinski et al.'s method](https://www.ifi.uzh.ch/dam/jcr:ffffffff-9e7f-2af5-ffff-ffffcf9259bb/FastShadows.pdf) by computing extinction rather than propagating external illumination [S1, unofficial].

## How it works
The method operates within the standard ray casting loop [Direct volume ray casting](19-direct-volume-ray-casting.md). It pre-computes shadow and AO terms into a light cache texture, which is then sampled during the ray casting pass [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Volume dataset $V$ with extinction coefficients $\sigma_t$.
    *   Light source position $\mathbf{l}$ and type (point/spot).
    *   Transfer function (TF) parameters (recomputes SAT if TF changes).
    *   Parameters: Cone angle $\Psi$, sampling frequency $f$, growth rate $g$, weighting function $h$.
*   **Outputs:**
    *   A 3D light cache texture storing shadow and AO/color bleeding terms.
    *   Shaded sample color $C_i$ during ray casting.

### Data Structures
*   **3D Summed Area Table (SAT):** A 3D texture where each voxel $(x,y,z)$ stores the cumulative sum of extinction coefficients from the origin $(0,0,0)$ to $(x,y,z)$ along the primary axis. The SAT allows $O(1)$ retrieval of the sum of coefficients within any axis-aligned box [S1, unofficial].
*   **Light Cache Texture:** A 3D texture storing the pre-computed shadow and AO terms. One additional texture lookup per sample is required during ray casting [S1, unofficial].

### Steps

1.  **Define Primary Axis:** Determine the primary axis for sampling by finding the coordinate axis (x, y, or z) with the smallest angle to the vector pointing from the voxel to the light source $\mathbf{l}$ [S1, unofficial]. This ensures the best coverage of the cone with axis-aligned cuboids [S1, unofficial].

2.  **Generate Sampling Points:** Define sampling points along the primary axis using a user-defined sampling frequency $f$ and growth rate $g$. The growth rate controls the change of frequency over distance, allowing fewer samples further away [S1, unofficial].

3.  **Derive Cuboid Queries:** For each sampling point, derive a cuboid that approximates the intersection of the light cone (defined by angle $\Psi$) and the volume. The cuboid is determined by the projection of the query cone onto the primary-secondary axis planes [S1, unofficial].

4.  **Compute Extinction via SAT:** Perform SAT lookups to sum the extinction coefficients for the derived cuboids. The sum represents the optical depth along the light path. Apply the weighting function $h$ to weight cuboids differently to strengthen or weaken the shadow effect [S1, unofficial].

5.  **Store in Light Cache:** Store the computed shadow and AO/color bleeding terms in the light cache texture. This avoids duplicate queries during rendering [S1, unofficial].

6.  **Ray Casting Pass:** During the volume ray casting loop, for each sample point $\mathbf{p}_i$, perform one additional texture lookup in the light cache to retrieve the illumination term $L(\mathbf{p}_i)$ [S1, unofficial].

### Formulas
The shaded sample color $C_i$ is computed as:
$$
C_i = C_{base}(\mathbf{p}_i) \cdot L(\mathbf{p}_i)
$$
where $L(\mathbf{p}_i)$ is retrieved from the light cache texture [S1, unofficial].

The extinction sum for a cuboid $B$ is retrieved from the SAT $S$ as:
$$
\text{Extinction}(B) = \sum_{v \in B} \sigma_t(v) \approx S(\mathbf{b}_{max}) - S(\mathbf{b}_{min})
$$
where $\mathbf{b}_{min}$ and $\mathbf{b}_{max}$ are the corners of the cuboid aligned with the primary axis [S1, unofficial]. Note: Exact SAT box sum formulas depend on the specific SAT implementation (e.g., inclusion-exclusion principle), but the source describes using SAT lookups to sum coefficients [S1, unofficial].

### Parameters
*   **Cone angle ($\Psi$):** Defines the angular spread of the shadow cone. Typical values: 1.0 to 5.0 degrees [S1, unofficial].
*   **Sampling frequency:** Defines the number of cuboids for approximation. Typically a few dozen lookups per cone and voxel are sufficient [S1, unofficial].
*   **Growth rate:** Controls the growth of cuboids (change of frequency over distance) [S1, unofficial].
*   **Weighting function ($h$):** Used to weight cuboids differently [S1, unofficial].

### Edge Cases and Pitfalls
*   **Cuboid Artifacts:** Using very few cone samples (e.g., 12) results in clearly visible cuboid artifacts [S1, unofficial].
*   **TF Changes:** The extinction SAT and AO/color bleeding terms must be recomputed every time the transfer function changes [S1, unofficial].
*   **Light Source Movement:** The terms for directional shadows must be recomputed every time the light source moves relative to the dataset [S1, unofficial].
*   **Light Sources Inside Volume:** The method trivially handles light sources even within or on the border of the dataset without additional effort [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-computation Phase
def compute_light_cache(volume, light_source, tf, params):
    """
    Pre-compute shadow and AO terms into a light cache texture.
    """
    # Recompute SAT if TF changes
    sat = build_3d_sat(volume, tf)
    
    light_cache = empty_3d_texture(volume.size)
    
    for each voxel v in volume:
        # Determine primary axis based on light source direction
        primary_axis = get_primary_axis(v, light_source.position)
        
        # Generate sampling points along primary axis
        points = generate_sampling_points(v, light_source, params.frequency, params.growth_rate)
        
        total_extinction = 0.0
        
        for point in points:
            # Derive cuboid from cone projection
            cuboid = derive_cuboid(point, light_source, params.cone_angle, primary_axis)
            
            # Sum extinction using SAT
            extinction = sum_extinction_in_cuboid(sat, cuboid)
            
            # Apply weighting function
            weight = params.weighting_function(cuboid)
            total_extinction += extinction * weight
            
        # Store result in light cache
        light_cache[v] = compute_shadow_term(total_extinction)
        
    return light_cache

# Ray Casting Phase
def ray_cast_sample(sample_point, light_cache):
    """
    Retrieve illumination term for a sample point.
    """
    # One additional texture lookup
    illumination = light_cache[sample_point]
    return illumination
```

## Subtopics

### Ambient Occlusion/Color Bleeding
The same SAT structure is used to compute AO by approximating the neighborhood of every voxel with cuboid shells. This is stored in the same light cache texture [S1, unofficial].

### Multiple Light Sources
The method can handle multiple light sources (point and spot) easily, computing terms for each and storing them in the cache [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** SAT construction is $O(N)$ where $N$ is the number of voxels. Shadow computation per frame is $O(N \cdot K)$ where $K$ is the number of cuboids per voxel (typically a few dozen) [S1, unofficial].
*   **Memory Complexity:** Requires storage for the 3D SAT and the light cache texture, both of size proportional to the volume resolution [S1, unofficial].
*   **Measured Performance:**
    *   SAT construction for $256^3$ volume: 0.311s [S1, unofficial].
    *   AO/color bleeding computation for $256^3$ texture with 3 shells: 0.0307s [S1, unofficial].
    *   Shadow computation for $64^3$ texture with 50 samples: 0.0079s/frame [S1, unofficial].
*   **Strengths:** Fast, flexible, handles any light type and multiple sources, handles light sources inside the volume, soft realistic shadows [S1, unofficial].
*   **Limitations:** Requires recomputation if TF or light source moves, cuboid artifacts if samples are too low [S1, unofficial].
*   **When to use:** Interactive volume rendering with dynamic transfer functions and light sources [S1, unofficial].

## Background
The method was described in "Extinction-based Shading and Illumination in GPU Volume Ray-Casting" by the University of Zurich (UZH) [S1, unofficial]. It was developed to provide interactive performance for volume rendering with realistic lighting effects, avoiding the high cost of classical shadow rays [S1, unofficial].

## Sources
[S1] PDF Extinction-based Shading and Illumination in GPU Volume Ray-Casting - UZH, https://www.ifi.uzh.ch/dam/jcr:ffffffff-9e7f-2af5-ffff-ffffcf9259bb/FastShadows.pdf, fetched 2026-10-30 (unofficial)
