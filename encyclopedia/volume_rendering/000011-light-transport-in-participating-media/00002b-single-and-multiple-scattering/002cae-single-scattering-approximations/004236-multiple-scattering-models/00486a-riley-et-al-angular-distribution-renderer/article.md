# Riley et al. Angular Distribution Renderer

Also called: Eye Ray Rendering with Angular Distribution, Final Gather with Angular Distribution, Riley et al

## Overview
The Riley et al. Angular Distribution Renderer is a single-pass volume rendering method designed to efficiently simulate forward multiple scattering in atmospheric media, such as clouds and fog, while capturing fine angularly dependent chromatic effects like rainbows and halos [S2]. It distinguishes itself from sibling methods by using pre-computed multiple scattering angular distribution functions to approximate the light distribution after a specific number of scattering events, allowing for efficient hardware-accelerated rendering without the premature darkening artifacts common in low-albedo models [S1, S2].

## How it works
The method renders a volume in a single pass by accumulating inscattered intensity into the eye ray using pre-computed angular distribution functions [S1]. It employs a half-angle slicing scheme to optimize data access for both the light source and the camera [S2].

### Inputs and Outputs
*   **Inputs:**
    *   Volume data (density, scattering coefficients).
    *   Phase function parameters (including the user-adjustable term $\xi$).
    *   View vector $\Omega$ and Light vector $\Omega_{lt}$.
    *   Number of expected scattering events $n(\vec{s})$.
*   **Outputs:**
    *   Eye buffer: Accumulated radiance directed into the eye rays.
    *   Light buffer: Attenuation of light traversing the volume.

### Data Structures
The implementation relies on two primary buffers accessed during the slicing pass [S2]:
1.  **Eye Buffer:** Stores the accumulated radiance $L(\vec{s}, \Omega)$ for each pixel.
2.  **Light Buffer:** Stores the transmittance/attenuation of the light as it passes through the volume.

### Parameters
*   $\xi$: A user-adjustable term added to the phase function to model light received from multiple scattering further along the path, preventing premature darkening [S1].
*   $n(\vec{s})$: The expected number of scattering events reaching sample $\vec{s}$ [S1].
*   $\beta_{sca}$: Scattering coefficient [S1].
*   $\Delta$: Step size [S1].

### Algorithm Steps
1.  **Slicing Orientation:** Slice the volume data at an angle halfway between the view vector $\Omega$ and the light vector $\Omega_{lt}$ [S2]. This orientation is suitable for both eye and light perspectives, optimizing memory access patterns for programmable graphics hardware [S2].
2.  **Buffer Initialization:** Initialize the Eye Buffer and Light Buffer to zero [S2].
3.  **Volume Pass:** Iterate through the volume along the slicing direction. For each sample point $\vec{s}$ along the eye ray:
    a.  Determine the number of scattering events $n(\vec{s})$ expected to reach this sample [S1].
    b.  Calculate the inscattered contribution. The method applies the angular distribution function for $n(\vec{s}) + 1$ scattering events, as the $(n+1)$-th event is the one that scatters light directly into the eye ray [S1].
    c.  Compute the intensity contribution $I(\vec{s})$ using the phase function modified by $\xi$ [S1].
    d.  Accumulate this contribution into the Eye Buffer, weighted by the transmittance from the start of the ray $\vec{s}_f$ to the current sample [S1].
    e.  Update the Light Buffer with the attenuation of light traversing the volume up to this point [S2].
4.  **Final Compositing:** Add skylight and aerial perspective effects based on a simplified single-scattering analytic model to the final image to increase realism [S2]. Store the total transparency of the volume in the alpha channel of the eye buffer to attenuate direct sunlight [S1].

### Formulas
The accumulation for a pixel, assuming front-to-back compositing starting from $\vec{s}_f$, is given by [S1]:

$$ L(\vec{s}, \Omega) = \sum_{i=front}^{back} T(\vec{s}, \vec{s}_f + i\vec{m}\Omega \Delta) I(\vec{s}_f + i\vec{m}\Omega \Delta) $$

where the sum is over sampling slabs $i$ that scatter light contributions towards the eye [S1].

The intensity contribution $I(\vec{s})$ for a sample is approximated by considering the worst-case attenuation for efficiency on graphics hardware, as linear interpolation for floating-point textures was not available at the time of implementation [S1]:

$$ I(\vec{s} + \vec{m}\Omega_{lt} \Delta) = I(\vec{s}) T(\vec{s}, \vec{s} + \vec{m}\Omega_{lt} \Delta) + I(\vec{s})(1 - T(\vec{s}, \vec{s} + \vec{m}\Omega_{lt} \Delta)) \left( A_{ms}^{\lfloor n(\vec{s}_f + i\vec{m}\Omega \Delta) + 1 \rfloor} (\Omega - \Omega_{lt}) + \xi \right) \beta_{sca} \Delta $$

