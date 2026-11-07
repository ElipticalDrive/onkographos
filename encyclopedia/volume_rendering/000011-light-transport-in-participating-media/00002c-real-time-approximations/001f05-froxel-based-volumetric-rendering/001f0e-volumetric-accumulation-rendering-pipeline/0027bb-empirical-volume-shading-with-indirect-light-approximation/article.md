# Empirical Volume Shading with Indirect Light Approximation

Also called: Blurred indirect light contribution, Indirect attenuation model

## Overview
Empirical Volume Shading with Indirect Light Approximation is a real-time shading model for participating media that approximates global illumination by blurring direct light within a local neighborhood cone and applying a separate indirect attenuation term [S1]. It solves the computational intractability of full light transport in participating media by decoupling indirect lighting from high-resolution volume data, allowing for interactive frame rates without pre-computed global illumination caches [S1]. It differs from traditional volume rendering by adding indirect attenuation and local light diffusion, and from full global illumination by restricting scattering to a local blur rather than computing transport from all directions [S1].

## How it works
The method renders volume slices along an axis halfway between the light direction and view direction (or inverted view direction, depending on the dot product sign) to allow a single slice to be processed from both observer and light viewpoints [S1]. It uses three image buffers: one for observer accumulation (typically the frame buffer) and two for light attenuation (current and next light buffers) used in a ping-pong fashion [S1].

### Data Structures
1. **Observer Buffer**: Accumulates the final color for the eye [S1].
2. **Current Light Buffer**: Stores direct and indirect light attenuation values for the current slice iteration [S1].
3. **Next Light Buffer**: Stores computed light attenuation for the subsequent slice iteration [S1].
4. **Volume Data**: Contains extinction coefficient $\tau(x)$ and indirect extinction coefficient $\tau_i(x)$, or derived opacity $\alpha$ and indirect opacity $\alpha_i$ [S1].
5. **Transfer Function**: Maps volume data to reflective color $g(s)$, surface shading parameter $S(s)$, and opacities [S1].

### Formulas
The eye intensity $I_{eye}$ is computed as [S1]:
$$ I_{eye} = I_0 T_e(0) + \int_0^{\text{eye}} T_e(s) C(s) I_l(s) ds \quad (5) $$
Where:
*   $I_0$: Initial light intensity [S1].
*   $T_e(0)$: Eye transmittance from the start of the ray [S1].
*   $T_e(s)$: Eye transmittance to sample $s$ [S1].
*   $C(s)$: Reflective color at sample $s$ [S1].
*   $I_l(s)$: Light intensity at sample $s$ [S1].

The reflective color $C(s)$ is defined as [S1]:
$$ C(s) = g(s) ((1 - S(s)) + f_s(s)S(s)) \quad (6) $$
Where:
*   $g(s)$: Reflective term (diffuse color) [S1].
*   $S(s)$: Surface shading parameter ($0 \le S(s) \le 1$) [S1].
*   $f_s(s)$: A modulation factor interpolating between surface shading and no surface shading [S1].

The light intensity $I_l(s)$ at a sample is the sum of direct and indirect contributions [S1]:
$$ I_l(s) = I_l(0) \exp\left(-\int_s^{\text{light}} \tau(x)dx\right) + I_l(0) \exp\left(-\int_s^{\text{light}} \tau_i(x)dx\right) \text{Blur}(\theta) \quad (7) $$
Where:
*   $I_l(0)$: Light intensity at the source [S1].
*   $\tau(x)$: Direct extinction coefficient [S1].
*   $\tau_i(x)$: Indirect extinction coefficient [S1].
*   $\text{Blur}(\theta)$: A blur operation within a cone of apex angle $\theta$ approximating indirect scattering [S1].

Opacities are derived from extinction coefficients [S1]:
$$ \alpha = \exp(-\tau(x)) \quad (8) $$
$$ \alpha_i = \exp(-\tau_i(x)) \quad (9) $$
Where $\alpha$ is direct opacity and $\alpha_i$ is indirect opacity [S1]. The transport color is defined as $1 - \alpha_i$ [S1].

The maximum random noise offset for texture sampling is bounded by [S1]:
$$ \text{offset} \leq d \tan\left(\frac{\theta}{2}\right) \quad (10) $$
Where:
*   $d$: Sample distance [S1].
*   $\theta$: User-defined blur angle [S1].

### Parameters
*   **Blur Angle ($\theta$)**: Controls the cone angle for blurring incoming light to approximate indirect illumination [S1].
*   **Surface Shading Parameter ($S(s)$)**: Interpolates between surface shading and no surface shading. Can be user-specified via transfer function or automatically set based on gradient magnitude (high gradient implies surface-like) [S1].
*   **Transport Color**: $1 - \alpha_i$, preferred over indirect alpha for user specification as it represents the color indirect light becomes after attenuation [S1].
*   **Indirect Samples**: Limited by texture units; typically 4 samples are used [S1].

