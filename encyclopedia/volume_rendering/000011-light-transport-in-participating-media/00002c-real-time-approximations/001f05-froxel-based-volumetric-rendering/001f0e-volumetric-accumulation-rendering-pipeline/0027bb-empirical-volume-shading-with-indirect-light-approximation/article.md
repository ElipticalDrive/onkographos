# Empirical Volume Shading with Indirect Light Approximation

Also called: Blurred indirect light contribution, Indirect attenuation model

## Overview
Empirical Volume Shading with Indirect Light Approximation is a real-time shading model for participating media that approximates global illumination by blurring direct light within a local neighborhood cone and applying a separate indirect attenuation term [S1]. It solves the computational intractability of full light transport in participating media by decoupling indirect lighting from high-resolution volume data, allowing for interactive frame rates without pre-computed global illumination caches [S1]. It differs from traditional volume rendering by adding indirect attenuation and local light diffusion, and from full global illumination by restricting scattering to a local blur rather than computing transport from all directions [S1].

## How it works
The method renders volume slices along an axis halfway between the light direction and view direction (or inverted view direction, depending on the dot product sign) to allow a single slice to be processed from both observer and light viewpoints [S1]. It uses three image buffers: one for observer accumulation (typically the frame buffer) and two for light attenuation (current and next light buffers) used in a ping-pong fashion [S1].

### Data Structures
1.  **Observer Buffer**: Accumulates the final color for the eye.
2.  **Current Light Buffer**: Stores direct and indirect light attenuation values for the current slice iteration.
3.  **Next Light Buffer**: Stores computed light attenuation for the subsequent slice iteration.
4.  **Volume Data**: Contains extinction coefficient $\tau(x)$ and indirect extinction coefficient $\tau_i(x)$, or derived opacity $\alpha$ and indirect opacity $\alpha_i$.
5.  **Transfer Function**: Maps volume data to reflective color $g(s)$, surface shading parameter $S(s)$, and opacities.

### Formulas
The eye intensity $I_{eye}$ is computed as [S1]:
$$ I_{eye} = I_0 * T_e(0) + \int_0^{eye} T_e(s) * C(s) * I_l(s) ds \quad (5) $$
Where:
*   $I_0$: Initial light intensity.
*   $T_e(0)$: Eye transmittance from the start of the ray.
*   $T_e(s)$: Eye transmittance to sample $s$.
*   $C(s)$: Reflective color at sample $s$.
*   $I_l(s)$: Light intensity at sample $s$.

The reflective color $C(s)$ is defined as [S1]:
$$ C(s) = g(s) ((1 - S(s)) + f_s(s)S(s)) \quad (6) $$
Where:
*   $g(s)$: Reflective term (diffuse color).
*   $S(s)$: Surface shading parameter ($0 \le S(s) \le 1$).
*   $f_s(s)$: A factor interpolating between surface shading and no surface shading (text implies this term modulates the surface shading contribution; if $S(s)=0$, $C(s)=g(s)$; if $S(s)=1$, $C(s)=g(s)f_s(s)$) [S1].

The light intensity $I_l(s)$ at a sample is the sum of direct and indirect contributions [S1]:
$$ I_l(s) = I_l(0) * \exp\left(-\int_s^{light} \tau(x)dx\right) + I_l(0) * \exp\left(-\int_s^{light} \tau_i(x)dx\right) \text{Blur}(\theta) \quad (7) $$
Where:
*   $I_l(0)$: Light intensity at the source.
*   $\tau(x)$: Direct extinction coefficient.
*   $\tau_i(x)$: Indirect extinction coefficient.
*   $\text{Blur}(\theta)$: A blur operation within a cone of apex angle $\theta$ approximating indirect scattering.

Opacities are derived from extinction coefficients [S1]:
$$ \alpha = \exp(-\tau(x)) \quad (8) $$
$$ \alpha_i = \exp(-\tau_i(x)) \quad (9) $$
Where $\alpha$ is direct opacity and $\alpha_i$ is indirect opacity. The transport color is defined as $1 - \alpha_i$ [S1].

The maximum random noise offset for texture sampling is bounded by [S1]:
$$ offset \leq d \tan\left(\frac{\theta}{2}\right) \quad (10) $$
Where:
*   $d$: Sample distance.
*   $\theta$: User-defined blur angle.