Where:
*   $T(\vec{s}_1, \vec{s}_2)$ is the transmittance between points $\vec{s}_1$ and $\vec{s}_2$.
*   $A_{ms}^{k} (\Omega - \Omega_{lt})$ is the multiple scattering angular distribution function for $k$ scattering events, evaluated at the angle between the view and light directions [S1].
*   $\lfloor \cdot \rfloor$ denotes the floor function, used to determine the discrete number of scattering events for the lookup [S1].
*   $\xi$ is the backscattering correction term [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_angular_distribution(volume, view_dir, light_dir, xi, beta_sca, step_size):
    """
    Renders volume using Riley et al. Angular Distribution Renderer.
    
    Args:
        volume: Volume data structure with density and scattering coefficients.
        view_dir: View vector Omega.
        light_dir: Light vector Omega_lt.
        xi: User-adjustable backscattering term.
        beta_sca: Scattering coefficient.
        step_size: Delta.
        
    Returns:
        eye_buffer: Accumulated radiance.
        light_buffer: Attenuation buffer.
    """
    # Initialize buffers
    eye_buffer = zeros_like(view_dir)
    light_buffer = zeros_like(light_dir)
    
    # Determine slicing direction (halfway between view and light)
    slice_dir = normalize(view_dir + light_dir)
    
    # Iterate through volume along slice direction
    for sample_point in volume.iterate_along(slice_dir, step_size):
        # Calculate expected scattering events at this sample
        # Note: n(s) calculation depends on specific volume density integration
        n_s = calculate_expected_scattering_events(sample_point, volume)
        
        # Determine number of scattering events for angular distribution
        # We use n(s) + 1 because the (n+1)-th event scatters into the eye
        num_scatter_events = n_s + 1
        
        # Get angular distribution function value
        # A_ms is the pre-computed multiple scattering angular distribution
        angle_diff = angle_between(view_dir, light_dir)
        angular_dist = A_ms.lookup(num_scatter_events, angle_diff)
        
        # Calculate transmittance from ray start to current sample
        # T(s, s_f) represents attenuation from the beginning of the ray
        transmittance_to_sample = calculate_transmittance(sample_point, volume.start_point, volume)
        
        # Calculate intensity contribution
        # Using worst-case attenuation approximation for hardware efficiency
        # I(s) = I(s_prev) * T + I(s_prev) * (1 - T) * (AngularDist + xi) * beta_sca * Delta
        # Note: I(s_prev) is the incoming intensity, which includes direct light attenuation
        incoming_intensity = light_buffer.get_attenuation(sample_point)
        
        # Transmittance over the step
        step_transmittance = exp(-volume.density(sample_point) * beta_sca * step_size)
        
        # Insattered light contribution
        # The term (angular_dist + xi) models the phase function with backscattering correction
        inscattered = incoming_intensity * (1 - step_transmittance) * (angular_dist + xi) * beta_sca * step_size
        
        # Accumulate into eye buffer
        # Weighted by transmittance from the start of the ray to the current sample
        eye_buffer.add(sample_point, transmittance_to_sample * inscattered)
        
        # Update light buffer with attenuation
        light_buffer.update_attenuation(sample_point, step_transmittance)
        
    # Add skylight and aerial perspective effects
    eye_buffer.add_skylight(volume, view_dir, light_dir)
    
    return eye_buffer, light_buffer
```

### Edge Cases and Pitfalls
*   **Premature Darkening:** Rendering without backscattered light causes the region where light first strikes the cloud to appear unnaturally dark [S1]. This is corrected by adding the term $\xi$ to the phase function [S1].
*   **Hardware Limitations:** The original implementation avoided linear interpolation for floating-point textures due to hardware limitations of the time, instead using a worst-case attenuation approximation for efficiency [S1].
*   **Accuracy:** The model is least accurate in dense inner regions of the cloud [S1].

## Operating characteristics
*   **Time Complexity:** Single pass through the volume, making it highly efficient for real-time or near-real-time rendering [S2].
*   **Memory Complexity:** Requires storage for Eye Buffer and Light Buffer, plus pre-computed angular distribution functions [S2].
*   **Strengths:** Efficient rendering on programmable graphics hardware; captures fine angularly dependent chromatic effects (rainbows, halos, glories); avoids premature darkening artifacts [S1, S2].
*   **Limitations:** Least accurate in dense inner regions of the cloud; relies on pre-computed angular distribution functions which may limit flexibility for arbitrary phase functions [S1, S2].
*   **When to use:** Suitable for atmospheric phenomena like clouds and fog where forward multiple scattering and chromatic effects are important, and hardware efficiency is a priority [S2].
*   **When not to use:** Not suitable for highly dense, optically thick media where the single-pass approximation breaks down in inner regions [S1].

## Background
The method was proposed by Riley et al. [76] as part of a multiple model system for fast rendering of atmospheric effects [S2]. It was designed to handle the forward multiple scattering behavior of large water and ice atmospheric particles [S2]. The core idea is to use multiple scattering angular distribution functions to approximate the distribution of light after a certain number of scattering events [S2]. It builds on the half-angle slicing scheme introduced by Kniss et al. [45] [S2]. The method is cited in surveys on participating media rendering as an example of efficient, hardware-accelerated approaches for atmospheric phenomena [S1, S2].

## Sources
[S1] Efficient Rendering of Atmospheric Phenomena, https://doi.org/10.2312/egwr/egsr04/375-386, fetched 2026-10-26
[S2] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
