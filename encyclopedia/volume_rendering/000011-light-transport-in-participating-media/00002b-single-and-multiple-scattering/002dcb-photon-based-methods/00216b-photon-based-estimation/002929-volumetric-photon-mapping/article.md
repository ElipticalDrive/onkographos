# Volumetric Photon Mapping

Also called: Photon Mapping in Participating Media, Screen-space volumetric photon mapping, Volume Photon Map Creation, Volumetric Photon Mapping (VPM) for Fetal Ultrasound, Volumetric photon mapping with screen-space destiny estimation

## Overview
Volumetric Photon Mapping (VPM) is a global illumination technique for participating media that simulates light transport by decoupling the simulation of indirect lighting from the final image synthesis [S1, S2]. It solves the problem of efficiently capturing complex scattering effects in participating media by storing photon interactions in a volume texture (or KD-tree) and estimating radiance during a subsequent gathering pass [S1, S2]. Unlike direct integration methods that require expensive Monte Carlo sampling at render time, VPM pre-computes photon paths, and it is distinguished from its sibling [Beam Radiance Estimate](1283-beam-radiance-estimate.md) by storing discrete photon packets rather than continuous beam radiance estimates [S1, S2].

## How it works
Volumetric Photon Mapping operates in two distinct phases: a preprocessing phase to build the photon map, and a rendering phase to estimate radiance [S2, S3].

### Inputs and Outputs
*   **Inputs:** Scene geometry, participating media properties (extinction $\sigma_t$, scattering $\sigma_s$, absorption $\sigma_a$, phase function $p$), light source positions and intensities, and camera parameters [G].
*   **Outputs:** A photon map data structure containing stored photon interactions [S6], and a final rendered image with direct and indirect illumination [S1].

### Data Structures
The core data structure is the **Photon Map** [S1]. While traditional implementations use KD-trees for nearest-neighbor queries [S1][S4], specific real-time implementations for ultrasound data store photons in a **3D Volume Texture** to minimize storage space and accelerate gathering by avoiding time-consuming tree traversals [S1].

**Volume Texture Layout:**
*   Stores photon position, incidence direction, and power (flux) [S1].
*   Dimensions match the volume resolution or a lower-resolution grid [S2].

**Standard Photon Map Layout (KD-Tree variant):**
*   Each node stores: Position, Direction, and Power (flux) [S1].

### Preprocessing Phase: Photon Emission and Storage
1.  **Photon Emission:** Photons are emitted from light sources into the volume. To prevent wastage, photons are cast onto the volume’s boundary voxels [S1].
2.  **Photon Tracing (Markov Random Walk):**
    *   A photon travels through the medium. The step size $t$ is sampled from the extinction coefficient $\sigma_t$:
        $$ t = -\frac{\ln(r)}{\sigma_t} $$
        where $r$ is a uniform random number in $[0, 1]$ [G].
    *   At each step, the photon's fate is determined by the scattering albedo $\omega = \sigma_s / \sigma_t$ [G].
    *   **Scattering:** If scattered, the photon's direction is updated using the phase function $p(\omega_i, \omega_o)$, and its energy is attenuated by $\omega$ [S1]. The Henyey-Greenstein phase function is commonly used:
        $$ p(\theta) = \frac{1-g^2}{(1+g^2-2g\cos\theta)^{2/3}} $$
        where $g \in [-1, 1]$ is the anisotropy coefficient [S1].
    *   **Absorption:** The photon is terminated or its energy is reduced [G].
    *   **Null Collisions:** In heterogeneous media, a majorant $\sigma_{max} \ge \sigma_t(\mathbf{x})$ is used [G].
    *   **Russian Roulette:** After a certain number of interactions (e.g., three), the photon may be terminated with a probability (e.g., 50%) to limit path length [S6].
3.  **Storage:**
    *   When a photon interacts with the volume (scatters or is absorbed), its state is recorded [S1].
    *   Data recorded: Position $\mathbf{x}$, incoming direction $\omega_i$, and power $P$ [S1].
    *   These are stored in the Photon Map (Volume Texture or KD-Tree) [S1][S4].
    *   **GPU Optimization:** To improve emission efficiency, threads co-complete emission tasks using a shared counter `Count`, initialized to 0, incremented by 1 each time a thread emits a photon, until it equals the total number of photons [S1].