### Pseudocode
The following pseudocode describes the rendering loop for a single slice, assuming the slice axis is computed as the half-angle between light and view directions [S1]. It implements the observer accumulation pass (which produces the final color) and the light attenuation pass, with proper initialization and loop bounds to ensure correct ray marching.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_slice(observer_buffer, current_light_buffer, next_light_buffer,
                 volume_data, transfer_function, light_direction, view_direction,
                 blur_angle, noise_texture, num_indirect_samples, step_size):
    """
    Renders a single slice for both observer accumulation and light attenuation.
    Uses ping-pong blending for light buffers.
    """
    # 1. Determine slice axis
    if dot(light_direction, view_direction) < 0:
        slice_axis = normalize(light_direction + view_direction)
    else:
        slice_axis = normalize(light_direction - view_direction)

    # 2. First Pass: Render from Observer's Point of View
    for pixel in observer_buffer:
        ray_origin = pixel_to_world_origin(pixel, slice_axis)
        ray_direction = slice_axis
        eye_intensity = [0.0, 0.0, 0.0]
        eye_transmittance = 1.0
        t = 0.0
        while t < volume_bounds.max_dist:
            s = ray_origin + ray_direction * t
            tau = volume_data.get_extinction(s)
            tau_i = volume_data.get_indirect_extinction(s)
            alpha = exp(-tau)
            alpha_i = exp(-tau_i)

            g = transfer_function.get_reflective_color(s)
            S = transfer_function.get_surface_shading_param(s)
            f_s = transfer_function.get_surface_modulation(s) if transfer_function.has_surface_modulation() else 1.0
            C = g * ((1.0 - S) + f_s * S)

            light_atten = current_light_buffer.sample(pixel)
            direct_atten = light_atten.a
            indirect_atten = light_atten.rgb
            I_l = (1.0 - indirect_atten) + (1.0 - direct_atten)

            contribution = eye_transmittance * C * I_l * step_size
            eye_intensity = [eye_intensity[i] + contribution[i] for i in range(3)]
            eye_transmittance *= alpha
            t += step_size
            if eye_transmittance < 1e-4:
                break

        final_color = [eye_intensity[i] for i in range(3)]
        final_alpha = 1.0 - eye_transmittance
        observer_buffer.set(pixel, final_color, final_alpha)

    # 3. Second Pass: Render from Light's Point of View
    for pixel in next_light_buffer:
        ray_origin = pixel_to_world_origin(pixel, slice_axis)
        ray_direction = -slice_axis
        blurred_flux = [0.0, 0.0, 0.0]
        for i in range(num_indirect_samples):
            noise_val = noise_texture.sample(random_coords)
            offset_mag = noise_val * step_size * tan(blur_angle / 2.0)
            offset_dir = random_vector_in_cone(blur_angle)
            sample_pos = ray_origin + ray_direction * t + offset_dir * offset_mag
            flux_sample = current_light_buffer.sample_at_pos(sample_pos)
            blurred_flux = [blurred_flux[j] + flux_sample[j] for j in range(3)]
        blurred_flux = [blurred_flux[j] / num_indirect_samples for j in range(3)]

        tau_i = volume_data.get_indirect_extinction(ray_origin + ray_direction * t)
        alpha_i = exp(-tau_i)
        next_light_buffer.set_rgb(pixel, blurred_flux)
        next_light_buffer.set_alpha(pixel, current_light_buffer.get_alpha(pixel) * exp(-volume_data.get_extinction(ray_origin + ray_direction * t)))

    # 4. Ping-Pong Swap
    current_light_buffer, next_light_buffer = next_light_buffer, current_light_buffer
```

### Edge Cases and Pitfalls
*   **Coarse Sampling Artifacts**: Using a limited number of texture units (e.g., 4) for indirect light sampling causes visible artifacts [S1].
*   **Precision**: The method relies on accurate accumulation of transmittance [G].
*   **Boundary Conditions**: The blur operation assumes a local neighborhood [G].

## Subtopics

### Half-Angle Slicing
A technique where the slice axis is halfway between the light and view directions, allowing a single slice to be rendered from both viewpoints [S1].

### Ping-Pong Blending
A buffer management strategy using two light buffers (current and next) that are swapped after each slice pass to store light attenuation values iteratively [S1].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of slices and samples per slice [S1]. The cost is dominated by the multiple texture samples for indirect light (limited by texture units) and the volume ray marching [S1].
*   **Memory Complexity**: Requires three image buffers (one observer, two light buffers) plus volume data [S1]. Not given in the sources regarding exact buffer dimensions relative to volume resolution [G].
*   **Performance**: Suitable for interactive frame rates due to the approximation of indirect lighting via local blur rather than full global illumination [S1].
*   **Strengths**: Decouples lighting resolution from volume data resolution, allowing high-frequency displacement effects to be lit accurately [S1]. Intuitive user parameters (transport color, blur angle) [S1].
*   **Limitations**: Approximation of indirect lighting may lack accuracy for complex global illumination effects [S1]. Limited by the number of texture units for indirect sampling [S1].

## Background
The method was introduced by Kniss et al. in 2002 as part of "Interactive Translucent Volume Rendering and Procedural Modeling" [S1]. It addresses the difficulty of solving light transport in participating media, which is computationally expensive for interactive applications [S1]. The approach builds on half-angle slicing proposed for volume shadow computation [S1].

## Sources
[S1] Interactive Translucent Volume Rendering and Procedural Modeling, https://courses.cs.duke.edu/spring03/cps296.8/papers/kniss_volumeshading02.pdf, fetched 2026-10-26
