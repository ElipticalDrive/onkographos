# Two-Pass Shadow Casting (Kajiya and Von Herzen)
Also called: Kajiya and Von Herzen Method, Shadow Calculation via Illumination Pass, Shadow Casting via Two-Pass Method, Two-Pass Numerical Algorithm for Shadows, Two-Pass Numerical Shadow Algorithm

## Overview
Two-Pass Shadow Casting is a numerical algorithm for computing shadows in volume rendering by explicitly accounting for the transparency of participating media between a light source and a surface point, and between that point and the viewpoint [S1, unofficial][S2]. It solves the problem of realistic shadowing in scenes containing smoke, fog, clouds, or turbid water, where standard opaque shadow techniques fail to capture the attenuation of light through the volume [S2]. Unlike recursive ray tracing, which sends discrete "shadow feelers" to the light source at each point along the primary ray, this method integrates the shadowing effect continuously along the ray [S2]. It is distinguished from its sibling [Shadow Ray Casting](2796-shadow-ray-casting.md) by its volumetric integration approach and from [Shadow Mapping](2934-shadow-mapping.md) by its direct handling of participating media density rather than depth buffers.

## How it works
The method decomposes the rendering equation into two sequential passes: an illumination pass and a gathering (radiance) pass [S1, unofficial][S2]. It assumes single scattering, meaning it accounts for only one reflection or scattering event from the illumination ray to the observer [S2].

