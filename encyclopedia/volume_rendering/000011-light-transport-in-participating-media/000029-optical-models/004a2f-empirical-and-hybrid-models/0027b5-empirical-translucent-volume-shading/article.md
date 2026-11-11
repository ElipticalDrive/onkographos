# Empirical Translucent Volume Shading

Also called: Empirical volume shading model, Interactive Translucent Volume Shading, Volumetric light attenuation model

## Overview
Empirical Translucent Volume Shading is an interactive optical model for volume rendering that captures the qualitative appearance of translucency and volumetric shadows by adding a blurred indirect light contribution to the direct light term [S1]. It solves the problem of rendering semi-transparent volumetric data with realistic lighting at interactive rates without requiring physically accurate multiple scattering simulations or homogeneous optical properties [S1]. Unlike physically accurate methods, it prioritizes visual fidelity over quantitative accuracy, and unlike analytical approximations, it imposes no restrictions on external illumination or complex optical properties [S1].

## How it works
The method computes the final eye radiance by integrating a composite shading term along the view ray, where the shading term blends between direct attenuation and surface shading based on a scattering factor, and the light intensity includes both direct attenuation and a blurred indirect component [S1].

**Inputs and Outputs**
*   **Inputs:** Scalar data field (volume), light intensity $I_l(0)$, light position/direction, view ray parameters, extinction coefficient $\tau(x)$, indirect extinction coefficient $\tau_i(x)$, scattering factor parameter $S(s)$, blur parameter $\theta$, background light $I_0$.
*   **Outputs:** Final eye radiance $I_{eye}$ per sample/pixel.

**Data Structures**
*   **Scalar Field:** 3D array of scalar values representing density or material properties.
*   **Extinction Coefficients:** 3D arrays for $\tau(x)$ and $\tau_i(x)$, derived from the scalar field or provided as auxiliary data.
*   **Gradient Field:** 3D array of normalized gradients $\nabla \hat{f}(s)$ computed from the scalar field for surface shading.

**Numbered Steps**

1.  **Precompute Gradients:** For each sample point $s$ in the volume, compute the gradient of the scalar data field and normalize it to obtain $\nabla \hat{f}(s)$ [S1].
2.  **Compute Surface Shading:** Evaluate the Blinn-Phong shading model $f_s(s)$ using the normalized gradient $\nabla \hat{f}(s)$ and the light direction [S1].
3.  **Compute Composite Shading $C(s)$:** Calculate the composite shading term using the scattering factor $S(s)$ and the emission/reflective term $g(s)$:
    $$C(s) = g(s) \left( (1 - S(s)) + f_s(s)S(s) \right)$$
    Here, $g(s)$ is the emission/reflective term, and $S(s)$ is the scattering factor parameter [S1].
4.  **Compute Direct Light Attenuation:** Calculate the direct light intensity at sample $s$ by attenuating the light source intensity $I_l(0)$ along the path from the light to $s$:
    $$I_{direct}(s) = I_l(0) * \exp\left(-\int_s^{light} \tau(x)dx\right)$$
    where $\tau(x)$ is the extinction coefficient [S1].
5.  **Compute Indirect Light Contribution:** Calculate the indirect light intensity by attenuating the light source intensity $I_l(0)$ along the path from the light to $s$ using the indirect extinction coefficient $\tau_i(x)$, then applying a blur operation $\text{Blur}(\theta)$:
    $$I_{indirect}(s) = I_l(0) * \exp\left(-\int_s^{light} \tau_i(x)dx\right) \text{Blur}(\theta)$$
    where $\text{Blur}(\theta)$ is a blur operation parameterized by $\theta$ [S1].
6.  **Compute Total Light Intensity $I_l(s)$:** Sum the direct and indirect components:
    $$I_l(s) = I_{direct}(s) + I_{indirect}(s)$$
