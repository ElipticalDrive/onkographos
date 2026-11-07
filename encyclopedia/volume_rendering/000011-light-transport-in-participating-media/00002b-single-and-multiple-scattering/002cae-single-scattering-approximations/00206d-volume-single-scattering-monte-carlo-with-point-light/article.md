# Volume Single Scattering (Monte Carlo with Point Light)
Also called: GetLightEnergy, Light Energy Calculation, PBRT SingleScatteringIntegrator::Li, Single Scattering Volume Integration (PBRT Implementation)

## Overview
This method computes the radiance contribution of a single scattering event in a participating medium using Monte Carlo integration with uniform sampling along the ray distance, specifically for scenes containing a point light source [S1, unofficial]. It solves the problem of estimating in-scattered light by discretizing the ray into fixed steps or random samples, summing the emitted light attenuated by transmittance from the eye and to the light source [S1, unofficial]. Unlike deterministic ray marching approaches [2318-ray-marching-with-in-out-scattering-and-density-scaling.md] or precomputed term approximations [4188-single-scattering-approximation-with-precomputed-terms.md], this method relies on stochastic sampling to estimate the integral, allowing for flexible handling of complex lighting interactions at the cost of noise [S3, unofficial].

## How it works
The method calculates the total radiance $L_o$ by combining the attenuated background radiance with the accumulated in-scattered radiance from the point light. The implementation varies slightly between the PBRT-based approach (deterministic stepping with random light selection) and the simplified homogeneous volume approach (uniform Monte Carlo distance sampling). Both rely on the single-scattering approximation defined in [Single Scattering Approximations](5340-single-scattering-approximations.md).

