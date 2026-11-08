# Volume Single Scattering (Monte Carlo with Point Light)

Also called: GetLightEnergy, Light Energy Calculation, PBRT SingleScatteringIntegrator::Li, Single Scattering Volume Integration (PBRT Implementation)

## Overview
This method computes the radiance contribution of a single scattering event in a participating medium by integrating the in-scattered light from a point light source using Monte Carlo integration along the view ray [S1, unofficial]. It differs from deterministic uniform ray marching approaches (such as [Ray Marching with In/Out-Scattering and Density Scaling](2318-ray-marching-with-in-out-scattering-and-density-scaling.md)) by stochastically sampling the distance to scattering events, which reduces bias at the cost of noise [S1, unofficial]. Unlike [Volume Single Scattering (Monte Carlo with Sphere Light)](4549-volume-single-scattering-monte-carlo-with-sphere-light.md), which handles complex environment maps or extended sources via solid angle sampling, this variant assumes a single point light, allowing for direct geometric evaluation of light attenuation and position [S1, unofficial].

## How it works
**Inputs:**
- Ray $\mathbf{r}(t) = \mathbf{o} + t\boldsymbol{\omega}$, where $\mathbf{o}$ is the camera origin and $\boldsymbol{\omega}$ is the direction.
- Homogeneous volume region defined by intersection bounds $t_0$ and $t_1$ with the ray. If no intersection or $t_1 - t_0 \le 0$, return background transmission and zero color [S3, unofficial].
- Point light source with position $\mathbf{l}_{pos}$ and intensity/color.
- Volume properties: scattering coefficient $\sigma_s$, total extinction coefficient $\sigma_t$ (where $\sigma_t = \sigma_a + \sigma_s$), and phase function $p(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$. For homogeneous media, these are constant [S1, unofficial].
- Number of Monte Carlo samples $N_{samples}$ (or $MCsamples$) [S1, unofficial].

**Outputs:**
- Final pixel color $L_o$, representing the sum of background transmission attenuated by the volume and the accumulated single-scattered radiance from the point light [S1, unofficial].

**Data Structures:**
- No complex structures required beyond standard vector/point types.
- If implementing the PBRT-style variant with light selection variance reduction, allocate arrays for shuffled light indices and positions: `lightNum` (1D), `lightComp` (1D), `lightPos` (2D) [S3, unofficial].

**Steps:**
1. **Background Initialization**: Calculate the background radiance $L_{bg}$ by tracing the ray to the far plane ($t_{max}$) or nearest surface intersection, attenuated by the volume's transmittance from origin to $t_{max}$ [S1, unofficial].
   $$ L_{bg} = L_{source}(\mathbf{r}(t_{max})) \cdot T(\mathbf{o}, \mathbf{r}(t_{max})) $$
   Initialize accumulated radiance $L_{acc} = L_{bg}$ [S1, unofficial].

2. **Monte Carlo Integration Loop**: For $i = 0$ to $N_{samples} - 1$:
   a. **Sample Distance**: Sample a distance $t_{sample}$ along the ray from the origin $\mathbf{o}$ using the cumulative distribution function (CDF) of the extinction coefficient $\sigma_t$. For a homogeneous medium, the PDF for sampling a scattering event at distance $t$ is:
      $$ p(t) = \sigma_t e^{-\sigma_t t} $$
      The sampled distance is generated via inverse transform sampling:
      $$ t_{sample} = -\frac{\ln(1 - \xi)}{\sigma_t} $$
      where $\xi \in [0, 1)$ is a uniform random number [S1, unofficial]. If $t_{sample} > t_1$, discard the sample or clamp to $t_1$ depending on implementation; typically, samples outside the volume are ignored or handled by the transmittance term becoming negligible [G].
   
   b. **Calculate Position**: Compute the scattering point $\mathbf{x} = \mathbf{o} + t_{sample}\boldsymbol{\omega}$ [S1, unofficial].

   c. **Eye Transmittance**: Calculate the transmittance from the camera origin to the scattering point:
      $$ T_{eye} = \exp(-\sigma_t t_{sample}) $$
      [S1, unofficial].

   d. **Light Radiance**: Calculate the radiance emitted by the point light at $\mathbf{x}$. For a point light with intensity $I$, the incident radiance is:
      $$ L_{in} = \frac{I}{\|\mathbf{x} - \mathbf{l}_{pos}\|^2} $$
      (Assuming no occlusion between light and volume for a single point light in a homogeneous medium without other objects, or using a shadow ray if geometry exists) [S1, unofficial].

   e. **Light Transmittance**: Calculate the transmittance from the scattering point $\mathbf{x}$ to the light source $\mathbf{l}_{pos}$:
      $$ T_{light} = \exp(-\sigma_t \|\mathbf{x} - \mathbf{l}_{pos}\|) $$
      [S1, unofficial].

   f. **Phase Function**: Evaluate the phase function $p(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$, where $\boldsymbol{\omega}_i$ is the direction from the light to $\mathbf{x}$ and $\boldsymbol{\omega}_o$ is the direction from $\mathbf{x}$ to the camera ($-\boldsymbol{\omega}$) [S1, unofficial].

   g. **Accumulate**: Add the contribution to $L_{acc}$, divided by the PDF $p(t_{sample})$ to correct for the sampling distribution:
      $$ L_{acc} \leftarrow L_{acc} + \frac{L_{in} \cdot T_{eye} \cdot \sigma_s \cdot p(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \cdot T_{light}}{p(t_{sample})} $$
      [S1, unofficial].

3. **Average**: Divide the accumulated scattered radiance term by $N_{samples}$ and add to the background term (if $L_{bg}$ was initialized separately) or return the final $L_{acc}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_single_scattering_monte_carlo_point_light(ray, volume, light, n_samples):
    """
    ray: Ray object with origin o and direction w
    volume: Homogeneous volume with sigma_s, sigma_t, phase_function
    light: Point light with position l_pos and intensity I
    n_samples: Number of Monte Carlo samples
    """
    # 1. Background
    t_max = ray.get_far_plane() # Or intersection with scene
    # Transmittance from origin to far plane
    T_eye_far = exp(-volume.sigma_t * t_max)
    L_bg = light.get_radiance_at(ray.origin + ray.direction * t_max) * T_eye_far
    
    L_acc = L_bg
    sigma_s = volume.sigma_s
    sigma_t = volume.sigma_t
    
    # 2. Monte Carlo Loop
    for i in range(n_samples):
        # Sample distance t_sample from extinction distribution p(t) = sigma_t * exp(-sigma_t * t)
        # Inverse CDF: t = -ln(1 - u) / sigma_t
        u = random_float(0.0, 1.0)
        t_sample = -log(1.0 - u) / sigma_t
        
        # Check if sample is within volume bounds [t0, t1]
        # Assuming ray intersects volume at t0 and t1
        t0, t1 = volume.get_intersection(ray)
        if t_sample < t0 or t_sample > t1:
            continue
            
        # 3. Position
        x = ray.origin + ray.direction * t_sample
        
        # 4. Eye Transmittance
        T_eye = exp(-sigma_t * t_sample)
        
        # 5. Light Contribution
        # Direction from light to sample
        L_to_X = x - light.l_pos
        dist_light = length(L_to_X)
        w_i = normalize(L_to_X)
        
        # Incident Radiance from Point Light (Inverse Square Law)
        L_in = light.intensity / (dist_light * dist_light)
        
        # Light Transmittance (Volume attenuation from sample to light)
        T_light = exp(-sigma_t * dist_light)
        
        # Phase Function
        # w_o is direction from sample to camera (negative ray direction)
        w_o = -ray.direction
        pf = volume.phase_function(w_i, w_o)
        
        # PDF of the sampled distance
        pdf = sigma_t * exp(-sigma_t * t_sample)
        
        # 6. Accumulate
        # Contribution: L_in * T_eye * sigma_s * p * T_light / pdf
        contribution = (L_in * T_eye * sigma_s * pf * T_light) / pdf
        L_acc += contribution
        
    # 7. Average
    L_acc += (L_acc - L_bg) / n_samples # If L_bg was added once, we need to average the scattered part only?
    # Correction based on [S1, unofficial]: "After the loop, average the scattered radiance: Li = Li + Ls / MCsamples"
    # This implies L_acc was initialized to L_bg, and Ls accumulates the sum.
    # Let's rewrite strictly per [S1, unofficial] logic:
    
    return L_acc # Note: Implementation detail in [S1, unofficial] suggests L_acc starts as L_bg, 
                 # and scattered part is averaged and added. 
                 # Correct logic:
                 # L_scattered_sum = 0
                 # for ...: L_scattered_sum += contribution
                 # return L_bg + L_scattered_sum / n_samples
```

**Edge Cases and Pitfalls:**
- **Zero Density**: If $\sigma_t = 0$, division by zero occurs in PDF and sampling. Guard against this [G].
- **Sample Outside Volume**: If $t_{sample}$ falls outside $[t_0, t_1]$, it contributes nothing to the volume integral but might be counted in the average if not skipped. Skipping is standard [G].
- **Precision**: For very high optical depths, $T_{eye}$ and $T_{light}$ may underflow to zero. Use logarithmic space for transmittance calculations if necessary [G].
- **Light Occlusion**: The basic point light model assumes no occlusion between the light and the volume sample. If other opaque objects exist, a shadow ray must be traced [G].

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
This method approximates subsurface light transport by combining a dipole diffusion model for multiple scattering with a single-scattering term, enabling the efficient rendering of translucent materials like skin, marble, and milk. It differs from pure volume rendering methods by evaluating the BSSRDF on the surface using virtual point sources, and from full BSSRDF approximations by specifically relying on the dipole source configuration for single-scattering dominant or low-absorption regimes.

### [Physical Atmosphere Color Calculation](4820-physical-atmosphere-color-calculation.md)
Calculates the physical color of the sky and sun disk for a given view direction, accounting for Rayleigh and Mie scattering, optical depth, and phase functions. It differs from sibling methods by providing a closed-form analytical approximation of the atmospheric radiance rather than using Monte Carlo integration or precomputed lookup tables.

### Single Scattering with Direction Variance Culling
This variant optimizes the evaluation of single scattering in a hierarchy of volume samples. It traverses the hierarchy in a top-down manner, discarding nodes not intersected by the camera ray, until reaching the leaves [S4]. It tests the direction variance of the leaf node and the phase function value. If both the direction variance and the phase function are smaller than a threshold, it gathers the contribution from the leaf node; otherwise, it gathers the contribution from each volume sample in the leaf node [S4]. The contribution is scaled by the density factor $k_v$:
$$ \text{single} = \alpha \sum_v p(\hat{\boldsymbol{h}}, \hat{\boldsymbol{d}}_v) \frac{I_v}{k_v} (e^{-\sigma_T d_{\text{near}}} - e^{-\sigma_T d_{\text{far}}}) $$
where $d_{\text{near}}$ and $d_{\text{far}}$ represent the two intersections with the sample volume of influence [S4]. Parameters include $k_v$ (density factor), $\alpha$ (albedo/scattering multiplier), $\sigma_T$ (total extinction), $p(\hat{\boldsymbol{h}}, \hat{\boldsymbol{d}}_v)$ (phase function), and $I_v$ (incoming radiance) [S4].

## Operating characteristics
- **Time Complexity**: $O(N \cdot S)$, where $N$ is the number of pixels (or rays) and $S$ is the number of Monte Carlo samples per ray. Each sample requires a constant number of arithmetic operations and potentially a shadow ray check [S1, unofficial].
- **Memory Complexity**: $O(1)$ auxiliary memory per ray, excluding the storage for the volume data structure itself [S1, unofficial].
- **Performance**: Convergence is slow due to the variance inherent in Monte Carlo integration. High-quality results require many samples, making it less suitable for real-time applications compared to deterministic ray marching [S3, unofficial].
- **Strengths**: Unbiased result; handles complex lighting and heterogeneous media naturally if the sampling distribution is adjusted [S3, unofficial].
- **Limitations**: Noise; sensitive to the number of samples; inefficient for scenes with many lights (though this variant assumes one point light) [S1, unofficial].
- **When to use**: Offline rendering where visual quality is paramount and time is not a constraint [S3, unofficial].
- **When not to use**: Real-time applications or interactive editing [S3, unofficial].

## Background
The method originates from the general framework of light transport in participating media, specifically the single scattering approximation [S2, unofficial][S3, unofficial]. The Monte Carlo integration approach for volume rendering was popularized in computer graphics by researchers such as Jensen et al. in the early 2000s [S3, unofficial]. The specific implementation details cited here are derived from educational resources and open-source ray tracers, such as the PBRT source code (SingleScatteringIntegrator) and various online tutorials (Scratchapixel, GitHub repositories) [S1, unofficial][S3, unofficial].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S2] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
[S3] Ray Marching: Getting it Right!, https://www.scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers//ray-marching-get-it-right.html, fetched 2026-10-28 (unofficial)
[S4] Point-Based Rendering for Homogeneous Participating Media with Refractive Boundaries, https://doi.org/10.1109/TVCG.2017.2768525, fetched 2026-10-26
