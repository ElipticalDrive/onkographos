# Volume Single Scattering (Monte Carlo with Point Light)

Also called: GetLightEnergy, Light Energy Calculation, PBRT SingleScatteringIntegrator::Li, Single Scattering Volume Integration (PBRT Implementation)

## Overview
This method computes the radiance contribution of a single scattering event in a participating medium by integrating the incident light from a point source along a view ray using Monte Carlo integration [S1, unofficial]. It solves the problem of rendering volumetric effects (such as fog, smoke, or clouds) where light from a specific source is scattered into the camera, attenuated by the medium's extinction [S3, unofficial]. Unlike deterministic uniform ray marching variants [S1, unofficial], this approach uses stochastic sampling of the ray distance to estimate the integral, which allows for efficient handling of complex lighting scenarios by decoupling the spatial integration from the light evaluation [S3, unofficial]. It builds on [Single Scattering Approximations](5340-single-scattering-approximations.md) by specifically implementing the Monte Carlo estimator for a point light source.

## How it works
**Inputs:**
*   Ray $\mathbf{r}(t) = \mathbf{o} + t\boldsymbol{\omega}$ defined by origin $\mathbf{o}$ and direction $\boldsymbol{\omega}$.
*   Volume object with homogeneous coefficients: total extinction $\sigma_t$, scattering coefficient $\sigma_s$ [S1, unofficial].
*   Phase function value $p(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$, assumed constant for homogeneous volumes [S1, unofficial].
*   Point light source with position $\mathbf{l}_{pos}$ and intensity $I$ [S1, unofficial].
*   Background color $C_{bg}$ [S1, unofficial].
*   Number of Monte Carlo samples $N_{samples}$ (denoted as `MCsamples` in sources) [S1, unofficial].
*   Maximum ray distance $t_{max}$ (distance to background intersection or volume boundary) [S1, unofficial].

**Outputs:**
*   Final radiance $L_o$ at the camera pixel.

**Data Structures:**
*   No complex data structures are required for the basic homogeneous case. The volume properties are accessed via getters (e.g., `volume->getSigmaS()`) [S1, unofficial].

**Steps:**

1.  **Calculate Background Contribution:**
    Determine the intersection point with the background or volume far boundary at $t_{max}$ [S1, unofficial]. Calculate the transmittance from the ray origin to this point:
    $$ T(\mathbf{o}, \mathbf{p}_{far}) = \exp(-\sigma_t \cdot t_{max}) $$
    The background radiance contribution is:
    $$ L_{bg\_contrib} = C_{bg} \cdot T(\mathbf{o}, \mathbf{p}_{far}) $$
    Initialize the accumulated radiance $L_{accum} = L_{bg\_contrib}$ [S1, unofficial].

2.  **Initialize Monte Carlo Integration:**
    Initialize the scattered radiance accumulator $L_{scatter} = 0$ [S1, unofficial].

3.  **Sample Distances:**
    For each sample $i$ from $0$ to $N_{samples} - 1$:
    a.  Sample a distance $t_{sample}$ along the ray from $\mathbf{o}$ to $t_{max}$ using uniform distance sampling [S1, unofficial]. The probability density function (PDF) for uniform sampling over $[0, t_{max}]$ is:
        $$ \text{pdf} = \frac{1}{t_{max}} $$
        [G: The source mentions `sample1DDBG` with mode `Distance`, which implies uniform sampling over the interval. If the sampling is not uniform, the PDF must be adjusted accordingly.]
    b.  Calculate the sampled position:
        $$ \mathbf{x}_{sample} = \mathbf{o} + t_{sample} \boldsymbol{\omega} $$

4.  **Calculate Transmittances:**
    a.  Calculate transmittance from the eye to the sample point:
        $$ T_{eye} = \exp(-\sigma_t \cdot t_{sample}) $$
        [S1, unofficial]
    b.  Calculate the distance from the sample point to the light source:
        $$ d_{light} = \| \mathbf{x}_{sample} - \mathbf{l}_{pos} \| $$
    c.  Calculate transmittance from the sample point to the light source. Assuming the medium is homogeneous and the light is outside the volume or the path is within the same medium:
        $$ T_{light} = \exp(-\sigma_t \cdot d_{light}) $$
        [S1, unofficial]

5.  **Calculate Incident Radiance:**
    Calculate the emitted radiance from the point light at the sample point. The incident radiance $L_{in}$ from a point light is:
    $$ L_{in} = \frac{I}{d_{light}^2} \cdot T_{light} $$
    [S1, unofficial]
    Note: The source code snippet `getPixelColorPointLight` implies this calculation includes the inverse square law and transmittance [S1, unofficial].

6.  **Accumulate Scattering Contribution:**
    Add the contribution of the single scattering event to $L_{scatter}$, divided by the PDF to correct for the sampling distribution:
    $$ L_{scatter} += \frac{L_{in} \cdot T_{eye} \cdot \sigma_s \cdot p(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)}{\text{pdf}} $$
    [S1, unofficial]
    Here, $\sigma_s$ is the scattering coefficient and $p$ is the phase function value [S1, unofficial].

7.  **Average and Combine:**
    Average the scattered radiance over the number of samples:
    $$ L_{scatter\_avg} = \frac{L_{scatter}}{N_{samples}} $$
    [S1, unofficial]
    The final radiance is:
    $$ L_o = L_{bg\_contrib} + L_{scatter\_avg} $$
    [S1, unofficial]

**Parameters:**
*   $N_{samples}$ (MCsamples): Number of Monte Carlo samples. Typical values depend on desired noise level; higher values reduce noise [S1, unofficial].
*   $t_{max}$: Maximum distance. Determined by scene geometry [S1, unofficial].
*   $\sigma_t, \sigma_s$: Volume coefficients. $\sigma_t \ge \sigma_s \ge 0$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Distance:** If $t_{sample} = 0$ or $d_{light} = 0$, transmittance is 1. Division by zero in inverse square law must be handled (e.g., clamp distance to a small epsilon) [G].
*   **PDF Mismatch:** If the sampling distribution is not uniform, the PDF in step 6 must be updated to match the sampling strategy [G].
*   **Homogeneous Assumption:** This method assumes constant $\sigma_t$ and $\sigma_s$ along the ray. For heterogeneous volumes, the transmittance calculation requires integration of the extinction coefficient along the path [S1, unofficial].

## Subtopics

### [Ray Marching with In/Out-Scattering and Density Scaling](2318-ray-marching-with-in-out-scattering-and-density-scaling.md)
This method computes single scattering radiance in participating media by ray marching, explicitly accounting for absorption, out-scattering (attenuation), and in-scattering (energy gain). It distinguishes between homogeneous media (uniform coefficients) and heterogeneous media (spatially varying density) by using a density scalar to scale extinction and scattering terms. Unlike the parent method [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md) which uses Monte Carlo integration, this approach uses deterministic uniform sampling along the ray, and unlike [Single Scattering Approximation with Precomputed Terms](4188-single-scattering-approximation-with-precomputed-terms.md) or [Analytical Approximation of Single Scattering Radiance](4244-analytical-approximation-of-single-scattering-radiance.md), it does not rely on precomputed terms or closed-form solutions but iteratively accumulates light contributions per step.

### [Single Scattering Approximation with Precomputed Terms](4188-single-scattering-approximation-with-precomputed-terms.md)
This method is a rendering approximation that considers only single scattering effects and uses pre-computed direction-independent radiance components to improve efficiency. It differs from its siblings by replacing stochastic Monte Carlo sampling with deterministic ray marching and by decoupling the angular integration into pre-computed terms, enabling differentiable optimization of the density field.

### [Analytical Approximation of Single Scattering Radiance](4244-analytical-approximation-of-single-scattering-radiance.md)
This method computes the in-scattered single scattering radiance $J_{ss}$ by approximating the phase function and incident radiance with low-order expansions, allowing the scattering contribution to be decomposed into direction-independent terms that are pre-computed per voxel. It differs from Monte Carlo approaches by avoiding repeated ray tracing for incident light estimation during rendering, making it significantly more efficient for differentiable rendering or multi-viewpoint reconstruction where gradients are required.

### [Volume Single Scattering (Monte Carlo with Sphere Light)](4549-volume-single-scattering-monte-carlo-with-sphere-light.md)
This method computes the single-scattering radiance for a participating medium using Monte Carlo integration, specifically designed for scenes with a sphere light (or environment map) acting as the background and emission source. It differs from the point-light variant by using `getPixelColorWithSphereLight` to sample the complex lighting environment via importance sampling of solid angles, rather than a direct geometric evaluation of a point source. It builds on [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md) by replacing the light evaluation step with a sphere-light-aware sampler.

### [Volume Single Scattering (Uniform Ray Marching with Point Light Background)](4550-volume-single-scattering-uniform-ray-marching-with-point-lig.md)
This method computes single scattering radiance in a participating medium using uniform ray marching, where the background radiance is determined by a point light model rather than a generic surface object color. It differs from its parent [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md) by using deterministic uniform sampling along the ray instead of Monte Carlo integration, and from siblings like [Ray Marching with In/Out-Scattering and Density Scaling](2318-ray-marching-with-in-out-scattering-and-density-scaling.md) by explicitly decoupling the background illumination calculation into a specific point light function.

### [BSSRDF Approximation with Dipole and Single Scattering](4750-bssrdf-approximation-with-dipole-and-single-scattering.md)
This method renders translucent materials by combining a dipole diffusion approximation for multiple scattering with a first-order single scattering term, addressing the problem of light entering and exiting a medium at different positions. It sets itself apart from pure surface BRDF models by accounting for subsurface diffusion and from full volumetric path tracing by using an analytical approximation to compute diffuse reflectance, offering a balance between physical accuracy and computational efficiency. It builds on [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md) by replacing the pure Monte Carlo single-scattering integration with an analytical single-scattering term and adding a dipole-based multiple scattering component.

### [Physical Atmosphere Color Calculation](4820-physical-atmosphere-color-calculation.md)
Calculates the physical color of the sky and sun disk for a given view direction, accounting for Rayleigh and Mie scattering, optical depth, and phase functions. It differs from sibling methods by providing a closed-form analytical approximation of the atmospheric radiance rather than using Monte Carlo integration or precomputed lookup tables.

### Single Scattering with Direction Variance Culling
This variant optimizes the single scattering calculation by traversing a hierarchy of volume samples. It discards nodes not intersected by the camera ray. For leaf nodes, it tests the direction variance of the phase function. If both the direction variance and the phase function value are below a threshold, it gathers the contribution from the leaf node as a whole; otherwise, it gathers contributions from individual volume samples [S4]. The contribution is scaled by a density factor $k_v$:
$$ \text{single} = \alpha \sum_v p(\hat{\boldsymbol{h}}, \hat{\boldsymbol{d}}_v) \frac{I_v}{k_v} (e^{-\sigma_T d_{\text{near}}} - e^{-\sigma_T d_{\text{far}}}) $$
where $d_{\text{near}}$ and $d_{\text{far}}$ are the intersections with the sample volume of influence, $\alpha$ is an albedo/scattering multiplier, $\sigma_T$ is the total extinction coefficient, and $I_v$ is the incoming radiance of volume sample $v$ [S4].

## Operating characteristics
*   **Time Complexity:** $O(N_{samples})$ per ray, where $N_{samples}$ is the number of Monte Carlo samples. The complexity is linear with respect to the number of samples.
*   **Memory Complexity:** $O(1)$ auxiliary memory for the homogeneous case, as no large data structures are needed beyond the volume coefficients [S1, unofficial].
*   **Measured Performance:** Performance depends heavily on the number of samples required to achieve low noise. For homogeneous volumes, the calculation per sample is very fast (exponential and basic arithmetic) [S1, unofficial].
*   **Scaling:** Scales linearly with the number of samples. Increasing $N_{samples}$ reduces noise by $\sqrt{N_{samples}}$ [G].
*   **Strengths:** Simple to implement for homogeneous volumes. Accurate for single scattering events. Flexible number of samples allows trade-off between speed and quality [S1, unofficial].
*   **Limitations:** Noisy with few samples. Inefficient for heterogeneous volumes as transmittance calculation becomes more complex. Does not account for multiple scattering [S1, unofficial].
*   **When to Use:** Real-time rendering of homogeneous volumes with point lights where single scattering is sufficient.
*   **When Not to Use:** Heterogeneous volumes, scenes requiring multiple scattering, or when deterministic results are needed without noise [S1, unofficial].

## Background
The method originates from basic Monte Carlo integration techniques applied to the radiative transfer equation in participating media. The specific implementation details, such as the use of `getPixelColorPointLight` and homogeneous volume assumptions, are found in educational ray tracing projects [S1, unofficial]. The PBRT implementation of `SingleScatteringIntegrator::Li` provides a more general framework for single scattering integration, which this method simplifies for point lights and homogeneous media [S3, unofficial]. The technique is widely used in real-time graphics for rendering fog, smoke, and clouds where computational efficiency is critical [S2, unofficial].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S2] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
[S3] Ray Marching: Getting it Right!, https://www.scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers//ray-marching-get-it-right.html, fetched 2026-10-28 (unofficial)
[S4] Point-Based Rendering for Homogeneous Participating Media with Refractive Boundaries, https://doi.org/10.1109/TVCG.2017.2768525, fetched 2026-10-26
