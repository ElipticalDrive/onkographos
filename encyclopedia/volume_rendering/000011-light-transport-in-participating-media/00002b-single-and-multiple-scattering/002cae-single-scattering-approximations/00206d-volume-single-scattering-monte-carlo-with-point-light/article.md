# Volume Single Scattering (Monte Carlo with Point Light)

Also called: Single Scattering Volume Integration

## Overview
This method computes the radiance contribution of a single scattering event in a participating medium using Monte Carlo integration with uniform sampling along the ray distance, specifically adapted for a point light source [S1, unofficial]. It solves the radiative transfer equation by discretizing the ray into fixed steps or random samples, summing the emitted light attenuated by transmittance from the eye and to the light source [S1, unofficial]. Unlike deterministic ray marching approaches [2318-ray-marching-with-in-out-scattering-and-density-scaling.md], this method relies on stochastic sampling to estimate the integral, making it unbiased but potentially noisy without sufficient samples. It is a foundational technique for understanding volumetric light transport before introducing multiple scattering or complex light sources [S2].

## How it works
**Inputs and Outputs**
*   **Input:** Ray origin $\mathbf{o}$ and direction $\boldsymbol{\omega}$; homogeneous volume coefficients $\sigma_s$ (scattering) and $\sigma_t$ (extinction); point light position $\mathbf{l}$ and emitted radiance $L_e$; background radiance $L_b$; number of Monte Carlo samples $N_{samples}$ [S1, unofficial].
*   **Output:** Accumulated radiance $L_o$ at the camera pixel [S1, unofficial].

**Data Structures**
*   **Volume Properties:** Homogeneous coefficients are constant throughout the medium. $\sigma_t = \sigma_a + \sigma_s$ [G].
*   **Light Model:** Point light defined by position $\mathbf{l}$ and intensity $L_e$. The incident radiance at a point $\mathbf{x}$ from the light is $L_i(\mathbf{x}, \mathbf{l}-\mathbf{x}) = L_e \cdot \delta(\text{direction})$, effectively treated as a constant source term attenuated by distance [S1, unofficial].

**Formulas**
The single scattering radiance $L_{ss}$ at the eye is the sum of the attenuated background radiance and the in-scattered light from the volume [S1, unofficial]:

$$ L_o = L_b \cdot T(\mathbf{o}, \mathbf{x}_{hit}) + \int_{t_0}^{t_1} L_e \cdot T(\mathbf{x}(t), \mathbf{l}) \cdot \sigma_s \cdot p(\boldsymbol{\omega}, \mathbf{l}-\mathbf{x}(t)) \cdot T(\mathbf{o}, \mathbf{x}(t)) \, dt $$

Where:
*   $T(\mathbf{a}, \mathbf{b}) = e^{-\sigma_t \|\mathbf{b} - \mathbf{a}\|}$ is the transmittance between points $\mathbf{a}$ and $\mathbf{b}$ [G].
*   $p(\cdot)$ is the phase function value [S1, unofficial].
*   $t_0$ is the entry point into the volume, $t_1$ is the exit point or background intersection [S1, unofficial].

**Parameters**
*   $N_{samples}$: Number of Monte Carlo samples. Typical values range from 1 to 16+ depending on noise tolerance [S1, unofficial].
*   $\sigma_s, \sigma_t$: Scattering and extinction coefficients [S1, unofficial].