7.  **Compute Transmittance to Eye:** Calculate the transmittance from sample $s$ to the eye:
    $$T_e(s) = \exp\left(-\int_s^{eye} \tau(x)dx\right)$$
    where $\tau(x)$ is the extinction coefficient [S1].
8.  **Integrate along View Ray:** Compute the final eye radiance $I_{eye}$ by integrating from the eye to the start of the volume (parameterized by $s$ from $0$ to $eye$):
    $$I_{eye} = I_0 * T_e(0) + \int_0^{eye} T_e(s) * C(s) * I_l(s) ds$$
    where $I_0$ is the background light intensity and $T_e(0)$ is the transmittance from the eye to the start of the volume [S1].

**Formulas and Definitions**
*   $I_{eye}$: Final eye radiance [S1].
*   $I_0$: Background light intensity [S1].
*   $T_e(s)$: Transmittance from sample $s$ to the eye, defined as $\exp\left(-\int_s^{eye} \tau(x)dx\right)$ [S1].
*   $C(s)$: Composite shading term, defined as $g(s) ((1 - S(s)) + f_s(s)S(s))$ [S1].
*   $g(s)$: Emission/reflective term at sample $s$ [S1].
*   $S(s)$: Scattering factor parameter controlling the blend between direct attenuation and surface shading [S1].
*   $f_s(s)$: Blinn-Phong surface shading model evaluated on the normalized gradient of the scalar data field at $s$ [S1].
*   $I_l(s)$: Total light intensity at sample $s$, defined as $I_l(0) * \exp\left(-\int_s^{light} \tau(x)dx\right) + I_l(0) * \exp\left(-\int_s^{light} \tau_i(x)dx\right) \text{Blur}(\theta)$ [S1].
*   $I_l(0)$: Light intensity at the source [S1].
*   $\tau(x)$: Extinction coefficient at sample $x$ [S1].
*   $\tau_i(x)$: Indirect extinction coefficient at sample $x$ [S1].
*   $\text{Blur}(\theta)$: Blur operation parameterized by $\theta$ [S1].

**Parameters**
*   $S(s)$: Scattering factor. No specific value or computation method is provided in the sources [S1].
*   $\theta$: Parameter for the blur operation. No specific value or kernel definition is provided in the sources [S1].
*   $\tau(x)$: Extinction coefficient. Derived from scalar data or provided [S1].
*   $\tau_i(x)$: Indirect extinction coefficient. Derived from scalar data or provided [S1].

