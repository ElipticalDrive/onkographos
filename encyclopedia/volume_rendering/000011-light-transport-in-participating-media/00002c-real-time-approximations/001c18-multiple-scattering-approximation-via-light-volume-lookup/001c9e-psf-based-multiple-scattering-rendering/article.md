# PSF-based Multiple Scattering Rendering

Also called: PSF convolution method, Point Spread Function method

## Overview
PSF-based Multiple Scattering Rendering is a practical method for rendering volumetric multiple scattering effects by treating the participating medium as a point spread function (PSF) that blurs incident radiance, thereby avoiding the costly direct numerical simulation of the radiative transfer equation [S1]. Unlike diffusion approximation, which is valid only for optically dense homogeneous media and assumes uniform angular distribution, this method handles spatially varying general materials and highly directional effects [S1]. Unlike the method by Narasimhan and Nayar, which is restricted to homogeneous isotropic media, this rendering algorithm allows for spatially varying participating media and accounts for anisotropic sources and visibility effects [S1].

## How it works
The core principle is that light transport in volumes can be formalized as a convolution of incident illumination and the PSF of the medium, where the PSF measures the spreading of incident radiance due to scattering events [S1]. The method relies on the mathematical path integration framework introduced by Tesfa [Tes87] and Perez et al. [PAS03] to describe multiple scattering effects in terms of spatial and angular blurring [S1].

### Inputs and Outputs
*   **Inputs**:
    *   Incident illumination field (light volume).
    *   Medium optical properties: scattering coefficients and phase function [S1].
    *   Medium geometry and spatial variation data.
*   **Outputs**:
    *   Rendered image or volume radiance field including multiple scattering contributions.

### Data Structures
The method utilizes a multi-level light volume representation.
*   **Light Volume**: A 3D grid or texture storing incident radiance.
*   **Blurred Levels**: A set of pre-computed blurred versions of the light volume, each corresponding to a different kernel size (level of detail) representing different degrees of scattering spreading [S1].

### Parameters
*   **Scattering Coefficients**: Basic optical properties of the medium used to express the PSF [S1].
*   **Phase Function**: Basic optical property of the medium used to express the PSF [S1].
*   **Kernel Sizes**: Different sizes of blurring kernels applied to the light volume [S1].

### Steps
1.  **Compute Reduced Intensity**: Calculate the amount of light available for redistribution due to multiple scattering by computing the simple attenuation of the incident illumination [S1].
2.  **Pre-compute Blurred Light Volume**: Apply the point spread function to the light volume by blurring it with kernels of different sizes. Store the light volume at various levels of detail corresponding to these blurs [S1].
3.  **Rendering Lookup**: During rendering, at every point in the volume, compute the amount of spreading (blurring) occurring due to multiple scattering. Look up the result from the pre-computed blurred representation of the light volume based on the computed spreading [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-computation Phase
def precompute_light_volume(incident_light_volume, scattering_coefficients, phase_function):
    """
    Step 1: Compute reduced intensity.
    Step 2: Apply PSF by blurring with different kernel sizes.
    """
    # Compute reduced intensity due to simple attenuation
    # Note: Exact attenuation formula depends on specific optical properties
    # [G] Assuming standard Beer-Lambert law for attenuation
    reduced_intensity = incident_light_volume * exp(-scattering_coefficients * distance)
    
    # Generate blurred levels of detail
    blurred_levels = []
    kernel_sizes = [size1, size2, size3] # Defined by scattering properties
    
    for size in kernel_sizes:
        # Blur the reduced intensity with a kernel of specific size
        # [G] Convolution operation
        blurred = convolve(reduced_intensity, get_kernel(size, phase_function))
        blurred_levels.append(blurred)
        
    return blurred_levels

def render_volume(viewpoint, camera_ray, blurred_levels, scattering_coefficients, phase_function):
    """
    Step 3: Look up result from pre-computed blurred representation.
    """
    # For each sample point along the ray in the volume:
    for point in volume_samples:
        # Compute the amount of spreading (blurring) occurring due to multiple scattering
        # at this specific point based on local optical properties
        spreading_width = compute_spreading_width(point, scattering_coefficients, phase_function)
        
        # Select the appropriate blurred level based on spreading_width
        # [G] Interpolation between levels if necessary
        blurred_level = select_blurred_level(blurred_levels, spreading_width)
        
        # Look up the radiance from the blurred light volume
        # [G] Trilinear or higher-order interpolation in the blurred volume
        multiple_scattering_radiance = sample_volume(blurred_level, point)
        
        # Accumulate radiance
        accumulate_radiance(multiple_scattering_radiance)
```

### Edge Cases and Pitfalls
*   **Computational Cost of Alternatives**: Volumetric Monte Carlo and finite element techniques are computationally expensive, with solution times ranging from a few hours to a few days, making them impractical for usual computer graphics imagery [S1]. This method aims to avoid such costs.
*   **Homogeneous vs. Inhomogeneous**: The method is designed to handle inhomogeneous media, unlike diffusion approximation which is limited to homogeneous media [S1].
*   **Directional Effects**: The method accounts for highly directional effects and visibility, which are not easily handled by simpler analytic models restricted to homogeneous media [S1].

## Operating characteristics
*   **Time Complexity**: The pre-computation phase involves blurring the light volume with multiple kernels, which is generally faster than full radiative transfer simulation. The rendering phase involves lookups and interpolations, which are efficient.
*   **Memory Complexity**: Requires storing multiple levels of blurred light volumes, increasing memory usage compared to a single light volume.
*   **Strengths**: Handles spatially varying participating media, anisotropic sources, and visibility effects. More practical than Monte Carlo methods for real-time or near-real-time applications.
*   **Limitations**: Relies on the accuracy of the PSF approximation. May not capture all complex scattering phenomena as accurately as full simulation.
*   **When to Use**: When realistic multiple scattering effects are needed in spatially varying volumes, and full simulation is too slow.
*   **When Not to Use**: When extreme accuracy is required and computational resources are sufficient for Monte Carlo methods, or when the medium is optically dense and homogeneous (diffusion approximation might be simpler).

## Background
The mathematical path integration framework used to describe multiple scattering effects was introduced by Tesfa [Tes87] and Perez et al. [PAS03] [S1]. The method was presented as a practical approach to computing lighting and volumetric effects in spatially varying inhomogeneous scattering media, taking directionally-varying lighting effects into account [S1]. It addresses the limitation of previous analytic models that were restricted to homogeneous and/or optically dense media [S1].

## Sources
[S1] Practical Rendering of Multiple Scattering Effects in Participating Media, https://cseweb.ucsd.edu/~ravir/HRPIEG.pdf, fetched 2026-10-28