**Algorithm Steps**
1.  **Calculate Background Intersection:** Determine the hit point $\mathbf{x}_{hit}$ on the background surface or volume boundary using the ray equation $\mathbf{x}_{hit} = \mathbf{o} + \boldsymbol{\omega} \cdot t_{max}$ [S1, unofficial].
2.  **Calculate Background Transmittance:** Compute the transmittance from the ray origin to the hit point: $T_{eye} = e^{-\sigma_t \|\mathbf{x}_{hit} - \mathbf{o}\|}$ [S1, unofficial].
3.  **Initialize Accumulator:** Set the initial indirect radiance $L_i = L_b \cdot T_{eye}$ [S1, unofficial]. Initialize scattered radiance accumulator $L_s = 0$ [S1, unofficial].
4.  **Monte Carlo Loop:** For $i = 0$ to $N_{samples} - 1$:
    a.  **Sample Distance:** Sample a distance $t_{sample}$ uniformly along the ray segment within the volume (or from origin to $t_{max}$ depending on implementation, typically uniform in $[0, t_{max}]$ or $[t_0, t_1]$) [S1, unofficial]. *Note: Some implementations use uniform distance sampling over the entire ray length for simplicity in homogeneous media [S1, unofficial].*
    b.  **Calculate Position:** $\mathbf{x}_{sample} = \mathbf{o} + \boldsymbol{\omega} \cdot t_{sample}$ [S1, unofficial].
    c.  **Calculate Eye Transmittance:** $T_{eye\_sample} = e^{-\sigma_t \|\mathbf{x}_{sample} - \mathbf{o}\|}$ [S1, unofficial].
    d.  **Calculate Light Transmittance:** Compute distance to light $d_{light} = \|\mathbf{l} - \mathbf{x}_{sample}\|$. $T_{light} = e^{-\sigma_t d_{light}}$ [S1, unofficial].
    e.  **Get Phase Function:** Evaluate $p = p(\boldsymbol{\omega}, \mathbf{l}-\mathbf{x}_{sample})$ [S1, unofficial].
    f.  **Get Light Radiance:** Retrieve $L_e$ from the point light [S1, unofficial].
    g.  **Accumulate Contribution:** Add the contribution divided by the PDF. For uniform sampling over length $L_{ray}$, $PDF = 1/L_{ray}$.
        $$ L_s \leftarrow L_s + \frac{L_e \cdot T_{eye\_sample} \cdot \sigma_s \cdot p \cdot T_{light}}{PDF} $$
        *Note: The step size $dt$ is absorbed into the PDF for uniform sampling [S1, unofficial].*
5.  **Average and Combine:** Average the scattered radiance: $L_{avg} = L_s / N_{samples}$ [S1, unofficial]. Final radiance $L_o = L_i + L_{avg}$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_single_scattering_monte_carlo_point_light(ray, volume, light, background, n_samples):
    """
    ray: Ray object with origin o and direction w
    volume: Homogeneous volume with sigma_s, sigma_t
    light: Point light with position l and radiance L_e
    background: Background radiance L_b
    n_samples: Number of Monte Carlo samples
    """
    # 1. Calculate hit point on background
    t_max = ray.get_intersection_distance() # Or scene intersection
    x_hit = ray.origin + ray.direction * t_max
    
    # 2. Calculate transmittance to background
    dist_to_hit = length(x_hit - ray.origin)
    t_eye = exp(-volume.sigma_t * dist_to_hit)
    
    # 3. Initialize accumulated radiance with background
    L_i = background * t_eye
    
    # 4. Initialize scattered radiance accumulator
    L_s = 0.0
    
    # 5. Monte Carlo Integration
    # Assuming uniform sampling over the ray length [0, t_max]
    # PDF for uniform sampling over length L is 1/L
    ray_length = t_max
    pdf = 1.0 / ray_length
    
    for i in range(n_samples):
        # Sample distance uniformly along the ray
        t_sample = random_float() * ray_length
        
        # Calculate sampled position
        x_sample = ray.origin + ray.direction * t_sample
        
        # Calculate transmittance from eye to sample point
        dist_eye = length(x_sample - ray.origin)
        t_eye_sample = exp(-volume.sigma_t * dist_eye)
        
        # Calculate transmittance from sample point to light
        dist_light = length(light.position - x_sample)
        t_light = exp(-volume.sigma_t * dist_light)
        
        # Get phase function value
        # Direction from sample to light
        w_light = normalize(light.position - x_sample)
        # Phase function p(w_out, w_in)
        # Note: In single scattering, w_out is ray.direction, w_in is from light
        pf = volume.get_phase_function(ray.direction, w_light)
        
        # Get emitted radiance from light
        L_e = light.get_radiance()
        
        # Scattering coefficient
        sigma_s = volume.get_sigma_s()
        
        # Accumulate contribution: Le * T_eye * sigma_s * p * T_light / PDF
        # Note: step size is not explicitly multiplied because it is part of the PDF denominator
        contribution = (L_e * t_eye_sample * sigma_s * pf * t_light) / pdf
        L_s += contribution
        
    # 6. Average scattered radiance
    L_avg = L_s / n_samples
    
    # 7. Final radiance
    L_o = L_i + L_avg
    
    return L_o
