# Two-Pass Shadow Casting (Kajiya and Von Herzen)

Also called: Kajiya and Von Herzen Method, Shadow Calculation via Illumination Pass, Shadow Casting via Two-Pass Method, Two-Pass Numerical Algorithm for Shadows, Two-Pass Numerical Shadow Algorithm

## Overview
Two-Pass Shadow Casting is a numerical algorithm for computing shadows in volume rendering by separating the calculation of incoming illumination from the accumulation of radiance along viewing rays [S1, unofficial][S2]. It solves the problem of realistic shadowing in participating media by propagating light flux through the volume layer-by-layer to determine local illumination, then scattering that illumination toward the viewer [S1, unofficial][S2]. Unlike recursive ray tracing, which sends a shadow feeler to the light source at each point along the primary ray, this method integrates the shadowing effect along the ray, taking only twice as long as a gathering pass without shadows [S2].

## How it works
**Inputs and Outputs**
*   **Input:** A volumetric dataset defining the extinction coefficient $\sigma_t(\mathbf{x})$ (or optical depth density $\tau$), an infinite light source with intensity $L$ in direction $-\boldsymbol{\omega}'$, a viewpoint position $\mathbf{X}$, and a viewing direction $\boldsymbol{\omega}$.
*   **Output:** The final radiance $I(\mathbf{X})$ for each pixel.

