# Spherical Gaussian Environment Map Representation

Also called: SG Environment Map, SG approximation of rendering equation, SG environment lighting, SG mixture, SG sky, Spherical-Gaussian Sky Representation

## Overview
This method represents the distant environment illumination (sky and horizon) as a mixture of Spherical Gaussian (SG) directional lobes, replacing traditional high-resolution HDR environment maps or cubemaps. It solves the problem of efficient, authorable, and analytically differentiable image-based lighting (IBL) by allowing diffuse irradiance and specular prefiltered radiance to be computed in closed form. Unlike standard pipelines that use separate assets or baked textures for appearance and lighting, this approach uses a single SG asset to serve both rendering (visual sky appearance) and lighting (IBL), enabling live derivation of lighting parameters without cubemap re-convolution [S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- An HDR environment map (panorama or cubemap) or a set of hand-authored SG parameters.
- Surface normal $\mathbf{n}$ and view direction $\mathbf{v}$ (for appearance).
- Reflection direction $\mathbf{R}$ and roughness $\alpha$ (for specular).

**Outputs:**
- **Appearance:** Per-pixel sky color (radiance).
- **Diffuse Lighting:** Irradiance $E(\mathbf{n})$.
- **Specular Lighting:** Prefiltered radiance $L(\mathbf{R}, \text{roughness})$.
- **Optional:** Analytic light sources for shadow casting.

### Data Structures
The environment is represented as a list of $M$ SG lobes. Each lobe $k$ is defined by:
- $\boldsymbol{\mu}_k \in S^2$: Lobe axis (direction).
- $\lambda_k \in \mathbb{R}^+$: Lobe sharpness.
- $\mathbf{a}_k \in \mathbb{R}^3_+$: Lobe amplitude (RGB color).

The system supports a layer-based architecture where each layer has a type (SG or Point) and independent channels:
1.  **Appearance:** Weight and Color.
2.  **Diffuse:** Weight and Color.
3.  **Specular:** Weight and Color.
4.  **PromoteToLight:** Boolean flag.

### Formulas
The fundamental SG function is defined as [S1][S2, unofficial]:
$$ G(\omega; \boldsymbol{\mu}, \lambda, \mathbf{a}) = \mathbf{a} \cdot \exp(\lambda (\boldsymbol{\mu} \cdot \omega - 1)) $$
where $\omega \in S^2$ is the input direction, $\boldsymbol{\mu} \in S^2$ is the axis, $\lambda > 0$ is the sharpness, and $\mathbf{a} \in \mathbb{R}^3_+$ is the amplitude.

The environment map $L_i(\omega)$ is the sum of $M$ lobes [S1]:
$$ L_i(\omega) = \sum_{k=1}^{M} G(\omega; \boldsymbol{\mu}_k, \lambda_k, \mathbf{a}_k) $$

**Energy (Integral over sphere)** [S2, unofficial]:
$$ \int_{S^2} G \, d\omega = \mathbf{a} \cdot \frac{2\pi}{\lambda}(1 - e^{-2\lambda}) $$
This is used for normalization and aggregate-to-ambient reduction for point sources.

**Product of two SGs** [S2, unofficial]:
The product of two SGs is another SG, defined by:
$$ \lambda_m = \|\lambda_1 \boldsymbol{\mu}_1 + \lambda_2 \boldsymbol{\mu}_2\| $$
$$ \boldsymbol{\mu}_m = \frac{\lambda_1 \boldsymbol{\mu}_1 + \lambda_2 \boldsymbol{\mu}_2}{\lambda_m} $$
$$ \mathbf{a}_m = \mathbf{a}_1 \mathbf{a}_2 e^{\lambda_m - \lambda_1 - \lambda_2} $$
This property enables the computation of the SG-SG inner product, which approximates the convolution of the environment with a BRDF lobe.

**Diffuse Irradiance** [S2, unofficial]:
$$ E(\mathbf{n}) = \int_{S^2} L(\omega) \max(0, \mathbf{n} \cdot \omega) \, d\omega $$
Computed per-lobe using a closed-form approximation of the SG convolved with the clamped cosine function (referencing Wang et al. 2009).

**Specular Prefiltered Radiance** [S2, unofficial]:
$$ L(\mathbf{R}, \text{roughness}) \approx \sum_i \langle G_{env,i}, G_{brdf} \rangle $$
The GGX BRDF is warped into an SG about the reflection direction $\mathbf{R}$ with sharpness $\lambda_r \approx f(\text{roughness})$. The result is the sum of SG-SG inner products.

### Parameters
- **M (Total Lobes):** 128 is a common default for the environment map representation [S1].
- **Irradiance Lobe Budget:** Approximately 12 lobes are sufficient for diffuse lighting [S2, unofficial].
- **Appearance Lobe Budget:** Approximately 300 lobes are used for visual sky appearance [S2, unofficial].

### Steps and Pseudocode

#### 1. Frequency Split and Initialization
The sky signal is split into low-frequency components (sun, moon, skyglow, clouds) represented by SG lobes, and high-frequency point sources (stars) represented by an additive point layer [S2, unofficial]. Stars are excluded from the lighting bake by default as they carry negligible energy; night illumination is derived from the moon (one bright SG) and skyglow (low-$\lambda$ SGs) [S2, unofficial].

#### 2. Fitting (Baking)
If starting from an HDRI, fit the SG mixture using a greedy or EM algorithm with a perceptual/log-domain loss to prevent the sun from dominating [S2, unofficial].
- **Determinism:** The random seed for initialization must be an explicit asset parameter to ensure byte-identical re-baking [S2, unofficial].

#### 3. Runtime Evaluation
For a given surface point with normal $\mathbf{n}$ and view direction $\mathbf{v}$:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_sg_environment(lobes, point_layer, normal, view_dir, roughness):
    """
    lobes: List of SG parameters (mu, lambda, a)
    point_layer: List of point source parameters (position, intensity)
    normal: Surface normal vector
    view_dir: View direction vector
    roughness: Material roughness
    """
    
    # 1. Appearance (Radiance)
    # Sum of SG lobes + additive point sprites
    appearance = vec3(0.0)
    for lobe in lobes:
        # G(omega; mu, lambda, a) = a * exp(lambda * (mu . omega - 1))
        dot_prod = dot(lobe.mu, view_dir)
        radiance = lobe.a * exp(lobe.lambda_val * (dot_prod - 1.0))
        appearance += radiance
        
    for star in point_layer:
        # Additive point sprites for high-frequency features
        if is_visible(star.position, view_dir):
            appearance += star.intensity
            
    # 2. Diffuse Irradiance
    # Analytic SG convolved with cosine
    diffuse_irradiance = vec3(0.0)
    for lobe in lobes:
        # Closed-form approximation of SG * clamped_cosine
        # See Wang et al. 2009 for specific coefficients
        irradiance_contribution = sg_convolve_cosine(lobe, normal)
        diffuse_irradiance += irradiance_contribution
        
    # Point sources aggregate to ambient (optional, default 0)
    # diffuse_irradiance += aggregate_point_flux(point_layer, normal)

    # 3. Specular Prefiltered Radiance
    # SG * GGX-as-SG inner product
    reflection_dir = reflect(-view_dir, normal)
    specular_radiance = vec3(0.0)
    
    # Warp GGX into an SG about reflection_dir with sharpness based on roughness
    brdf_lobe = warp_ggx_to_sg(reflection_dir, roughness)
    
    for lobe in lobes:
        # Product of two SGs is an SG
        # lambda_m = ||lambda1*mu1 + lambda2*mu2||
        # mu_m = (lambda1*mu1 + lambda2*mu2) / lambda_m
        # a_m = a1 * a2 * exp(lambda_m - lambda1 - lambda2)
        
        combined_lambda = norm(lobe.lambda_val * lobe.mu + brdf_lobe.lambda_val * brdf_lobe.mu)
        combined_mu = (lobe.lambda_val * lobe.mu + brdf_lobe.lambda_val * brdf_lobe.mu) / combined_lambda
        combined_amplitude = lobe.a * brdf_lobe.a * exp(combined_lambda - lobe.lambda_val - brdf_lobe.lambda_val)
        
        # The inner product is the integral of the product SG
        # Integral = combined_amplitude * (2*pi / combined_lambda) * (1 - exp(-2*combined_lambda))
        # However, for prefiltered radiance, we often just use the peak or a simplified inner product
        # The source states: L(R, roughness) approx sum <G_env, G_brdf>
        # The inner product of two SGs G1 and G2 is:
        # <G1, G2> = a1 . a2 * (2*pi / (lambda1 + lambda2)) * exp((lambda1*lambda2 * (mu1.mu2 - 1)) / (lambda1 + lambda2))
        # Note: The source describes the product SG properties, but the inner product formula is standard for SGs.
        # Using the standard SG inner product for approximation:
        dot_mu = dot(lobe.mu, brdf_lobe.mu)
        inner_product = dot(lobe.a, brdf_lobe.a) * (2.0 * PI / (lobe.lambda_val + brdf_lobe.lambda_val)) * \
                        exp((lobe.lambda_val * brdf_lobe.lambda_val * (dot_mu - 1.0)) / (lobe.lambda_val + brdf_lobe.lambda_val))
        specular_radiance += inner_product

    return appearance, diffuse_irradiance, specular_radiance
```

### Edge Cases and Pitfalls
- **Ringing/Blur:** SGs are a smooth low-frequency basis and suffer from ringing or blurring on point-like features (same failure mode as Spherical Harmonics). High-frequency features like stars must not be represented as SG lobes [S2, unofficial].
- **Star Representation:** Representing ~5,000 stars as SG lobes would annihilate the compression win. Stars must be handled as cheap point sprites [S2, unofficial].
- **Determinism:** Greedy/EM initialization in the fitter uses randomness. To ensure determinism, the seed must be an explicit asset parameter so re-baking the same HDRI produces byte-identical results [S2, unofficial].
- **Scope Limitation:** This method is limited to environment IBL (sky reflected in surfaces). Object-to-object reflections (SSR, planar, RT) are explicitly out of scope [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Evaluation is $O(M)$ per pixel, where $M$ is the number of lobes. With $M \approx 128-300$, this is very fast and suitable for real-time rendering.
- **Memory Complexity:** $O(M)$ storage for the lobe parameters. Significantly smaller than high-resolution cubemaps (e.g., 8K–32K HDRI) [S2, unofficial].
- **Performance:** Enables live derivation of lighting. Dragging a sun lobe or channel weight re-derives the lighting in the same frame without cubemap re-convolution, which would take hundreds of milliseconds [S2, unofficial].
- **Strengths:** Closed-form analytic solutions for irradiance and specular; authorable; deterministic; decoupled appearance and lighting channels.
- **Limitations:** Cannot represent high-frequency details (stars) as SGs; limited to environment lighting; not suitable for volumetric scattering or object-to-object reflections [S2, unofficial].

## Background
Spherical Gaussians have been used in AAA games since approximately 2015, specifically in *Physically Based Lighting in The Order: 1886* (Ready at Dawn, SIGGRAPH 2015) [S2, unofficial]. The method builds on the mathematical properties of SGs for efficient approximation of the rendering equation [S1]. The specific application to environment maps with layer/channel decoupling and live derivation is described in recent engine design discussions [S2, unofficial].

## Sources
[S1] PhySG: Inverse Rendering with Spherical Gaussians for Physics-based Material Editing and Relighting, https://doi.org/10.1109/cvpr46437.2021.00541, fetched 2026-10-26
[S2] Spherical-Gaussian sky + authorable environment lighting (umbrella), https://github.com/brooteskan/wozzits/issues/220, fetched 2026-10-28 (unofficial)