### Rendering Phase: Density Estimation and Integration
1.  **Ray Casting:** For each pixel, a camera ray is cast into the volume [S1].
2.  **Direct Illumination:** Calculated separately via single scattering (SS) [S1].
    $$ L_{ss}(x', \omega_0) = \int_{\Omega} s(x', \omega_i, \omega_0) L_i(x', \omega_i) d\omega_i $$
    where $s$ is the material shading function and $L_i$ is attenuated incident radiance [S1].
3.  **Indirect Illumination (Gathering):**
    *   The ray is sampled at discrete points $x_j$ [S1].
    *   For each sample point, nearby photons are gathered to estimate in-scattered radiance $L_{ms}$ (multiple scattering) [S3].
    *   **Screen-Space Destiny Estimation:** A novel method proposed for real-time ultrasound rendering to accelerate computation. It avoids complicated storage structures by estimating radiance in screen space [S1].
    *   **Standard Gathering (Point-to-Point):**
        $$ L_{ms}(x') \approx \frac{1}{N} \sum_{i=1}^{N} \frac{P_i}{V_i} K\left(\frac{\|x' - p_i\|}{r}\right) p(\omega_i, \omega_0) $$
        where $N$ is the number of photons in search radius $r$, $P_i$ is photon power, $V_i$ is volume of influence, $K$ is a kernel function, and $p$ is the phase function [G].
    *   **Implementation Note (Pseudocode Logic):** In some implementations, a search radius `sqRadius` (e.g., 0.7) is used [S5, unofficial]. Photons within this radius contribute to the energy accumulator [S5, unofficial]. The weight may be calculated based on distance and normal alignment, though specific implementations may simplify this (e.g., overwriting weight with `power/num_vol`) [S4].
