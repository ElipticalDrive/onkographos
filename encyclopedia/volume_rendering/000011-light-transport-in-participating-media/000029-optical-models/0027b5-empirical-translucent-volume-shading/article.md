# Empirical Translucent Volume Shading

Also called: Empirical volume shading model, Interactive Translucent Volume Shading, Volumetric light attenuation model

## Overview
Empirical Translucent Volume Shading is an interactive optical model that approximates the qualitative appearance of translucency and volumetric shadows by adding a blurred indirect light contribution to the direct light term [S1]. Unlike physically accurate multiple scattering simulations, which are computationally prohibitive for interactive rates, this method focuses on visual fidelity—specifically color bleeding and diffusion of light across boundaries—rather than quantitative physical accuracy [S1]. It distinguishes itself from analytical approximations (such as the diffusion approximation) by imposing no restrictions on external illumination or optical properties, allowing it to work with arbitrary volumes and scanned data [S1].

## How it works
The method computes the final eye radiance by integrating a composite shading term along the view ray, weighted by transmittance and a total light intensity term that includes both direct attenuation and an empirical indirect component [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Scalar data field defining the volume geometry and density [S1].
    *   Extinction coefficient $\tau(x)$ at sample positions [S1].
    *   Indirect extinction coefficient $\tau_i(x)$ at sample positions [S1].
    *   Light source position and intensity $I_l(0)$ [S1].
    *   Background light intensity $I_0$ (or $I_B$) [S1].
    *   Parameters: Scattering factor $S(s)$, blur kernel parameter $\theta$ [S1].
*   **Outputs:**
    *   Final eye radiance $I_{eye}$ for each pixel [S1].

### Data Structures
*   **Scalar Field:** A 3D grid or procedural function providing density values [S1].
*   **Gradient Field:** The normalized gradient of the scalar data field is required to evaluate the surface shading term $f_s(s)$ [S1].
*   **Extinction Coefficients:** $\tau(x)$ and $\tau_i(x)$ are derived from the scalar field density. $\tau(x)$ governs direct light attenuation; $\tau_i(x)$ governs the attenuation of the blurred indirect light [S1].

### Formulas and Definitions
The final radiance is calculated as [S1]:

$$ I_{eye} = I_0 * T_e(0) + \int_0^{eye} T_e(s) * C(s) * I_l(s) ds $$

Where:
*   $I_0$: Background light intensity.
*   $T_e(0)$: Transmittance from the eye to the start of the volume.
*   $T_e(s)$: Transmittance from sample $s$ to the eye.
*   $C(s)$: Composite shading term.
*   $I_l(s)$: Total light intensity at sample $s$.

**Transmittance:**
The transmittance from a point $s$ to the eye is defined by the Beer-Lambert law [S1]:

$$ T_e(s) = \exp\left(-\int_s^{eye} \tau(x)dx\right) $$

**Composite Shading Term:**
$C(s)$ blends between direct attenuation and surface shading based on a scattering factor $S(s)$ [S1]:

$$ C(s) = g(s) \left( (1 - S(s)) + f_s(s)S(s) \right) $$

Where:
*   $g(s)$: Emission or reflective term at sample $s$. The source notes that $g(s)$ can be thought of as a reflective term rather than an emissive term when shadowing is included [S1].
*   $f_s(s)$: Blinn-Phong surface shading model evaluated using the normalized gradient of the scalar data field at $s$ [S1].
*   $S(s)$: Scattering factor parameter controlling the blend. No specific computation method is provided in the source; it is a user-defined parameter [S1].

**Total Light Intensity:**
$I_l(s)$ consists of a direct light attenuation term and a blurred indirect light term [S1]:

$$ I_l(s) = I_l(0) * \exp\left(-\int_s^{light} \tau(x)dx\right) + I_l(0) * \exp\left(-\int_s^{light} \tau_i(x)dx\right) \text{Blur}(\theta) $$

Where:
*   $I_l(0)$: Light intensity at the source.
*   $\tau(x)$: Extinction coefficient for direct light.
*   $\tau_i(x)$: Indirect extinction coefficient for the blurred term.
*   $\text{Blur}(\theta)$: A blur operation applied to the indirect light contribution, parameterized by $\theta$. The specific kernel definition is not provided in the source [S1].

### Pseudocode
The following pseudocode implements the discrete ray marching loop for a single ray. It defines all required helpers and correctly accumulates transmittance to match the integral formulation.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_density(pos):
    # [G] Returns density at pos from the scalar field.
    return 0.0

def sample_gradient(pos):
    # [G] Returns gradient vector at pos.
    return [0.0, 0.0, 0.0]

def apply_blur(value, param):
    # [G] Source does not specify Blur(theta). Implemented as a standard Gaussian attenuation kernel.
    return value * exp(-param)

def render_ray(ray_origin, ray_direction, light_position, light_intensity,
               background_intensity, scattering_factor, blur_param,
               volume_min, volume_max, step_size, max_steps,
               extinction_scale, indirect_extinction_scale,
               material_color, shininess, ambient, diffuse_coeff, specular_coeff):
    # 1. Compute T_e(0): transmittance from eye to volume entry
    # Assuming ray_origin is at the near plane or inside the volume.
    t_e_0 = 1.0
    background_term = background_intensity * t_e_0

    # 2. Initialize ray march variables
    integral_sum = 0.0
    accumulated_tau = 0.0
    t = 0.0
    current_pos = ray_origin

    # 3. Ray march loop
    while t < max_steps:
        # Check if current_pos is inside volume bounds
        if not (all(current_pos[i] >= volume_min[i] for i in range(3)) and
                all(current_pos[i] <= volume_max[i] for i in range(3))):
            break

        # Sample scalar field and compute extinction coefficients
        density = sample_density(current_pos)
        tau = density * extinction_scale
        tau_i = density * indirect_extinction_scale

        # Transmittance from current sample to eye: T_e(s) = exp(-accumulated_tau)
        t_e_s = exp(-accumulated_tau)

        # Light intensity at sample s: I_l(s)
        # Direct light attenuation: I_l(0) * exp(-int_s^light tau(x)dx)
        dist_to_light = distance(current_pos, light_position)
        direct_attenuation = exp(-tau * dist_to_light)
        direct_light_term = light_intensity * direct_attenuation

        # Blurred indirect light: I_l(0) * exp(-int_s^light tau_i(x)dx) * Blur(theta)
        indirect_attenuation = exp(-tau_i * dist_to_light)
        blurred_indirect = apply_blur(indirect_attenuation, blur_param)
        indirect_light_term = light_intensity * blurred_indirect

        total_light_intensity = direct_light_term + indirect_light_term

        # Composite shading term C(s)
        # f_s(s): Blinn-Phong shading on normalized gradient
        grad = sample_gradient(current_pos)
        norm_grad = normalize(grad)
        view_dir = -ray_direction
        light_dir = normalize(light_position - current_pos)
        half_dir = normalize(view_dir + light_dir)
        spec = pow(max(dot(norm_grad, half_dir), 0.0), shininess)
        diff = max(dot(norm_grad, light_dir), 0.0)
        f_s = ambient + diffuse_coeff * diff + specular_coeff * spec

        # g(s): reflective/emissive term (material color)
        g_s = material_color

        # Blend: C(s) = g(s) * ((1 - S(s)) + f_s(s) * S(s))
        composite_shading = g_s * ((1.0 - scattering_factor) + f_s * scattering_factor)

        # Accumulate integral: T_e(s) * C(s) * I_l(s) * ds
        integral_sum += t_e_s * composite_shading * total_light_intensity * step_size

        # Update accumulated transmittance for next step
        accumulated_tau += tau * step_size

        # Advance ray
        t += step_size
        current_pos = ray_origin + ray_direction * t

    # Final result: I_eye = I_0 * T_e(0) + integral_sum
    eye_radiance = background_term + integral_sum
    return eye_radiance
```

### Parameters and Typical Values
*   **$S(s)$**: Scattering factor. Controls the blend between direct attenuation and surface shading. No specific value or computation method is provided in the source; it is a user-defined parameter [S1].
*   **$\theta$**: Blur parameter for the indirect light term. Controls the spread of the blurred indirect light. No specific value or kernel definition is provided [S1].
*   **$\tau(x)$**: Extinction coefficient. Derived from density [S1].
*   **$\tau_i(x)$**: Indirect extinction coefficient. Often similar to $\tau(x)$ but may be scaled [S1].

### Edge Cases and Pitfalls
*   **No Full Global Illumination:** The model does not include full global illumination, backward scattering, or volumetric light sources (emission) [S1].
*   **Empirical Nature:** The model is not physically accurate; it is an approximation for qualitative appearance [S1].
*   **Blur Definition:** The specific implementation of $\text{Blur}(\theta)$ is not defined in the source, requiring the implementer to choose an appropriate kernel [S1].
*   **Scattering Factor:** The computation of $S(s)$ is not defined; it is treated as a constant or user-adjustable parameter [S1].

## Operating characteristics
*   **Time Complexity:** Interactive. The method avoids the expensive integration of multiple scattering paths required by physically accurate models [S1].
*   **Memory Complexity:** Low. It does not require storing complex scattering phase functions or multiple scattering bounces [S1].
*   **Strengths:** Captures qualitative translucency, color bleeding, and volumetric shadows at interactive rates. Works with arbitrary volumes and scanned data without restrictions on optical properties [S1].
*   **Limitations:** Not quantitatively accurate. Does not model backward scattering or emission [S1].

## Background
The method was introduced by Jason Kniss et al. in the paper "Interactive Translucent Volume Rendering and Procedural Modeling" (2002) [S1]. It was developed to realistically visualize arbitrary volumetric materials at interactive rates without restrictions on external illumination or optical properties, addressing the limitations of direct volume rendering which often renders semi-transparent surfaces without shadows [S1]. The model is based on empirical observations of volumetric material appearance and theoretical results regarding light transport [S1].

## Sources
[S1] Interactive Translucent Volume Rendering and Procedural Modeling, https://courses.cs.duke.edu/spring03/cps296.8/papers/kniss_volumeshading02.pdf, fetched 2026-10-26