```

**Edge Cases and Pitfalls**
*   **Zero Length Ray:** If the ray does not intersect the volume or $t_{max}=0$, return background radiance directly [S1, unofficial].
*   **High Extinction:** If $\sigma_t$ is very large, transmittance becomes negligible, leading to dark results. Ensure numerical stability in exponent calculations [S1, unofficial].
*   **PDF Mismatch:** If sampling is not uniform over the integration domain, the PDF must be adjusted accordingly. Using uniform sampling over $[0, t_{max}]$ is simple but may be inefficient if the volume is small compared to the ray length [S1, unofficial].
*   **Light Inside Volume:** If the light is inside the volume, the transmittance calculation $T_{light}$ still holds, but the geometry may require handling occlusion if other objects are present [S1, unofficial].

## Subtopics

### [Ray Marching with In/Out-Scattering and Density Scaling](2318-ray-marching-with-in-out-scattering-and-density-scaling.md)
This method computes single scattering radiance in participating media by ray marching, explicitly accounting for absorption, out-scattering (attenuation), and in-scattering (energy gain). It distinguishes between homogeneous media (uniform coefficients) and heterogeneous media (spatially varying density) by using a density scalar to scale extinction and scattering terms. Unlike the parent method which uses Monte Carlo integration, this approach uses deterministic uniform sampling along the ray, and unlike [Single Scattering Approximation with Precomputed Terms](4188-single-scattering-approximation-with-precomputed-terms.md) or [Analytical Approximation of Single Scattering Radiance](4244-analytical-approximation-of-single-scattering-radiance.md), it does not rely on precomputed terms or closed-form solutions but iteratively accumulates light contributions per step.

### [Single Scattering Approximation with Precomputed Terms](4188-single-scattering-approximation-with-precomputed-terms.md)
This method is a rendering approximation that considers only single scattering effects and uses pre-computed direction-independent radiance components to improve efficiency. It differs from its siblings by replacing stochastic Monte Carlo sampling with deterministic ray marching and by decoupling the angular integration into pre-computed terms, enabling differentiable optimization of the density field.

### [Analytical Approximation of Single Scattering Radiance](4244-analytical-approximation-of-single-scattering-radiance.md)
This method computes the in-scattered single scattering radiance $J_{ss}$ by approximating the phase function and incident radiance with low-order expansions, allowing the scattering contribution to be decomposed into direction-independent terms that are pre-computed per voxel. It differs from Monte Carlo approaches by avoiding repeated ray tracing for incident light estimation during rendering, making it significantly more efficient for differentiable rendering or multi-viewpoint reconstruction where gradients are required.

### [Volume Single Scattering (Uniform Ray Marching with Point Light Background)](4550-volume-single-scattering-uniform-ray-marching-with-point-lig.md)
This method computes single scattering radiance in participating media using Monte Carlo integration with uniform sampling along the ray distance, specifically adapted for a sphere light background and emission source. It differs from its parent [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md) by replacing the point light model with a sphere light, requiring equi-angular sampling for the light interaction and specific transmittance calculations from the light position.

### [Volume Single Scattering (Uniform Ray Marching with Point Light Background)](4550-volume-single-scattering-uniform-ray-marching-with-point-lig.md)
This method computes single scattering radiance in a participating medium using uniform ray marching, where the background radiance is computed via a point light model instead of a generic surface object color. It replaces the Monte Carlo integration of its parent method with deterministic uniform sampling along the ray.

### [BSSRDF Approximation with Dipole and Single Scattering](4750-bssrdf-approximation-with-dipole-and-single-scattering.md)
The BSSRDF approximation with dipole diffusion and single scattering computes subsurface light transport by evaluating the Bidirectional Surface Scattering Reflectance Distribution Function (BSSRDF) on the surface boundary, combining a dipole diffusion solution for multiple scattering with a first-order scattering term. It replaces expensive volumetric path tracing with an analytical diffusion-based solution using real and virtual point sources, and explicitly accounts for single scattering along refracted ray segments.

### [Physical Atmosphere Color Calculation](4820-physical-atmosphere-color-calculation.md)
Calculates the physical color of the sky and sun disk for a given view direction, accounting for Rayleigh and Mie scattering, optical depth, and phase functions. It differs from sibling methods by providing a closed-form analytical approximation of the atmospheric radiance rather than using Monte Carlo integration or precomputed lookup tables.

### Single Scattering with Direction Variance Culling
This variant traverses a hierarchy of volume samples in a top-down manner, discarding nodes not intersected by the camera ray. At leaf nodes, it tests the direction variance of the phase function. If both the direction variance and the phase function value are below a threshold, it gathers the contribution from the leaf node as a whole; otherwise, it gathers contributions from individual volume samples [S5]. The contribution is scaled by a density factor $k_v$:
$$ \text{single} = \alpha \sum_v p(\hat{\boldsymbol{h}}, \hat{\boldsymbol{d}}_v) \frac{I_v}{k_v} (e^{-\sigma_T d_{\text{near}}} - e^{-\sigma_T d_{\text{far}}}) $$
where $d_{\text{near}}$ and $d_{\text{far}}$ are the intersections with the sample volume of influence [S5]. Parameters include the density factor $k_v$, albedo/scattering multiplier $\alpha$, total extinction $\sigma_T$, phase function $p$, and incoming radiance $I_v$ [S5].

### Sakas's Projective Polygonal Rendering
A method to render arbitrary distributions of volume densities using projective polygonal rendering and solid texturing [S6]. It derives the general radiance equation for single scattering and solves it with front-to-back Monte Carlo sampling. Simplifications include neglecting absorption for low-density volumes and assuming constant density. It uses a Bresenham algorithm to traverse a voxel field modeled as a participating medium, often created using fractal techniques [S6].

## Operating characteristics
*   **Time Complexity:** $O(N_{samples})$ per ray, where $N_{samples}$ is the number of Monte Carlo samples. Each sample requires calculating transmittance to the eye and light, and evaluating the phase function [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory, as it only stores current radiance accumulators and temporary variables [S1, unofficial].
*   **Performance:** The method is unbiased but converges slowly ($1/\sqrt{N}$). Noise is prominent with low sample counts, especially in highly scattering media [S2].
*   **Strengths:** Simple to implement; unbiased; handles homogeneous media efficiently [S1, unofficial].
*   **Limitations:** Noisy with few samples; inefficient for heterogeneous media without adaptive sampling; limited to single scattering [S2].
*   **When to Use:** Educational purposes, simple homogeneous volumes, or as a base for more complex estimators [S2].
*   **When Not to Use:** Heterogeneous media, multiple scattering, or real-time applications requiring low noise [S2].

## Background
The method originates from fundamental principles of radiative transfer in participating media, specifically the single scattering approximation [S1, unofficial]. It is commonly implemented in educational ray tracers and basic volume rendering engines [S1, unofficial]. The use of Monte Carlo integration for volume rendering was popularized in the early days of computer graphics, with specific implementations for point lights being a standard example in tutorials [S4, unofficial]. The approach is detailed in various online resources and code repositories, such as the WangYue ray tracer [S1, unofficial] and Scratchapixel tutorials [S4, unofficial].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S2] Volumes — Blender Manual, https://www.blender.org/manual/nb/3.6/render/materials/components/volume.html, fetched 2026-10-26
[S3] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
[S4] Ray Marching: Getting it Right!, https://www.scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers//ray-marching-get-it-right.html, fetched 2026-10-28 (unofficial)
[S5] Point-Based Rendering for Homogeneous Participating Media with Refractive Boundaries, https://doi.org/10.1109/TVCG.2017.2768525, fetched 2026-10-26
[S6] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