### Inputs and Outputs
*   **Inputs:**
    *   Volume density field $\tau(X)$, defined over a 3D domain.
    *   Light source intensity $L$ (assumed infinite/directional in direction $-\omega'$).
    *   Viewpoint position $X$ and viewing direction $\omega$.
    *   Shading rule $r(X, \omega, \omega')$, typically a phase function (e.g., Henyey-Greenstein) [S1, unofficial][S2].
    *   Background intensity $I_0$.
    *   Ray length $D$ (distance from viewpoint to the far edge of the volume).
*   **Outputs:**
    *   Final pixel intensity $I(X)$.

### Data Structures
*   **Volume Grid:** A 3D array storing the extinction coefficient $\tau$ at voxel centers or corners.
*   **Illumination Buffer:** A 3D array (same dimensions as the volume) storing the computed illumination $i(X, \omega')$ for each voxel. This buffer is the output of Pass 1 and input to Pass 2.

### Numbered Steps

1.  **Pass 1: Compute Illumination $i(X, \omega')$**
    Calculate the illumination reaching each point $X$ in the volume from the light source. For an infinite light source in direction $-\omega'$, the illumination is:
    $$i(X, \omega') = L \exp\left(-\int_0^\infty \tau(X - t\omega') dt\right)$$
    In practice, the integral runs only to the edge of the data volume [S1, unofficial][S2].
    
    *Implementation Detail:* Propagate flux from the light source through the volume one voxel layer at a time. For each layer, account for the layer's transparency before propagating to the next [S1, unofficial][S2]. This is numerically approximated using a Riemann sum or similar integration along rays parallel to $-\omega'$ [S2].

2.  **Pass 2: Gather Radiance $I(X)$**
    Scatter the computed illumination to the viewpoint. The shading rule is defined as:
    $$g(X, \omega) = r(X, \omega, \omega') i(X, \omega')$$
    where $g(X, \omega)$ is the reflected/scattered intensity at $X$ towards the viewpoint [S1, unofficial][S2].
    
    The final intensity $I(X)$ is gathered along the viewing ray from the viewpoint $X$ in direction $\omega$. Using reversed ray parametrization (where $s$ starts at the viewpoint and goes out in direction $-\omega$, reaching $X - s\omega$ at distance $s$), the equation is:
    $$I(X) = I_0 \exp\left(-\int_0^D \tau(X - t\omega) dt\right) + \int_0^D g(X - s\omega) \exp\left(-\int_0^s \tau(X - t\omega) dt\right) ds$$
    
    Substituting the expressions for $g$ and $i$, the full equation becomes:
    $$I(X) = I_0 \exp\left(-\int_0^D \tau(X - t\omega) dt\right) + \int_0^D r(X - s\omega, \omega, \omega') L \exp\left(-\int_0^\infty \tau(X - sw - t\omega') dt\right) \exp\left(-\int_0^s \tau(X - t\omega) dt\right) ds$$
    
    *Implementation Detail:* For each pixel, trace a ray from the viewpoint. At each sample point along the ray, retrieve the pre-computed illumination $i$ from the buffer, apply the phase function $r$, and accumulate the result weighted by the transmittance from the viewpoint to that point [S1, unofficial][S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_pass_shadow_casting(volume, light_direction, view_direction, viewpoint, phase_function, background_intensity):
    """
    volume: 3D array of extinction coefficients tau(X)
    light_direction: vector omega' (direction of light propagation is -omega')
    view_direction: vector omega
    viewpoint: 3D point X
    phase_function: function r(X, omega, omega')
    background_intensity: float I0
    """
    
    # Pass 1: Compute Illumination Buffer
    # Initialize buffer with zeros
    illumination_buffer = create_3d_array_like(volume)
    
    # Propagate light through volume along -light_direction
    # For each voxel X in the volume:
    for X in volume.voxels():
        # Calculate optical depth from light source to X
        # Integral from 0 to infinity (clamped to volume bounds)
        optical_depth = 0.0
        t = 0.0
        step_size = volume.step_size()
        
        # Trace ray from X backwards along light_direction (towards light)
        # Stop when leaving volume
        current_pos = X
        while current_pos is inside_volume():
            # Add contribution of current segment
            tau_val = volume.get(current_pos)
            optical_depth += tau_val * step_size
            
            # Move towards light source
            current_pos = current_pos - light_direction * step_size
            
        # Compute illumination i(X, omega')
        # L is assumed to be 1.0 for normalized intensity, or passed as parameter
        L = 1.0 
        illumination_buffer.set(X, L * exp(-optical_depth))

    # Pass 2: Gather Radiance along viewing rays
    final_image = create_2d_array()
    
    for pixel in image_pixels():
        ray_origin = viewpoint
        ray_dir = view_direction
        
        accumulated_intensity = 0.0
        transmittance_from_viewpoint = 1.0
        t = 0.0
        step_size = volume.step_size()
        
        # Trace ray from viewpoint forward
        # s is distance from viewpoint
        current_pos = ray_origin
        
        while t < D and current_pos is inside_volume():
            # Get pre-computed illumination at this point
            # Note: illumination_buffer stores i(X, omega')
            i_val = illumination_buffer.get(current_pos)
            
            # Apply shading rule: g(X, omega) = r(X, omega, omega') * i(X, omega')
            # r depends on scattering angle between view_dir and light_dir
            g_val = phase_function(current_pos, ray_dir, light_direction) * i_val
            
            # Accumulate scattered light, weighted by transmittance from viewpoint
            # Term: g(X - s*omega) * exp(-int_0^s tau(X - t*omega) dt)
            accumulated_intensity += g_val * transmittance_from_viewpoint * step_size
            
            # Update transmittance from viewpoint to next point
            tau_val = volume.get(current_pos)
            transmittance_from_viewpoint *= exp(-tau_val * step_size)
            
            # Advance ray
            current_pos = current_pos + ray_dir * step_size
            t += step_size
            
        # Add background intensity attenuated by total transmittance
        final_intensity = background_intensity * transmittance_from_viewpoint + accumulated_intensity
        final_image.set(pixel, final_intensity)
        
    return final_image
```

### Parameters and Edge Cases
*   **Light Source:** The derivation assumes an infinite (directional) light source [S1, unofficial][S2]. For point lights, the integral limits and geometry must be adjusted [S2].
*   **Volume Bounds:** The illumination integral $\int_0^\infty$ is truncated to the edge of the data volume [S1, unofficial][S2].
*   **Single Scattering:** The method is a single-scattering model. It is only valid if the albedo or density is low, so that multiple scattering is unlikely [S2]. It is not suitable for highly scattering media like atmospheric clouds where multiple scattering is significant [S2].
*   **Amortization:** The illumination pass can be amortized over several animated frames if only the viewpoint moves, as the illumination $i(X, \omega')$ depends only on the light direction and volume, not the view [S2].

## Subtopics

### [Two-pass Spectral Rendering Integral](5266-two-pass-spectral-rendering-integral.md)
Adapts the Kajiya and von Herzen two-pass algorithm to spectral representation, handling wavelength-dependent opacity and emission spectra for effects like fluorescence.

## Operating characteristics
*   **Time Complexity:** The two-pass method takes only twice as long as the gathering pass without shadows [S1, unofficial][S2]. Pass 1 requires tracing rays parallel to the light direction through the volume; Pass 2 requires tracing rays from the viewpoint.
*   **Memory Complexity:** Requires an additional 3D buffer to store the illumination $i(X, \omega')$ for every voxel in the volume [S1, unofficial][S2].
*   **Strengths:** Provides physically accurate shadows in participating media; amortizable over frames for static volumes; simpler than full recursive ray tracing for single scattering [S1, unofficial][S2].
*   **Limitations:** Single-scattering approximation limits accuracy in dense or highly scattering media (e.g., thick clouds) [S2]; requires significant memory for the illumination buffer; not suitable for dynamic volumes without recomputing Pass 1 [S2].
*   **When to Use:** Photorealistic rendering of opaque objects in smoke, fog, or turbid water where single scattering is a reasonable approximation [S2]. Used in the "Heidelberg ray tracing model" for medical images [S2].
*   **When Not to Use:** Highly scattering media where multiple scattering is dominant; real-time applications with limited memory; dynamic volumes where recomputing illumination is too expensive [S2].

## Background
The two-pass numerical algorithm was suggested by Kajiya and Von Herzen [S1, unofficial][S2]. It was introduced in the context of optical models for direct volume rendering [S1, unofficial][S2]. The method was applied in the "Heidelberg ray tracing model" by Meinzer et al. for medical images [S2]. While its utility in general volume rendering applications has been questioned due to the single-scattering limitation, it remains a foundational technique for shadow calculation in participating media [S2].

## Sources
[S1] Optical Models for Direct Volume Rendering - Duke Computer Science, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S2] Optical models for direct volume rendering (PDF) - Academia.edu, https://www.academia.edu/49467168/Optical_models_for_direct_volume_rendering, fetched 2026-10-27
