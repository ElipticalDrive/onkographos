# Volumetric Photon Mapping

Also called: Photon Mapping in Participating Media, Real-time volume rendering for three-dimensional fetal ultrasound using volumetric photon mapping, Screen-space volumetric photon mapping, Volume Photon Map Creation, Volumetric Photon Mapping (VPM) for Fetal Ultrasound, Volumetric Photon Mapping (VPM) for Real-Time Fetal Ultrasound, Volumetric photon mapping with screen-space destiny estimation

## Overview
Volumetric Photon Mapping (VPM) is a global illumination technique for participating media that simulates light transport by storing volumetric scattering events as discrete photons and estimating in-scattered radiance via density estimation [S1, S2]. It solves the problem of computing indirect illumination (multiple scattering) in volumes, which is computationally expensive to integrate directly from the Radiative Transfer Equation (RTE) [S1]. Unlike standard ray-marching methods that require expensive integration of in-scattering terms per pixel, VPM decouples the light transport into a preprocessing phase (photon tracing) and a rendering phase (density estimation), allowing for efficient approximation of complex lighting effects like soft shadows and subsurface scattering [S1]. It sets itself apart from its siblings by generalizing surface photon mapping to volumes; specifically, it differs from [Beam Radiance Estimate](1283-beam-radiance-estimate.md) by using point-to-point density estimation rather than point-to-beam estimation, and from [Photon Beam Tracing](5401-photon-beam-tracing.md) by storing discrete interaction points rather than beam segments [S2].

## How it works
**Inputs:**
*   Volume dataset (e.g., 3D ultrasound, fog density field).
*   Light source positions and intensities.
*   Medium properties: absorption coefficient $\sigma_a$, scattering coefficient $\sigma_s$, extinction coefficient $\sigma_t = \sigma_a + \sigma_s$, and phase function $p(\omega_i, \omega_o)$ [S1].
*   Camera parameters (origin, direction, near/far planes).
*   Parameters: Total photon count $N_{photons}$, exposure $E$, search radius $r$ (or squared radius $r^2$), anisotropy coefficient $g$ [S1][S4, unofficial].

**Outputs:**
*   Rendered image with direct and indirect illumination.
*   Photon map data structure (volume texture or KD-tree) containing photon positions, directions, and power [S1].

**Data Structures:**
1.  **Photon Map:**
    *   *Traditional:* KD-tree storing tuples $(\mathbf{p}, \boldsymbol{\omega}, P)$, where $\mathbf{p}$ is position, $\boldsymbol{\omega}$ is incidence direction, and $P$ is power [S1].
    *   *GPU/Optimized Variant:* 3D Volume Texture. Stores flux/photon data directly in voxel space to avoid KD-tree construction and search overhead [S1].
2.  **Illumination Volumes:** Separate buffers for direct ($L_{ss}$) and indirect ($L_{ms}$) illumination to allow independent rendering [S1].

**Algorithm Steps:**

1.  **Preprocessing: Photon Emission and Tracing**
    *   Emit $N_{photons}$ photons from light sources [S1].
    *   For each photon, trace its path through the volume using a Markov random walk [S1, S3]:
        *   Cast a ray from the light source. Check for intersection with scene geometry or volume boundaries [S6, unofficial].
        *   If inside the volume, step through the medium with a constant step size $\Delta t$ (often with jitter) [S6, unofficial].
        *   At each step, determine interaction using the cumulative distribution function (CDF) of the extinction coefficient [S6, unofficial].
        *   If interaction occurs:
            *   Record the photon state: position $\mathbf{p}$, incoming direction $\boldsymbol{\omega}_{in}$, and current power $P$ [S1].
            *   Store this photon in the Photon Map [S1].
            *   Determine fate (absorption, scattering, or transmission) using Russian Roulette [S6, unofficial]. Termination probability is often set to 50% after 3 bounces [S6, unofficial].
            *   If scattered, sample a new direction $\boldsymbol{\omega}_{out}$ from the phase function $p(\boldsymbol{\omega}_{in}, \boldsymbol{\omega}_{out})$ [S1].
            *   Update power: $P_{new} = P_{old} \cdot \frac{\sigma_s}{\sigma_t} \cdot p(\boldsymbol{\omega}_{in}, \boldsymbol{\omega}_{out})$ [S1][S6, unofficial].
            *   Continue tracing with the new direction [S6, unofficial].
        *   If no interaction occurs over a step, advance the ray [S6, unofficial].
        *   If the ray hits a surface, handle surface interaction (reflection/absorption) and potentially store surface photons [S1][S6, unofficial].