4.  **Integration:**
    *   The total radiance along the ray is integrated using transmittance $T$:
        $$ L(x_s, \omega_0) = L_0(x_0, \omega_0)T(x_0, x_s) + \int_{x_0}^{x_s} \lambda_s(x') L_s(x', \omega_0) T(x', x_s) dx' $$
        where $\lambda_s$ is the scattering coefficient and $L_s = L_{ss} + L_{ms}$ [S1].
    *   Transmittance between samples $x_i$ and $x_j$:
        $$ T(x_i, x_j) = e^{-\int_{x_i}^{x_j} \tau_t dt} $$
        where $\tau_t$ is the extinction coefficient [S1].
    *   For discrete samples $x_j$:
        $$ L_{x_n} = \sum_{j=1}^{n} \prod_{i=j}^{n-1} T(x_i, x_{i+1}) \int_{x_{j-1}}^{x_j} T(x', x_j) \sigma_s(x') L_{ms}(x') dx' $$
        [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volumetric_photon_mapping(scene, camera, num_photons):
    # Phase 1: Preprocessing
    photon_map = []  # List of (position, direction, power)
    count = 0
    
    for _ in range(num_photons):
        current_energy = scene.light_power
        current_pos = scene.light_origin
        current_dir = scene.light_direction
        photon_bounces = 0
        photon_alive = True
        
        while current_energy > 0 and photon_alive:
            sigma_t = scene.get_extinction(current_pos)
            if sigma_t == 0:
                break
            r = random.uniform(0, 1)
            step = -math.log(r) / sigma_t
            
            next_pos = (current_pos[0] + current_dir[0] * step,
                        current_pos[1] + current_dir[1] * step,
                        current_pos[2] + current_dir[2] * step)
            
            if is_inside_volume(next_pos, scene.bounds):
                sigma_s = scene.get_scattering(current_pos)
                sigma_a = scene.get_absorption(current_pos)
                sigma_t = sigma_s + sigma_a
                
                if photon_bounces > 3:
                    if random.random() > 0.5:
                        photon_alive = False
                        break
                
                if random.random() < sigma_s / sigma_t:
                    photon_map.append((current_pos, current_dir, current_energy))
                    # [G] Sample new direction using Henyey-Greenstein phase function
                    g = scene.anisotropy
                    current_dir = rotate_direction(current_dir, g)
                    current_energy *= sigma_s / sigma_t
                    photon_bounces += 1
                else:
                    photon_alive = False
                    break
            else:
                handle_surface_interaction(current_pos, current_dir, current_energy)
                break
            
            current_pos = next_pos
        count += 1

    # Phase 2: Rendering
    image = create_image(camera.resolution)
    
    for pixel in camera.rays:
        ray_origin = pixel.origin
        ray_direction = pixel.direction
        direct_L = calculate_direct_scattering(ray_origin, ray_direction, scene)
        
        indirect_L = 0.0
        t = 0.0
        step_size = scene.volume_step
        
        while t < ray.max_distance:
            sample_pos = (ray_origin[0] + ray_direction[0] * t,
                          ray_origin[1] + ray_direction[1] * t,
                          ray_origin[2] + ray_direction[2] * t)
            
            gathered_energy = 0.0
            search_radius = 0.7
            search_radius_sq = search_radius * search_radius
            
            nearby_photons = [p for p in photon_map if distance_sq(sample_pos, p[0]) < search_radius_sq]
            
            for photon in nearby_photons:
                # [G] Weight calculation; implementations often overwrite normal-based weight with power/num_photons
                weight = photon[2] / len(nearby_photons)
                gathered_energy += weight
            
            indirect_L += gathered_energy * scene.scattering_coefficient(sample_pos) * step_size
            t += step_size
            
        end_pos = (ray_origin[0] + ray_direction[0] * t,
                   ray_origin[1] + ray_direction[1] * t,
                   ray_origin[2] + ray_direction[2] * t)
        transmittance = calculate_transmittance(ray_origin, end_pos, scene)
        final_radiance = direct_L * transmittance + indirect_L * transmittance
        
        image[pixel] = apply_hdr(final_radiance, exposure=0.4)
        
    return image

# [G] Helper functions for volume bounds, distance, direct scattering, transmittance, HDR, and image creation are standard and omitted for brevity.
```

### Parameters
*   **Anisotropy coefficient $g$:** Controls scattering direction. $g \in [-1, 1]$. Negative for back-scattering, 0 for isotropic, positive for forward-scattering [S1].
*   **Exposure:** Luminance value assigned to each photon. Typical values: 0.3 to 0.4 [S1].
*   **Number of Photons:** Controls realism vs. speed. Higher counts improve realism but reduce FPS. Large datasets require higher photon counts to avoid unrealistic results [S1].
*   **Samples:** Number of samples per ray. Fixed at 800 in tested datasets [S1].
*   **Search Radius:** Used in gathering. Example value: 0.7 [S5, unofficial].

### Edge Cases and Pitfalls
*   **Speckle Noise:** In ultrasound images, speckle noise degrades signal-to-noise ratio and makes gradient-based normal calculation unreliable. VPM is less sensitive to this than local illumination methods [S1].
*   **Storage Overhead:** Traditional KD-tree storage is memory-intensive and slow for gathering. Using volume textures mitigates this by avoiding time-consuming tree traversals [S1].
*   **GPU Thread Scheduling:** Creating a thread per photon can lead to resource wastage. Co-completing emission tasks with a shared counter improves efficiency [S1].
*   **Normal Weighting Bug:** In some implementations, the normal vector $N$ is initialized to zero, rendering normal-based weighting ineffective. The weight is overwritten by `power/num_vol` [S5, unofficial].
*   **Emission Term:** For fetal ultrasound, emission is often ignored, simplifying the radiance transport equation [S1].

## Subtopics

### [Beam Radiance Estimate](1283-beam-radiance-estimate.md)
The Beam Radiance Estimate (BRE) is a photon-based technique for estimating accumulated in-scattered radiance along an entire camera ray in participating media, serving as the core gathering step for Volumetric Photon Mapping. Unlike conventional volumetric photon mapping, which estimates incident radiance at discrete points along a ray via spherical neighborhood queries, BRE queries all photons intersecting the ray line simultaneously, replacing Dirac delta and Heaviside functions with a 2D smoothing kernel perpendicular to the ray. This approach eliminates redundant photon gathering and noise associated with large step sizes in ray marching, concentrating computational effort only where photons intersect the viewing path. Choose it when ray-aligned density estimation is preferred over point-wise queries.

### [Photon Emission and Storage](3880-photon-emission-and-storage.md)
Photon Emission and Storage is the preprocessing phase of Volumetric Photon Mapping that generates photon paths from light sources and records their positions, directions, and energy densities into spatial data structures for both surface intersections and volumetric scattering events. It solves the problem of efficiently capturing indirect lighting and caustics in participating media by decoupling the complex light transport simulation from the final image synthesis, allowing for density estimation during a subsequent gather pass. Unlike direct integration methods, it pre-computes photon interactions, and it is distinguished from its sibling [Beam Radiance Estimate](1283-beam-radiance-estimate.md) by storing discrete photon packets rather than continuous beam radiance estimates. Choose it when decoupled preprocessing is required for complex media.

### [Progressive Photon Mapping](6672-progressive-photon-mapping.md)
Progressive Photon Mapping is a volumetric light transport method developed by Hachisuka et al. that uses a multipass rendering technique to continuously emit photons into the image, gradually reducing inaccuracy. It builds on [Volumetric Photon Mapping](2190-volumetric-photon-mapping.md) by applying a multipass strategy to photon emission and accumulation. It differs from sibling methods like [Beam Radiance Estimate](1283-beam-radiance-estimate.md) by refining illumination through iterative photon passes rather than beam-based approximations [G]. Choose it when memory is constrained and iterative convergence is acceptable.

### [Volumetric Photon Mapping with Radiance Transfer](6673-volumetric-photon-mapping-with-radiance-transfer.md)
Volumetric Photon Mapping with Radiance Transfer is a rendering technique that integrates volumetric photon mapping into a pre-designed radiance transfer procedure to achieve real-time, realistic volume rendering. It distinguishes itself from standard volumetric photon mapping by explicitly separating direct and indirect lighting computations within a radiance transfer framework, allowing for effective updates to the photon map when transfer functions change. Unlike [Beam Radiance Estimate](1283-beam-radiance-estimate.md) which focuses on density estimation via beam kernels, this method prioritizes the structural integration of photon data into a transfer-based rendering pipeline to handle wide intensity ranges. Choose it when dynamic transfer function updates are required.

## Operating characteristics
*   **Time Complexity:** Preprocessing scales linearly with the number of photons and medium traversal steps [G]. Rendering complexity scales with the number of rays, samples per ray, and the cost of gathering photons from the volume texture or KD-tree [S1].
*   **Memory Complexity:** Memory complexity scales with the number of stored photons, requiring a huge volumetric photon map [S3]. Volume textures reduce overhead compared to KD-trees but still require significant memory for high-resolution volumes [S1].
*   **Measured Performance:** On Intel E5, 8GB RAM, GPU: Fetus 1 (289x212x266, 800 samples, exposure 0.4, 17 photons) -> 217 FPS; Fetus 2 (183x115x126, 800 samples, exposure 0.4, 22 photons) -> 222 FPS; Fetus 3 (209x175x209, 800 samples, exposure 0.3, 18 photons) -> 218 FPS; Fetus 4 (196x128x122, 800 samples, exposure 0.4, 20 photons) -> 220 FPS. Real-time performance is maintained even when photon count increases to ten million [S1].
*   **Strengths:** Simulates realistic global illumination effects (scattering, shadows) in participating media; handles heterogeneous media; less sensitive to speckle noise in ultrasound images compared to local illumination methods [S1].
*   **Limitations:** High memory usage for large photon maps; requires GPU acceleration for real-time performance; insufficient photons on large volumes produce unrealistic results; traditional KD-tree gathering is slow [S1, S2].
*   **When to Use:** Real-time volume rendering of participating media where global illumination effects are critical, such as medical ultrasound visualization or foggy scenes [S1].

## Background
Volumetric Photon Mapping was developed as an extension of the original photon mapping algorithm by Jensen [29], which was designed for surface global illumination [S1]. Jensen and Christensen generalized the algorithm from light travel between object surfaces to light transport in participating media by storing volumetric interactions as photons and estimating density using a 3D kernel [S2]. Later, Jarosz et al. improved convergence by gathering along camera rays (point-to-beam) and replacing photons with photon beams [S2]. The method has been applied to real-time volume rendering for 3D fetal ultrasound to overcome the limitations of local illumination methods, which produce plastic-like visuals and fail to differentiate tissues due to noise sensitivity [S1].

## Sources
[S1] Real-time volume rendering for three-dimensional fetal ultrasound using volumetric photon mapping., https://doi.org/10.1186/s42492-024-00177-4, fetched 2026-10-26
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S3] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
[S4] jacklv123/Volumetric-photon-mapping, https://github.com/jacklv123/Volumetric-photon-mapping, fetched 2026-10-27
[S5] The Beam Radiance Estimate for Volumetric Photon Mapping, http://graphics.ucsd.edu/~henrik/papers/beam_radiance_estimate/, fetched 2026-10-30 (unofficial)
[S6] CS348b Final Project - Rendering Underwater Scenes, http://www.graphics.stanford.edu/courses/cs348b-competition/cs348b-04/underwater/index.html, fetched 2026-10-30