### Parameters
*   **Blur Angle ($\theta$)**: Controls the cone angle for blurring incoming light to approximate indirect illumination [S1].
*   **Surface Shading Parameter ($S(s)$)**: Interpolates between surface shading and no surface shading. Can be user-specified via transfer function or automatically set based on gradient magnitude (high gradient implies surface-like) [S1].
*   **Transport Color**: $1 - \alpha_i$, preferred over indirect alpha for user specification as it represents the color indirect light becomes after attenuation [S1].
*   **Indirect Samples**: Limited by texture units; typically 4 samples are used [S1].

### Pseudocode
The following pseudocode describes the rendering loop for a single slice, assuming the slice axis is computed as the half-angle between light and view directions [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_slice(observer_buffer, current_light_buffer, next_light_buffer, 
                 volume_data, transfer_function, light_direction, view_direction, 
                 blur_angle, noise_texture, num_indirect_samples):
    """
    Renders a single slice for both observer accumulation and light attenuation.
    Uses ping-pong blending for light buffers.
    """
    
    # 1. Determine slice axis
    # Axis is halfway between light and view, or light and inverted view
    if dot(light_direction, view_direction) < 0:
        slice_axis = normalize(light_direction + view_direction)
    else:
        slice_axis = normalize(light_direction - view_direction)
        
    # 2. First Pass: Render from Observer's Point of View
    # Accumulate light into observer buffer
    for each pixel in slice:
        # Ray march through volume along slice axis
        eye_intensity = 0.0
        eye_transmittance = 1.0
        
        for each sample s along ray:
            # Get volume properties
            tau = volume_data.get_extinction(s)
            tau_i = volume_data.get_indirect_extinction(s)
            
            # Compute opacities
            alpha = exp(-tau)
            alpha_i = exp(-tau_i)
            
            # Get transfer function results
            reflective_color_g = transfer_function.get_reflective_color(s)
            surface_shading_S = transfer_function.get_surface_shading_param(s)
            # f_s(s) is assumed to be 1.0 or derived from transfer function if not specified
            # Based on Eq 6: C(s) = g(s) * ((1 - S(s)) + f_s(s) * S(s))
            # If f_s is not explicitly defined in TF, assume standard diffuse reflection modulation
            # Note: The source text mentions f*s*(s) but does not define it further. 
            # We assume it is part of the material definition or 1.0 for basic diffuse.
            f_s = 1.0 
            reflective_color = reflective_color_g * ((1 - surface_shading_S) + f_s * surface_shading_S)
            
            # Get light intensity at sample s from current light buffer
            # This requires sampling current_light_buffer at the slice position
            light_intensity_at_s = current_light_buffer.sample(s)
            
            # Accumulate: I_eye = I_0 * T_e(0) + integral(T_e(s) * C(s) * I_l(s) ds)
            # In discrete steps:
            contribution = eye_transmittance * reflective_color * light_intensity_at_s
            eye_intensity += contribution * step_size
            
            # Update eye transmittance
            eye_transmittance *= alpha
            
            if eye_transmittance < epsilon:
                break
                
        # Blend into observer buffer using alpha
        observer_buffer.blend(pixel, eye_intensity, alpha)

    # 3. Second Pass: Render from Light's Point of View
    # Compute light attenuation for the next slice
    for each pixel in slice:
        # Ray march from light towards volume along slice axis (reverse direction)
        light_flux = 0.0
        light_transmittance = 1.0
        
        # Compute blurred indirect contribution
        # Sample current light buffer multiple times with random offsets
        blurred_flux = 0.0
        for i in range(num_indirect_samples):
            # Calculate random offset bounded by blur angle
            # offset <= d * tan(theta / 2)
            noise_sample = noise_texture.sample(random_coords)
            offset_magnitude = noise_sample * d * tan(blur_angle / 2)
            offset_direction = random_vector_in_cone(blur_angle)
            sample_pos = s + offset_direction * offset_magnitude
            
            # Sample current light buffer at offset position
            flux_sample = current_light_buffer.sample(sample_pos)
            blurred_flux += flux_sample
            
        blurred_flux /= num_indirect_samples
        
        # Compute direct attenuation along the path from light to sample
        # I_l(s) = I_l(0) * exp(-int_s^light tau(x)dx) + ...
        # We accumulate direct attenuation for the next buffer
        direct_attenuation = 1.0
        for each sample x from light to current slice:
            tau = volume_data.get_extinction(x)
            direct_attenuation *= exp(-tau)
            
        # The indirect term in Eq 7 is: I_l(0) * exp(-int_s^light tau_i(x)dx) * Blur(theta)
        # However, the implementation description says:
        # "The blurred inward flux is attenuated using alpha_i and written to the RGB components of the next light buffer."
        # And "The alpha value from the current light buffer ... is blended with the alpha from the transfer function 
        # to compute the direct attenuation and stored in the alpha component of the next light buffer."
        
        # So, Next Light Buffer RGB = Blurred Flux * Indirect Attenuation
        # Next Light Buffer Alpha = Direct Attenuation (accumulated)
        
        # Get indirect attenuation for current sample
        tau_i = volume_data.get_indirect_extinction(s)
        alpha_i = exp(-tau_i)
        
        # Write to next light buffer
        next_light_buffer.set_rgb(pixel, blurred_flux * (1 - alpha_i)) # Using transport color 1-alpha_i? 
        # Source says: "blurred inward flux is attenuated using alpha_i". 
        # Usually attenuation means multiply by alpha_i or (1-alpha_i)? 
        # Eq 7 has exp(-int tau_i). This is the transmittance for indirect light.
        # If alpha_i = exp(-tau_i), then the term is alpha_i * Blur.
        # But source also says "transport color which is 1 - alpha_i ... is the color the indirect light will become".
        # Let's stick to the formula: I_l includes exp(-int tau_i). This is transmittance.
        # So we multiply blurred flux by alpha_i (transmittance).
        next_light_buffer.set_rgb(pixel, blurred_flux * alpha_i)
        
        # Compute direct attenuation for alpha channel
        # Blend current light buffer alpha with transfer function alpha
        current_alpha = current_light_buffer.get_alpha(pixel)
        new_alpha = current_alpha * alpha # Standard alpha blending for transmittance
        next_light_buffer.set_alpha(pixel, new_alpha)

    # 4. Ping-Pong Swap
    # After the second pass, swap current and next light buffers
    temp = current_light_buffer
    current_light_buffer = next_light_buffer
    next_light_buffer = temp
```

### Edge Cases and Pitfalls
*   **Coarse Sampling Artifacts**: Using a limited number of texture units (e.g., 4) for indirect light sampling causes visible artifacts. Mitigated by randomizing sample offsets using a noise texture [S1].
*   **Precision**: The method relies on accurate accumulation of transmittance. Floating-point precision in the light buffers may limit the dynamic range of indirect lighting [G].
*   **Boundary Conditions**: The blur operation assumes a local neighborhood. At volume boundaries, the blur cone may extend outside the volume, requiring clamping or zero-padding [G].

## Subtopics

### Half-Angle Slicing
A technique where the slice axis is halfway between the light and view directions, allowing a single slice to be rendered from both viewpoints. This avoids the need for pre-computed shadow maps [S1].

### Ping-Pong Blending
A buffer management strategy using two light buffers (current and next) that are swapped after each slice pass to store light attenuation values iteratively [S1].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of slices and samples per slice. The cost is dominated by the multiple texture samples for indirect light (limited by texture units) and the volume ray marching [S1].
*   **Memory Complexity**: Requires three image buffers (one observer, two light buffers) plus volume data. The size of light buffers depends on the slice resolution, not the full 3D volume resolution [S1].
*   **Performance**: Suitable for interactive frame rates due to the approximation of indirect lighting via local blur rather than full global illumination [S1].
*   **Strengths**: Decouples lighting resolution from volume data resolution, allowing high-frequency displacement effects to be lit accurately [S1]. Intuitive user parameters (transport color, blur angle) [S1].
*   **Limitations**: Approximation of indirect lighting may lack accuracy for complex global illumination effects. Limited by the number of texture units for indirect sampling [S1].

## Background
The method was introduced by Kniss et al. in 2002 as part of "Interactive Translucent Volume Rendering and Procedural Modeling" [S1]. It addresses the difficulty of solving light transport in participating media, which is computationally expensive for interactive applications. The approach builds on half-angle slicing proposed for volume shadow computation [S1].

## Sources
[S1] Interactive Translucent Volume Rendering and Procedural Modeling, https://courses.cs.duke.edu/spring03/cps296.8/papers/kniss_volumeshading02.pdf, fetched 2026-10-26