2.  **Rendering: Direct Illumination**
    *   Render direct illumination ($L_{ss}$) separately using standard ray marching or ray casting [S1].
    *   Calculate single scattering at each sample point $x'$ along the camera ray:
        $$ L_{ss}(x', \omega_0) = \int_{\Omega} s(x', \omega_i, \omega_0) L_i(x', \omega_i) d\omega_i $$
        where $L_i$ is attenuated incident radiance and $s$ is the shading function [S1].
    *   Store result in a direct illumination volume [S1].

3.  **Rendering: Indirect Illumination (Gathering)**
    *   For each sample point $x_s$ along the camera ray, estimate the in-scattered radiance $L_{ms}(x', \omega_0)$ from the photon map [S1].
    *   *Standard Method (KD-Tree):* Query the $K$-nearest neighbors of $x_s$ in the photon map [S1].
    *   *Screen-Space/Volume Texture Method:* Use a novel screen-space destiny estimation or texture filtering to accelerate computation [S1].
    *   Density Estimation Formula:
        $$ L_{ms}(x') \approx \sum_{i=1}^{K} \frac{P_i}{\pi r^2} $$
        (Note: Specific kernel weights vary; some implementations use simple averaging or Gaussian kernels [S2]).
    *   The radiance reaching the camera sample $x_s$ is computed by integrating the RTE along the ray segment $[x_0, x_s]$ [S1]:
        $$ L(x_s, \omega_0) = L_0(x_0, \omega_0)T(x_0, x_s) + \int_{x_0}^{x_s} \left[ \lambda_a(x')L_e(x', \omega_0) + \lambda_s(x')L_s(x', \omega_0) \right] T(x', x_s) dx' $$
        where $L_s = L_{ss} + L_{ms}$ [S1].
    *   For discrete samples $x_j$ along the ray:
        $$ L_{x_n} = \sum_{j=1}^{n} \prod_{i=j}^{n-1} T(x_i, x_{i+1}) \int_{x_{j-1}}^{x_j} T(x', x_j) \sigma_s(x') L_{ms}(x') dx' $$
        where $T(x_i, x_j) = e^{-\int_{x_i}^{x_j} \tau_t dt}$ is the transmittance [S1].

4.  **Post-Processing**
    *   Combine direct and indirect illumination volumes [S1].
    *   Apply High Dynamic Range (HDR) tone mapping to handle intensity ranges exceeding display capabilities [S1].
    *   (Optional) Apply anisotropic diffusion filtering to reduce speckle noise in ultrasound data [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volumetric_photon_mapping(volume_data, lights, camera, num_photons, exposure, search_radius_sq):
    """
    Main pipeline for Volumetric Photon Mapping.
    """
    # 1. Preprocessing: Photon Tracing
    photon_map = [] # List of Photon objects: (position, direction, power)
    
    for _ in range(num_photons):
        photon = trace_photon_from_light(lights, volume_data)
        photon_map.append(photon)
        
    # 2. Rendering
    image = create_empty_image(camera.resolution)
    
    for pixel in camera.rays():
        ray = pixel.ray
        radiance = Vector3(0.0, 0.0, 0.0)
        
        # Ray march through volume
        t = ray.origin
        while t < ray.far_plane:
            sample_pos = ray.at(t)
            
            # Get medium properties at sample_pos
            sigma_a = volume_data.absorption(sample_pos)
            sigma_s = volume_data.scattering(sample_pos)
            sigma_t = sigma_a + sigma_s
            
            # Direct Illumination (Single Scattering)
            # Simplified: assume direct light contribution L_ss
            L_ss = compute_direct_illumination(sample_pos, lights, volume_data)
            
            # Indirect Illumination (Multiple Scattering) via Photon Map
            # Gather nearby photons
            L_ms = estimate_indirect_radiance(sample_pos, photon_map, search_radius_sq)
            
            # Total scattered radiance
            L_s = L_ss + L_ms
            
            # Transmittance from previous point to current
            # Simplified: assume previous transmittance T_prev is carried over
            # In full implementation, integrate T along the ray segment
            dt = ray.step_size
            transmittance_segment = exp(-sigma_t * dt)
            
            # Accumulate radiance contribution
            # L_out = L_in * T + (sigma_s * L_s / sigma_t) * (1 - T)
            # Note: This is a simplified discrete integration of RTE
            radiance_contribution = (sigma_s / sigma_t) * L_s * (1.0 - transmittance_segment)
            
            radiance += radiance_contribution * exposure # Apply exposure
            
            # Update transmittance for next step
            # (In a real implementation, maintain a running transmittance variable)
            
            t += dt
            
        image.set_pixel(pixel.coords, radiance)
        
    return image

def trace_photon_from_light(lights, volume_data):
    """
    Trace a single photon through the volume.
    """
    light = lights[0] # Simplified: single light
    position = light.position
    direction = random_direction_on_sphere()
    power = light.intensity
    
    photons = []
    
    for _ in range(100): # Max bounces
        # Check interaction
        if random() < volume_data.extinction_coefficient(position):
            # Store photon
            photons.append(Photon(position, direction, power))
            
            # Russian Roulette
            if random() > 0.5: # 50% termination chance after 3 bounces (simplified)
                break
                
            # Scatter
            new_direction = sample_phase_function(direction, volume_data.anisotropy_g)
            power *= volume_data.scattering_coefficient(position) / volume_data.extinction_coefficient(position)
            power *= phase_function_value(direction, new_direction, volume_data.anisotropy_g)
            
            direction = new_direction
            position += direction * step_size
        else:
            position += direction * step_size
            
    return photons

def estimate_indirect_radiance(query_point, photon_map, radius_sq):
    """
    Estimate radiance using density estimation from nearby photons.
    """
    total_radiance = Vector3(0.0, 0.0, 0.0)
    count = 0
    
    for photon in photon_map:
        dist_sq = length_sq(query_point - photon.position)
        
        if dist_sq < radius_sq:
            # Simple point-to-point estimator
            # Weight can be based on distance or normal alignment
            # Note: Some implementations use normal alignment, but for volume, 
            # simple distance weighting or uniform kernel is common.
            weight = 1.0 / radius_sq # Uniform kernel normalization
            
            # Add contribution
            total_radiance += photon.power * weight
            count += 1
            
    if count > 0:
        return total_radiance / count
    else:
        return Vector3(0.0, 0.0, 0.0)
```

**Parameters:**
*   $N_{photons}$: Total number of photons. Controls realism vs. speed. Higher counts improve quality but reduce FPS. Large datasets require higher counts [S1].
*   $r$ (or $r^2$): Search radius for gathering. Typical value $r=0.7$ (squared radius 0.49) [S4, unofficial].
*   $E$ (Exposure): Luminance scaling factor. Typical values 0.3–0.4 [S1].
*   $g$: Anisotropy coefficient for Henyey-Greenstein phase function. $g \in [-1, 1]$ [S1].
*   $K$: Number of nearest neighbors (if using KD-tree). Not explicitly defined for screen-space method [S1].
*   Samples: Number of samples per pixel. Fixed at 800 in tested datasets [S1].

**Edge Cases and Pitfalls:**
*   **Speckle Noise:** In ultrasound, speckle noise degrades gradient-based normal calculation. VPM is less sensitive to this than local illumination methods [S1].
*   **Storage Overhead:** Traditional KD-tree storage is memory-intensive and slow to search. Using volume textures mitigates this but requires GPU acceleration [S1].
*   **Thread Scheduling:** Emitting photons one-by-one on GPU causes idle time. Co-completing emission tasks with shared counters improves efficiency [S1].
*   **Insufficient Photons:** Large volumes with low photon counts produce unrealistic results [S1].
*   **Normal Weighting Bug:** In some implementations, the normal vector $N$ is initialized to zero, rendering normal-based weighting ineffective [S4, unofficial].

## Subtopics

### [Beam Radiance Estimate](1283-beam-radiance-estimate.md)
The Beam Radiance Estimate (BRE) is a photon-based technique for estimating accumulated in-scattered radiance along an entire camera ray in participating media, serving as the core gathering step for Volumetric Photon Mapping. Unlike conventional volumetric photon mapping, which estimates incident radiance at discrete points along a ray via spherical neighborhood queries, BRE queries all photons intersecting the ray line simultaneously, replacing Dirac delta and Heaviside functions with a 2D smoothing kernel perpendicular to the ray. This approach eliminates redundant photon gathering and noise associated with large step sizes in ray marching, concentrating computational effort only where photons intersect the viewing path.

### [Photon Emission and Storage](3880-photon-emission-and-storage.md)
Photon Emission and Storage is the data generation phase of Volumetric Photon Mapping, responsible for tracing photons from light sources through participating media and surfaces to populate a spatial data structure with scattering events. It solves the problem of approximating indirect illumination and inscattering by storing photon positions, directions, and energies. It differs from its sibling [Beam Radiance Estimate](1283-beam-radiance-estimate.md) by storing discrete point samples rather than beam-like structures, and from [Photon Beam Tracing](5401-photon-beam-tracing.md) by using standard ray-object intersection for storage rather than beam tracing through volumes [G].

### [Photon Beam Tracing](5401-photon-beam-tracing.md)
Photon Beam Tracing is a volumetric radiance estimation method where photon paths are stored as beam segments rather than discrete points, allowing for density estimation via beam intersection. It differs from standard volumetric photon tracing by storing the photon energy up to the point of intersection with solid geometry, rather than discarding the photon upon collision. This approach provides a superset of the information contained in a standard photon map, enabling more efficient handling of boundaries between participating media and opaque surfaces.

### [Progressive Photon Mapping](6672-progressive-photon-mapping.md)
Progressive Photon Mapping is a volumetric light transport method that addresses the memory bottlenecks of traditional photon-based estimation by replacing a single, massive volumetric photon map with a series of small, iterative photon maps. It builds on [Volumetric Photon Mapping](2190-volumetric-photon-mapping.md) by adopting a multipass strategy where photon contributions are updated incrementally, allowing for the efficient handling of participating media without exhausting system memory. It sets itself apart from siblings like [Beam Radiance Estimate](1283-beam-radiance-estimate.md) by using a global, iterative refinement of photon density rather than local beam-based approximations.

## Operating characteristics
*   **Time Complexity:** Preprocessing is $O(N_{photons} \cdot B)$, where $B$ is the average number of bounces per photon. Rendering is $O(N_{samples} \cdot K)$ for KD-tree gathering or $O(N_{samples})$ for texture-based gathering, where $K$ is the number of neighbors or texture lookup cost [S1, S2].
*   **Memory Complexity:** $O(N_{photons})$ for storing the photon map. Traditional KD-trees require significant memory; volume textures also consume VRAM proportional to the volume resolution [S1, S2].
*   **Measured Performance:** On Intel E5, 8GB RAM, GPU: Fetus 1 (289x212x266, 800 samples, exposure 0.4, 26 photons) -> 217 FPS; Fetus 2 (183x115x126, 800 samples, exposure 0.4, 22 photons) -> 222 FPS; Fetus 3 (209x175x209, 800 samples, exposure 0.3, 18 photons) -> 218 FPS; Fetus 4 (196x128x122, 800 samples, exposure 0.4, 20 photons) -> 220 FPS [S1]. Real-time performance is maintained even with up to 10 million photons [S1].
*   **Strengths:** Real-time capability with GPU acceleration; handles complex global illumination effects (soft shadows, subsurface scattering); robust against speckle noise in ultrasound [S1].
*   **Limitations:** Memory intensive for large volumes; requires GPU for real-time performance; photon count significantly impacts quality and speed; traditional KD-tree gathering is slow [S1, S2].
*   **When to Use:** Real-time visualization of participating media, especially when global illumination effects are critical and hardware acceleration is available (e.g., medical ultrasound, fog, smoke) [S1].
*   **When Not to Use:** CPU-only environments with large volumes; scenes requiring exact analytical solutions without approximation; extremely high dynamic range scenes without HDR tone mapping [S1].

## Background
Volumetric Photon Mapping was developed by Jensen and Christensen, who generalized the surface photon mapping algorithm by Jensen to light transport in participating media [S2]. The method involves tracing light in the volume and storing each volumetric interaction as a photon, then gathering contributions during the rendering pass using density estimation with a 3D kernel [S2]. Later, Jarosz et al. improved convergence by gathering along camera rays (point-to-beam) and replacing photons with photon beams [S2]. The specific application to real-time 3D fetal ultrasound was proposed by Qin et al., who introduced screen-space destiny estimation and volume texture storage to accelerate computation and avoid KD-tree overhead [S1].

## Sources
[S1] Real-time volume rendering for three-dimensional fetal ultrasound using volumetric photon mapping., https://doi.org/10.1186/s42492-024-00177-4, fetched 2026-10-26
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S3] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
[S4] jacklv123/Volumetric-photon-mapping, https://github.com/jacklv123/Volumetric-photon-mapping, fetched 2026-10-27 (unofficial)
[S5] The Beam Radiance Estimate for Volumetric Photon Mapping, http://graphics.ucsd.edu/~henrik/papers/beam_radiance_estimate/, fetched 2026-10-30
[S6] CS348b Final Project - Rendering Underwater Scenes, http://www.graphics.stanford.edu/courses/cs348b-competition/cs348b-04/underwater/index.html, fetched 2026-10-30 (unofficial)
