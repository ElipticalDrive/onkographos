# Multiple Scattering via Spatial Spreading and Ray Marching

Also called: Path Integral Based Multiple Scattering, Practical Multiple Scattering

## Overview
This method renders multiple scattering effects in volumetric media by approximating the spatial spreading of light using a point spread function (PSF) derived from path integral analysis, rather than simulating individual photon paths [S1, unofficial]. It distinguishes itself from Monte Carlo methods by capturing qualitative features like spatial blurring and shadowing significantly faster, at the cost of higher memory usage and reduced quantitative accuracy regarding phase function shape [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume grid storing density $\rho(\mathbf{x})$; absorption cross-section $C_{abs}$; scattering cross-section $C_{sca}$; precomputed light attenuation volumes (light volume pyramid) for each light source; camera parameters.
*   **Output:** Final image color for each pixel.

**Data Structures:**
1.  **Density Grid:** Stores $\rho(\mathbf{x})$. For a $128^3$ voxel grid, this requires 8 Mb [S1, unofficial].
2.  **Light Volume Pyramid:** Stores reduced intensity/light attenuation for each light source. Requires 32 Mb [S1, unofficial].
3.  **Precomputed $\ell_2$ Volume:** Stores precomputed values for the light path component. Requires 8 Mb [S1, unofficial].
*   **Total Memory:** Approximately 48 Mb for the described setup [S1, unofficial].

**Scattering Coefficients:**
The extinction coefficient $a(\mathbf{x})$ and scattering coefficient $b(\mathbf{x})$ are computed from density:
$$ a(\mathbf{x}) = C_{abs}\rho(\mathbf{x}) $$
$$ b(\mathbf{x}) = C_{sca}\rho(\mathbf{x}) $$
where $\rho(\mathbf{x})$ is the density at point $\mathbf{x}$ [S1, unofficial].

**Algorithm Steps:**
1.  **Preprocessing:** Construct the light volume pyramid and precompute $\ell_2$ values for all light sources. This step is performed once per light source [S1, unofficial].
2.  **Ray Marching Setup:** For each pixel, cast a ray from the camera into the volume. Initialize the camera path length $s_1 = 0$ and the associated light transport term $\ell_1 = 1.0$ [S1, unofficial].
3.  **Adaptive Step Sizing:** Determine the step size for the current sample.
    *   **Single Scattering Component:** Use a small, fixed step size to resolve fine features of the participating media [S1, unofficial].
    *   **Multiple Scattering Component:** Use a progressively larger step size as the ray moves further from the camera. This is valid because the blur size (spatial spreading) monotonically increases with distance [S1, unofficial].
4.  **Sample Integration:** At each step along the ray:
    *   Sample the density $\rho(\mathbf{x})$ and compute $a(\mathbf{x})$ and $b(\mathbf{x})$.
    *   **Camera Path ($\ell_1$):** Update $\ell_1$ based on the transmittance along the camera path segment $s_1$.
    *   **Light Path ($\ell_2$):** Retrieve the precomputed $\ell_2$ value from the light volume pyramid for the current position [S1, unofficial].
    *   **Multiple Scattering Contribution:** Apply the spatial spreading (blurring) approximation using the PSF derived from path integral analysis to the incident radiance. This approximates the effect of light scattering multiple times before reaching the camera [S1, unofficial].
    *   **Accumulation:** Accumulate the color contribution using front-to-back compositing, combining single scattering (computed with fine steps) and multiple scattering (computed with adaptive, coarser steps) [S1, unofficial].
5.  **Termination:** Stop when the ray exits the volume or the accumulated transmittance falls below a threshold.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_multiple_scattering(volume_grid, light_volumes, camera_ray):
    # Precomputed constants
    C_abs = 1.0  # Example value
    C_sca = 1.0  # Example value
    
    # Initialize camera path variables
    s1 = 0.0
    l1 = 1.0
    
    # Output color
    color = (0.0, 0.0, 0.0)
    
    # Step through the volume
    # Note: Step size logic differs for single vs multiple scattering components
    # Single scattering requires small steps; multiple scattering uses adaptive larger steps
    
    while s1 < volume_grid.max_distance:
        # Get current position
        pos = camera_ray.origin + camera_ray.direction * s1
        
        # Sample density
        rho = volume_grid.sample(pos)
        
        # Compute coefficients
        a = C_abs * rho
        b = C_sca * rho
        
        # 1. Single Scattering Contribution (Fine Steps)
        # Requires small step size to resolve fine features
        # Compute transmittance and single scatter term
        # ... (Standard single scattering integration) ...
        
        # 2. Multiple Scattering Contribution (Adaptive Steps)
        # Step size increases as s1 increases (blur size increases)
        
        # Retrieve precomputed light path term
        # l2 is precomputed for each light source and stored in light_volumes
        l2_total = 0.0
        for light in lights:
            l2 = light_volumes[light].sample(pos)
            l2_total += l2
            
        # Apply spatial spreading (PSF blur)
        # The method approximates multiple scattering by blurring incident radiance
        # based on distance from camera.
        blur_width = calculate_blur_width(s1, a, b) # Derived from path integral analysis
        blurred_radiance = apply_psf_blur(l2_total, blur_width)
        
        # Accumulate multiple scattering
        # Note: Exact weighting of single vs multiple scattering is not detailed in sources
        # [G] Assuming a standard front-to-back accumulation where multiple scattering
        # is added to the single scattering term, weighted by the scattering coefficient.
        multiple_scatter_term = b * blurred_radiance * l1
        
        color += multiple_scatter_term * step_size
        
        # Update camera path transmittance
        transmittance = exp(-a * step_size)
        l1 *= transmittance
        
        # Adaptive step size for multiple scattering
        # Step size grows with distance s1
        step_size = calculate_adaptive_step_size(s1)
        
        s1 += step_size
        
    return color
```

**Edge Cases and Pitfalls:**
*   **Phase Function Approximation:** The method considers only the overall statistics of the phase function, not its particular shape. This results in quantitative differences compared to Monte Carlo simulations [S1, unofficial].
*   **Attenuation Calculation:** The computation of multiply scattered light attenuation is not straightforward and may require further investigation for high accuracy [S1, unofficial].
*   **Light Source Count:** Precomputing light attenuation volumes for each light source becomes expensive in terms of time and space if there are many light sources [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Significantly faster than Monte Carlo methods. For a complex inhomogeneous scene (800x800 resolution, 9 samples per pixel), rendering took approximately 30 minutes on a 1 GHz Powerbook G4 [S1, unofficial]. For a rising smoke example, it took ~10 minutes compared to ~3 hours for Monte Carlo [S1, unofficial].
*   **Memory Complexity:** Higher than Monte Carlo. The described implementation uses ~48 Mb (8 Mb density, 32 Mb light volume pyramid, 8 Mb precomputed $\ell_2$) compared to ~8 Mb for Monte Carlo [S1, unofficial].
*   **Scaling:** Performance scales with the number of light sources due to the precomputation of light volumes.
*   **Strengths:** Captures qualitative features like spatial blurring and shadowing efficiently; much faster than Monte Carlo for qualitative results.
*   **Limitations:** Less quantitative accuracy than Monte Carlo; high memory usage; precomputation cost for many lights.
*   **When to Use:** When qualitative multiple scattering effects (glow, blurring) are desired and computational time is a constraint, and memory is available.
*   **When Not to Use:** When quantitative accuracy of phase function effects is critical, or when memory is severely constrained.

## Background
The method was presented by Preim et al. in "Practical Rendering of Multiple Scattering Effects in Participating Media" [S1, unofficial]. It addresses the high computational cost of direct numerical simulation of multiple scattering (e.g., via Monte Carlo ray marching) by leveraging spatial spreading. The core innovation is the derivation of a point spread function (PSF) from path integral analysis, which allows for the approximation of multiple scattering effects through blurring rather than explicit path tracing. This approach was shown to be effective for both homogeneous and inhomogeneous media, such as clouds and smoke [S1, unofficial].

## Sources
[S1] PDF Practical Rendering of Multiple Scattering Effects in ParticipatingMedia, https://www.cs.cornell.edu/courses/cs667/2005sp/readings/premoze04.pdf, fetched 2026-10-28 (unofficial)