**Data Structures**
*   **Volumetric Grid:** A 3D array storing the extinction coefficient $\sigma_t$ or optical depth density.
*   **Illumination Buffer:** A 3D array storing the computed illumination $i(\mathbf{X}, \boldsymbol{\omega}')$ for every voxel in the volume.

**Notation**
*   $\mathbf{X}$: Position of interaction.
*   $\boldsymbol{\omega}$: View ray direction (unit vector).
*   $\boldsymbol{\omega}'$: Light direction (unit vector).
*   $L$: Intensity of the infinite light source.
*   $\tau(\mathbf{x})$: Optical depth density (extinction coefficient) at position $\mathbf{x}$.
*   $r(\mathbf{x}, \boldsymbol{\omega}, \boldsymbol{\omega}')$: Shading rule (phase function/scattering coefficient) at $\mathbf{x}$ for scattering from $\boldsymbol{\omega}'$ to $\boldsymbol{\omega}$.
*   $D$: Total length of the view ray through the volume.
*   $s$: Distance along the view ray from the viewpoint.
*   $t$: Distance along the illumination ray from the point of interest.

**Steps**

1.  **Pass 1: Compute Illumination**
    Calculate the illumination $i(\mathbf{X}, \boldsymbol{\omega}')$ reaching each point $\mathbf{X}$ in the volume from the light source. This is done by propagating flux from the light source through the volume, one voxel layer at a time, accounting for the transparency of each layer before propagating to the next [S1, unofficial][S2].

    The illumination at point $\mathbf{X}$ from an infinite light source in direction $-\boldsymbol{\omega}'$ is given by:
    $$ i(\mathbf{X}, \boldsymbol{\omega}') = L \exp\left(-\int_0^\infty \tau(\mathbf{X} - t\boldsymbol{\omega}') dt\right) $$
    In practice, the integral runs only to the edge of the data volume [S1, unofficial][S2].

2.  **Pass 2: Gather Radiance**
    Scatter the computed illumination to the viewpoint using a shading rule $g(\mathbf{X}, \boldsymbol{\omega}) = r(\mathbf{X}, \boldsymbol{\omega}, \boldsymbol{\omega}') i(\mathbf{X}, \boldsymbol{\omega}')$. Gather the reflected intensity $g(\mathbf{X}, \boldsymbol{\omega})$ along viewing rays [S1, unofficial][S2].

    To simplify the integration, reverse the ray parameterization so that $s$ starts at the viewpoint $\mathbf{X}$ and goes out in direction $-\boldsymbol{\omega}$, reaching $\mathbf{X} - s\boldsymbol{\omega}$ at distance $s$ [S2]. The final intensity $I(\mathbf{X})$ is:
    $$ I(\mathbf{X}) = I_0 \exp\left(-\int_0^D \tau(\mathbf{X} - t\boldsymbol{\omega}) dt\right) + \int_0^D r(\mathbf{X} - s\boldsymbol{\omega}, \boldsymbol{\omega}, \boldsymbol{\omega}') L \exp\left(-\int_0^\infty \tau(\mathbf{X} - s\boldsymbol{\omega} - t\boldsymbol{\omega}') dt\right) \exp\left(-\int_0^s \tau(\mathbf{X} - t\boldsymbol{\omega}) dt\right) ds $$
    where $I_0$ is the background radiance [S1, unofficial][S2].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_pass_shadow_casting(volume, light_direction, light_intensity, view_direction, viewpoint, bounds):
    """
    Computes volumetric shadows using the Kajiya and Von Herzen two-pass method.
    
    Args:
        volume: 3D array of extinction coefficients (sigma_t).
        light_direction: Unit vector pointing FROM light TO scene (-omega').
        light_intensity: Scalar intensity L of the infinite light source.
        view_direction: Unit vector of the view ray (omega).
        viewpoint: 3D position of the camera.
        bounds: Bounding box of the volume.
        
    Returns:
        radiance: Final radiance value for the ray.
    """
    
    # Pass 1: Compute Illumination Buffer
    # Propagate flux from light source through the volume layer by layer.
    # This creates a 3D buffer of illumination i(X, omega') for all voxels.
    illumination_buffer = compute_illumination_pass(volume, light_direction, light_intensity, bounds)
    
    # Pass 2: Gather Radiance along the view ray
    # Ray parameterization: X(s) = viewpoint - s * view_direction
    # Note: The standard formulation uses X - s*omega where s goes from 0 to D.
    # We assume the ray starts at the viewpoint and travels in direction -view_direction
    # to match the integral limits in the source equations.
    
    # Calculate total path length D through the volume for the view ray
    D = calculate_ray_length(viewpoint, view_direction, bounds)
    
    # Discretize the view ray into segments for numerical integration
    # Using Riemann sum approximation as per source [S2]
    num_segments = 100  # Implementation detail: resolution of integration
    delta_s = D / num_segments
    
    accumulated_radiance = 0.0
    current_transmittance = 1.0  # Accumulates exp(-int_0^s tau dt)
    
    # Background term: Light from behind the volume
    # I_0 * exp(-int_0^D tau dt)
    # We calculate the total transmittance of the view ray at the end
    total_view_transmittance = 1.0
    
    for i in range(1, num_segments + 1):
        s = i * delta_s
        pos = viewpoint - s * view_direction
        
        # Check if position is within volume bounds
        if not is_inside_bounds(pos, bounds):
            continue
            
        # Get extinction at current position
        sigma_t = volume.get_extinction(pos)
        
        # Update transmittance for the segment from s-delta_s to s
        # exp(-int_{s-delta_s}^s tau dt) approx exp(-sigma_t * delta_s)
        segment_transmittance = exp(-sigma_t * delta_s)
        current_transmittance *= segment_transmittance
        total_view_transmittance = current_transmittance
        
        # Get illumination at this position from the pre-computed buffer
        # i(X, omega') is the light reaching point X from the light source
        illumination = illumination_buffer.get_illumination(pos)
        
        # Shading rule: r(X, omega, omega') * i(X, omega')
        # For simplicity, assume a constant phase function or retrieve from volume
        # r is often modeled as a Henyey-Greenstein phase function
        scattering_coeff = get_scattering_coefficient(pos, view_direction, light_direction)
        
        # Contribution of in-scattered light at this segment
        # g(X, omega) * exp(-int_0^s tau dt) * ds
        # Note: The integral in the source is ds, so we multiply by delta_s
        contribution = scattering_coeff * illumination * current_transmittance * delta_s
        
        accumulated_radiance += contribution
        
    # Add background term
    # I_0 is typically 0 or the sky color
    background_radiance = 0.0 
    final_radiance = background_radiance * total_view_transmittance + accumulated_radiance
    
    return final_radiance

def compute_illumination_pass(volume, light_direction, light_intensity, bounds):
    """
    Pass 1: Propagate light from the source through the volume.
    """
    # Initialize illumination buffer with zero
    illumination_buffer = create_3d_buffer(bounds)
    
    # Propagate flux one voxel layer at a time
    # This requires iterating through the volume in the direction of the light
    # For each voxel, calculate the accumulated optical depth from the light source edge
    # to the voxel, then apply the exponential attenuation.
    
    # Simplified implementation:
    # For each voxel X in the volume:
    #   Calculate optical depth tau = integral from light edge to X along -light_direction
    #   i(X) = L * exp(-tau)
    
    # Note: The source states this is done "one voxel layer at a time"
    # implying a sweep-line or ray-casting approach from the light's perspective.
    
    for voxel in volume.iterate_voxels():
        pos = voxel.position
        # Integral from 0 to infinity (or volume edge) of tau(X - t*omega') dt
        # In practice, integrate from the light source direction until entering the volume
        optical_depth = compute_optical_depth_from_light(pos, light_direction, volume, bounds)
        illumination = light_intensity * exp(-optical_depth)
        illumination_buffer.set_illumination(pos, illumination)
        
    return illumination_buffer

def compute_optical_depth_from_light(point, light_direction, volume, bounds):
    """
    Computes the optical depth from the light source to the point.
    Light travels in direction -light_direction.
    Ray: R(t) = point + t * (-light_direction) = point - t * light_direction
    We integrate from t=0 to t=infinity (or until leaving volume).
    """
    # Start ray at point and go opposite to light direction
    ray_origin = point
    ray_dir = -light_direction
    
    # Find intersection with volume bounds
    t_min, t_max = intersect_ray_with_bounds(ray_origin, ray_dir, bounds)
    
    # Integrate extinction along the ray from t_min to t_max
    # Since we are going from the point towards the light, we are integrating
    # the path the light took to reach the point.
    # The source formula is int_0^infinity tau(X - t*omega') dt.
    # Here X is the point, omega' is the light direction vector (pointing from light to scene).
    # So X - t*omega' traces back towards the light.
    
    optical_depth = 0.0
    num_samples = 50
    delta_t = (t_max - t_min) / num_samples
    
    for i in range(num_samples):
        t = t_min + i * delta_t
        sample_pos = ray_origin + t * ray_dir
        sigma_t = volume.get_extinction(sample_pos)
        optical_depth += sigma_t * delta_t
        
    return optical_depth
```

**Edge Cases and Pitfalls**
*   **Precision:** The exponential terms can lead to underflow if optical depth is large. Use log-domain arithmetic if necessary [S2].
*   **Boundaries:** The integral for illumination runs to the edge of the data volume, not infinity [S1, unofficial][S2].
*   **Multiple Scattering:** This method is a single-scattering model. It is only valid if the albedo or density is low, so that multiple scattering is unlikely. It is not suitable for atmospheric clouds where multiple scattering is significant [S1, unofficial].
*   **Amortization:** The illumination pass can be amortized over several animated frames if only the viewpoint moves [S2].

## Subtopics

### [Two-pass Spectral Rendering Integral](5266-two-pass-spectral-rendering-integral.md)
The Two-pass Spectral Rendering Integral is a method for computing volumetric shadows in participating media by adapting the Kajiya and von Herzen two-pass algorithm to spectral representation. It separates shadow computation (illumination phase) from radiance accumulation (radiation phase), allowing for the simulation of complex spectral effects such as fluorescence and phosphorescence. It is distinguished from its parent [Two-Pass Shadow Casting (Kajiya and Von Herzen)](2196-two-pass-shadow-casting-kajiya-and-von-herzen.md) by its explicit handling of wavelength-dependent opacity and emission spectra.

## Operating characteristics
*   **Time Complexity:** The two-pass method takes only twice as long as the gathering pass without shadows [S1, unofficial][S2].
*   **Memory Complexity:** Requires storage for the illumination buffer (3D array) in addition to the volume data.
*   **Strengths:** Provides realistic shadows in participating media; illumination pass can be amortized over frames if the viewpoint moves [S2].
*   **Limitations:** Single-scattering model; not valid for high albedo or high density media where multiple scattering is significant (e.g., atmospheric clouds) [S1, unofficial].
*   **When to Use:** When realistic shadows are needed for opaque objects in smoke, fog, clouds, or turbid water, and multiple scattering is negligible [S1, unofficial][S2].
*   **When Not to Use:** For scenes with high density or high albedo where multiple scattering dominates, or when computational resources are extremely limited and shadows are not critical.

## Background
The two-pass numerical algorithm was suggested by Kajiya and Von Herzen [S1, unofficial][S2]. It was used in the "Heidelberg ray tracing model" of Meinzer et al. for medical images [S1, unofficial]. The method is based on the conservation of energy for a differential cylinder of light passing through a medium [S1, unofficial].

## Sources
[S1] [PDF] Optical Models for Direct Volume Rendering - Duke Computer Science, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S2] Optical models for direct volume rendering (PDF) - Academia.edu, https://www.academia.edu/49467168/Optical_models_for_direct_volume_rendering, fetched 2026-10-27