### Inputs and Outputs
*   **Inputs**:
    *   Ray origin $\mathbf{o}$ and direction $\boldsymbol{\omega}$.
    *   Volume properties: Homogeneous scattering coefficient $\sigma_s$, total extinction coefficient $\sigma_t$ (where $\sigma_t = \sigma_a + \sigma_s$), and phase function value $p(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$ [S1, unofficial].
    *   Light source: Position $\mathbf{l}$ and intensity/color [S1, unofficial].
    *   Geometry: Intersection boundaries $t_0, t_1$ with the volume region, or maximum ray distance $t_{max}$ [S1, unofficial][S3, unofficial].
    *   Parameters: Number of samples $N$ (or step size), Monte Carlo sample count $MCsamples$ [S1, unofficial].
*   **Outputs**:
    *   Final pixel color/radiance $L_o$ [S1, unofficial].

### Data Structures
*   **Volume Region**: Stores $\sigma_s$, $\sigma_t$, and phase function $p$. For homogeneous volumes, these are constant scalars or vectors [S1, unofficial].
*   **Sample Patterns**: Arrays for light selection indices and positions, shuffled to ensure low-discrepancy sampling [S3, unofficial].
*   **Accumulators**:
    *   $L_v$: Accumulated in-scattered radiance (Spectrum/Color) [S3, unofficial].
    *   $T_r$: Global transmittance (Spectrum/Color) [S3, unofficial].
    *   $L_i$: Total accumulated radiance (including background) [S1, unofficial].

### Steps

#### Variant A: PBRT-style Ray Marching with Random Light Selection [S3, unofficial]
This variant uses deterministic spatial steps but randomizes light selection to estimate the integral.

1.  **Intersection**: Find intersection boundaries $t_0$ and $t_1$ with the volume region. If no intersection or zero length, return transmission 1 and color 0 [S3, unofficial].
2.  **Initialization**:
    *   Initialize final color $L_v = 0$ and global transmission $T_r = 1$ [S3, unofficial].
    *   Calculate number of samples $n_{samples} = \lceil (t_1 - t_0) / \text{stepSize} \rceil$ [S3, unofficial].
    *   Calculate actual step size $\Delta t = (t_1 - t_0) / n_{samples}$ [S3, unofficial].
    *   Set initial position $\mathbf{p} = \mathbf{o} + t_0 \boldsymbol{\omega}$ [S3, unofficial].
    *   Jitter the starting position: $t_0 \leftarrow t_0 + \text{sample}[0] \times \Delta t$ to avoid coherence [S3, unofficial].
    *   Allocate and shuffle sample patterns for light selection: `lightNum`, `lightComp`, and `lightPos` using scrambled 1D/2D patterns [S3, unofficial].
3.  **Ray Marching Loop** (for $i = 0$ to $n_{samples} - 1$):
    *   Advance $t_0 \leftarrow t_0 + \Delta t$ [S3, unofficial].
    *   Update previous position $\mathbf{p}_{prev} = \mathbf{p}$ and current position $\mathbf{p} = \mathbf{o} + t_0 \boldsymbol{\omega}$ [S3, unofficial].
    *   **Optical Depth**: Compute optical depth $\tau$ for segment $\mathbf{p}_{prev} \to \mathbf{p}$ using $\sigma_t$. Update global transmission: $T_r \leftarrow T_r \times \exp(-\tau \Delta t)$ [S3, unofficial].
    *   **Russian Roulette**: If $T_r < 10^{-3}$, terminate with probability 0.5 (set $T_r=0$) or continue with $T_r \leftarrow T_r / 0.5$ [S3, unofficial].
    *   **Self-Emission**: Add emitted light contribution: $L_v \leftarrow L_v + T_r \times L_{ve}(\mathbf{p})$ [S3, unofficial].
    *   **In-Scattering**:
        *   Get scattering coefficient $\sigma_s = \sigma_s(\mathbf{p})$ [S3, unofficial].
        *   If $\sigma_s > 0$ and lights exist:
            *   Select light index $l_n = \min(\lfloor \text{lightNum}[i] \times N_{lights} \rfloor, N_{lights}-1)$ [S3, unofficial].
            *   Get light radiance $L_d$ attenuated by transmittance from light to $\mathbf{p}$ [S3, unofficial].
            *   Add contribution: $L_v \leftarrow L_v + T_r \times \sigma_s \times p(\boldsymbol{\omega}, -\boldsymbol{\omega}_l) \times L_d \times \frac{N_{lights}}{\text{pdf}}$ [S3, unofficial]. Note: Step size is NOT multiplied here [S3, unofficial].
4.  **Finalize**: Return $L_v \times \Delta t$ (or accumulated $L_v$ scaled appropriately) [S3, unofficial].

#### Variant B: Homogeneous Volume Monte Carlo Distance Sampling [S1, unofficial]
This variant uses uniform random sampling of distances along the ray for a homogeneous volume with a single point light.

1.  **Background**:
    *   Calculate hit point on background: $\mathbf{p}_{bg} = \mathbf{o} + t_{max} \boldsymbol{\omega}$ [S1, unofficial].
    *   Calculate transmittance from eye to background: $T_{bg} = \exp(-\sigma_t \|\mathbf{p}_{bg} - \mathbf{o}\|)$ [S1, unofficial].
    *   Get background radiance $L_{bg}$ from point light at $\mathbf{p}_{bg}$ [S1, unofficial].
    *   Initialize $L_i = L_{bg} \times T_{bg}$ [S1, unofficial].
2.  **Monte Carlo Integration**:
    *   Initialize accumulated scattered radiance $L_s = 0$ [S1, unofficial].
    *   For $k = 0$ to $MCsamples - 1$:
        *   Sample distance $t_{sample}$ uniformly from $[0, t_{max}]$ [S1, unofficial].
        *   Calculate sampled position: $\mathbf{x}_{sample} = \mathbf{o} + t_{sample} \boldsymbol{\omega}$ [S1, unofficial].
        *   Calculate transmittance from eye to sample: $T_{eye} = \exp(-\sigma_t t_{sample})$ [S1, unofficial].
        *   Calculate transmittance from sample to light: $T_{light} = \exp(-\sigma_t \|\mathbf{l} - \mathbf{x}_{sample}\|)$ [S1, unofficial].
        *   Get light radiance at sample: $L_e = \text{PointLightRadiance}(\mathbf{x}_{sample})$ [S1, unofficial].
        *   Get phase function value $p$ [S1, unofficial].
        *   Add contribution: $L_s \leftarrow L_s + \frac{L_e \times T_{eye} \times \sigma_s \times p \times T_{light}}{\text{pdf}}$ [S1, unofficial].
        *   PDF for uniform distance sampling is $1/t_{max}$ [G].
3.  **Finalize**:
    *   Average scattered radiance: $L_s \leftarrow L_s / MCsamples$ [S1, unofficial].
    *   Total radiance: $L_o = L_i + L_s$ [S1, unofficial].

### Formulas
*   **Transmittance (Beer-Lambert)**:
    $$ T(\mathbf{o}, \mathbf{x}) = \exp\left(-\int_{\mathbf{o}}^{\mathbf{x}} \sigma_t(\mathbf{u}) d\mathbf{u}\right) $$
    For homogeneous media with constant $\sigma_t$ and distance $d$:
    $$ T = \exp(-\sigma_t d) $$ [S1, unofficial][S2, unofficial]
*   **Single Scattering Radiance (General)**:
    $$ L_{ss} = \int_{t_0}^{t_1} T(\mathbf{o}, \mathbf{x}(t)) \sigma_s(\mathbf{x}(t)) p(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) L_{in}(\mathbf{x}(t), \boldsymbol{\omega}_i) dt $$ [S3, unofficial]
*   **Monte Carlo Estimator**:
    $$ L_{ss} \approx \frac{1}{N} \sum_{i=1}^{N} \frac{f(\mathbf{x}_i)}{\text{pdf}(\mathbf{x}_i)} $$ [S1, unofficial]
*   **Direction Variance Culling Formula** (for optimized variants):
    $$ \text{single} = \alpha \sum_v p(\hat{\boldsymbol{h}}, \hat{\boldsymbol{d}}_v) \frac{I_v}{k_v} (e^{-\sigma_T d_{\text{near}}} - e^{-\sigma_T d_{\text{far}}}) $$ [S4]

### Parameters
*   $t_{max}$: Distance from ray origin to background intersection or volume boundary [S1, unofficial].
*   $\sigma_s$: Scattering coefficient [S1, unofficial].
*   $\sigma_t$: Total extinction coefficient ($\sigma_a + \sigma_s$) [S1, unofficial].
*   $p(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$: Phase function value [S1, unofficial].
*   $MCsamples$: Number of Monte Carlo samples [S1, unofficial].
*   $k_v$: Density factor scaling contribution [S4].
*   $\alpha$: Albedo or scattering coefficient multiplier [S4].

### Edge Cases and Pitfalls
*   **Zero Length Intersection**: If $t_1 - t_0 \approx 0$, return transmission 1 and color 0 to avoid division by zero or infinite steps [S3, unofficial].
*   **Russian Roulette Threshold**: If global transmission $T_r$ drops below $10^{-3}$, Russian Roulette is applied to prevent excessive computation in deep shadows [S3, unofficial].
*   **PDF Calculation**: In Variant B, the PDF for uniform distance sampling must be correctly accounted for ($1/t_{max}$) to avoid bias [G].
*   **Step Size Multiplication**: In PBRT-style implementation, step size is multiplied only once at the end, not inside the loop, for optimization [S3, unofficial].

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
This variant optimizes the single scattering calculation by traversing a hierarchy of volume samples. It discards nodes not intersected by the camera ray. For leaf nodes, it tests the direction variance of the phase function. If both the direction variance and the phase function value are below a threshold, it gathers the contribution from the leaf node using a scaled formula; otherwise, it gathers contributions from individual volume samples [S4]. The contribution is scaled by a density factor $k_v$ [S4].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot L)$ where $N$ is the number of samples (or steps) along the ray and $L$ is the number of lights (or 1 if light is selected randomly) [S1, unofficial][S3, unofficial].
*   **Memory Complexity**: $O(1)$ for the homogeneous Monte Carlo variant, as it only requires storing current position and accumulators [S1, unofficial]. The PBRT variant requires $O(N)$ memory for shuffled sample patterns [S3, unofficial].
*   **Performance**: Noise levels depend on the number of Monte Carlo samples. Convergence is slow for highly anisotropic phase functions or deep volumes [S3, unofficial].
*   **Strengths**: Physically accurate for single scattering; flexible with lighting; handles homogeneous volumes efficiently [S1, unofficial].
*   **Limitations**: Noisy with few samples; does not account for multiple scattering; performance degrades with many lights if not sampled efficiently [S3, unofficial].
*   **When to Use**: Real-time rendering with single light sources; offline rendering where noise can be averaged; educational purposes [S1, unofficial][S3, unofficial].
*   **When Not to Use**: Scenes with multiple complex light sources without importance sampling; scenes requiring multiple scattering effects [S3, unofficial].

## Background
The method originates from standard Monte Carlo integration techniques applied to the Radiative Transfer Equation (RTE) for participating media. The PBRT implementation is detailed in the *PBRT* book and source code, specifically the `SingleScatteringIntegrator` [S3, unofficial]. The simplified homogeneous volume approach is found in various educational ray tracing tutorials and open-source projects [S1, unofficial]. The technique is widely used in real-time graphics for volumetric effects like fog, smoke, and clouds, often combined with other approximations for performance [S2, unofficial][S3, unofficial].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S2] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
[S3] Ray Marching: Getting it Right!, https://www.scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers//ray-marching-get-it-right.html, fetched 2026-10-28 (unofficial)
[S4] Point-Based Rendering for Homogeneous Participating Media with Refractive Boundaries, https://doi.org/10.1109/TVCG.2017.2768525, fetched 2026-10-26