**Edge Cases and Pitfalls**
*   The model does not include full global illumination, backward scattering, or volumetric light sources (emission) as part of its calculation [S1].
*   The sources do not specify how to handle boundaries or degenerate input (e.g., zero gradient) for the Blinn-Phong shading [S1].
*   The sources do not specify the implementation details of the blur operation $\text{Blur}(\theta)$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def empirical_translucent_volume_shading(volume, light_intensity, light_pos, view_ray_origin, view_ray_direction, extinction_coeff, indirect_extinction_coeff, scattering_factor, blur_param, background_light):
    """
    Computes empirical translucent volume shading.
    
    Args:
        volume: 3D array of scalar values.
        light_intensity: Float, I_l(0).
        light_pos: Vector, position of the light source.
        view_ray_origin: Vector, origin of the view ray.
        view_ray_direction: Vector, direction of the view ray.
        extinction_coeff: 3D array, tau(x).
        indirect_extinction_coeff: 3D array, tau_i(x).
        scattering_factor: Function or 3D array, S(s).
        blur_param: Float, theta for blur operation.
        background_light: Float, I_0.
        
    Returns:
        Float, I_eye.
    """
    
    # Precompute gradients for surface shading
    # Note: Gradient computation details are not provided in sources [G]
    gradients = compute_gradients(volume)
    normalized_gradients = normalize(gradients)
    
    # Precompute Blinn-Phong shading for all samples
    # Note: Blinn-Phong implementation details are not provided in sources [G]
    surface_shading = compute_blinn_phong(normalized_gradients, light_pos)
    
    # Precompute transmittance to eye for all samples along the view ray
    # Note: Integration details for transmittance are not provided in sources [G]
    transmittance_to_eye = compute_transmittance(view_ray_origin, view_ray_direction, extinction_coeff)
    
    # Precompute light intensity at all samples along the view ray
    light_intensity_at_samples = []
    for s in samples_along_view_ray(view_ray_origin, view_ray_direction):
        # Direct light attenuation
        # Note: Integration details for direct light are not provided in sources [G]
        direct_attenuation = compute_transmittance(s, light_pos, extinction_coeff)
        direct_light = light_intensity * direct_attenuation
        
        # Indirect light contribution
        # Note: Integration details for indirect light are not provided in sources [G]
        indirect_attenuation = compute_transmittance(s, light_pos, indirect_extinction_coeff)
        indirect_light = light_intensity * indirect_attenuation
        
        # Apply blur operation
        # Note: Blur implementation details are not provided in sources [G]
        blurred_indirect_light = apply_blur(indirect_light, blur_param)
        
        total_light = direct_light + blurred_indirect_light
        light_intensity_at_samples.append(total_light)
        
    # Integrate along view ray
    eye_radiance = background_light * transmittance_to_eye[0]
    for i, s in enumerate(samples_along_view_ray(view_ray_origin, view_ray_direction)):
        # Composite shading
        # Note: Emissive term g(s) is not explicitly defined in sources [G]
        emissive_term = compute_emissive_term(volume, s) # Assumed to be derived from volume
        s_val = scattering_factor(s)
        f_s_val = surface_shading[i]
        composite_shading = emissive_term * ((1 - s_val) + f_s_val * s_val)
        
        # Add to integral
        # Note: Integration method (e.g., trapezoidal, midpoint) is not provided in sources [G]
        eye_radiance += transmittance_to_eye[i] * composite_shading * light_intensity_at_samples[i]
        
    return eye_radiance
```

## Subtopics

### Hybrid Surface-Volume Rendering
Combines surface and volume rendering techniques. Choose when you need to render both opaque surfaces and participating media in the same scene. See [Hybrid Surface-Volume Rendering](12114-hybrid-surface-volume-rendering.md).

## Operating characteristics
*   **Time and Memory Complexity:** The model is designed for interactive rates, implying lower complexity than full multiple scattering simulations [S1]. Memory usage depends on storing the scalar field, extinction coefficients, and gradients.
*   **Measured Performance:** Specific performance metrics are not provided in the sources [S1].
*   **Scaling:** The model scales with the resolution of the volume and the complexity of the blur operation [S1].
*   **Strengths:** Captures qualitative appearance of translucency and volumetric shadows; works with arbitrary volumes and scanned data; no restrictions on external illumination or optical properties; interactive rates [S1].
*   **Limitations:** Does not include full global illumination, backward scattering, or volumetric light sources; empirical approximation may lack quantitative accuracy [S1].
*   **When to Use:** When interactive rendering of translucent volumetric data with realistic lighting is required, and quantitative accuracy is not critical [S1].
*   **When Not to Use:** When full global illumination, backward scattering, or volumetric emission is required, or when quantitative accuracy is critical [S1].

## Background
The model was presented by Kniss et al. in 2002 in the paper "Interactive Translucent Volume Rendering and Procedural Modeling" [S1]. It is based on empirical observations of the appearance of volumetric materials and some recent theoretical results [S1]. The goal was to realistically visualize arbitrary volumetric materials at interactive rates without any restrictions on external illumination or optical properties [S1].

## Sources
[S1] Interactive Translucent Volume Rendering and Procedural Modeling, https://courses.cs.duke.edu/spring03/cps296.8/papers/kniss_volumeshading02.pdf, fetched 2026-10-26
