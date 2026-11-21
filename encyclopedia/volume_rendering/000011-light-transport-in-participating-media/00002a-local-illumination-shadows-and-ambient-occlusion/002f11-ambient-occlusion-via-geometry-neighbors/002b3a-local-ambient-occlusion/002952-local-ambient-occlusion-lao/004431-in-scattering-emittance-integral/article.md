# In-scattering / Emittance Integral

Also called: Discretized Compositing Scheme, Progressive Refinement In-scattering

## Overview
The In-scattering / Emittance Integral method estimates the light scattered into a volume sample from local light sources by discretizing the emittance integral into a compositing scheme [S1, unofficial]. It sets itself apart from its parent, Local Ambient Occlusion (LAO), by applying similar local sampling principles to compute in-scattered and emissive light contributions rather than just shadowing [S1, unofficial]. It builds on [Local Ambient Occlusion (LAO)](4502-local-ambient-occlusion-lao.md).

## How it works
**Inputs and Outputs:**
*   **Input:** Volume density field, transfer function, light source position/direction, optical properties (extinction coefficient $\sigma_t$), and parameters $R_\Omega$ (ray count) and $I_{bias}$ [S1, unofficial].
*   **Output:** In-scattered radiance contribution for each volume sample [S1, unofficial].

**Data Structures:**
*   **Ray Buffer:** Stores the direction vectors for the $R_\Omega$ samples. In the progressive refinement variant, only one direction is processed per pass [S1, unofficial].
*   **Accumulator:** A buffer storing the accumulated in-scattered light, initialized to zero [S1, unofficial].

**Parameters:**
*   $R_\Omega$: Number of rays used to sample the hemisphere around the light vector. Typical values are 16 or 48 [S1, unofficial].
*   $I_{bias}$: Minimum light value clamped for a volume sample to prevent dark artifacts. Typical values are 0.0 or 0.2 [S1, unofficial].

**Steps:**
1.  **Initialization:** Initialize the in-scatter accumulator buffer to zero.
2.  **Progressive Refinement Loop:** For each rendering pass $k$:
    a.  Select one light direction $\mathbf{d}_k$ from the set of $R_\Omega$ directions. If $R_\Omega$ directions are not pre-generated, generate one random direction in the hemisphere oriented around the light vector [S1, unofficial].
    b.  For each volume sample point $\mathbf{x}$ along the ray:
        i.  Compute the transmittance $T$ from the light source to $\mathbf{x}$ along direction $\mathbf{d}_k$. This involves integrating the extinction coefficient $\sigma_t$ along the path [S1, unofficial].
        ii. Estimate the in-scattered light contribution $L_{in}$ using the emittance integral approximation. The contribution is weighted by the density and the phase function at $\mathbf{x}$ [S1, unofficial].
        iii. Apply the bias: $L_{final} = \max(L_{in}, I_{bias})$ [S1, unofficial].
        iv. Add $L_{final}$ to the accumulator buffer for sample $\mathbf{x}$ [S1, unofficial].
3.  **Accumulation:** After $R_\Omega$ passes, the accumulator contains the total in-scattered light estimate [S1, unofficial].
4.  **Compositing:** Composite the in-scattered light with the direct lighting and other illumination terms [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_in_scattering(volume, light_directions, R_Omega, I_bias, sigma_t):
    """
    Computes in-scattered light using progressive refinement.
    
    Args:
        volume: 3D array of density values.
        light_directions: List of R_Omega direction vectors.
        R_Omega: Number of rays to sample.
        I_bias: Minimum light value clamp.
        sigma_t: Extinction coefficient field.
        
    Returns:
        in_scatter_buffer: Accumulated in-scattered radiance.
    """
    # Initialize accumulator
    in_scatter_buffer = zeros_like(volume)
    
    # Progressive refinement: one direction per pass
    for k in range(R_Omega):
        direction = light_directions[k]
        
        # For each sample point in the volume (or along rays)
        for x in volume_samples:
            # 1. Compute transmittance from light to x along direction
            # This requires integrating sigma_t along the path from light source to x
            # T = exp(-integral(sigma_t * ds))
            transmittance = compute_transmittance(light_source_position, x, direction, sigma_t)
            
            # 2. Estimate in-scattered contribution
            # L_in = light_intensity * phase_function(direction, x) * density(x) * transmittance
            # Note: Exact phase function and light intensity model depends on specific implementation
            # [S1, unofficial] mentions "emittance integral" but does not provide the full formula
            # We assume a standard single-scattering approximation for the direction
            density = volume[x]
            phase_val = evaluate_phase_function(direction, x) # [G] Standard phase function evaluation
            light_intensity = get_light_intensity(light_source_position, x) # [G] Light model
            
            L_in = light_intensity * phase_val * density * transmittance
            
            # 3. Apply bias
            L_final = max(L_in, I_bias)
            
            # 4. Accumulate
            in_scatter_buffer[x] += L_final
            
    return in_scatter_buffer
```

**Edge Cases and Pitfalls:**
*   **Precision:** Accumulating small values over many passes may lead to precision loss if not using floating-point buffers [S1, unofficial].
*   **Boundaries:** Transmittance calculation near the volume boundary must handle exit/entry correctly [S1, unofficial].
*   **Degenerate Input:** If density is zero, in-scatter should be zero, but bias may still apply depending on implementation [S1, unofficial].

## Subtopics

### Ambient Volume Scattering
A simplified optical model that confines multiple scattering into spheres of finite radii around any given sample point, generalizing local shadowing techniques like volumetric ambient occlusion [S2, unofficial]. It preintegrates light transport using a Monte-Carlo simulation for material parameters (anisotropy, extinction) and stores results in a small preintegration table. At rendering, ambient light transport is estimated via a simple texture lookup on this table [S2, unofficial]. The anisotropy parameter of the phase function can be controlled interactively [S2, unofficial]. This approach solves the issue of far-range scattering effects contributing little radiance due to exponential attenuation by limiting the scattering radius [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(R_\Omega \cdot N)$, where $N$ is the number of volume samples. Progressive refinement allows for incremental updates [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for the accumulator buffer. Ambient Volume Scattering requires $O(1)$ additional memory for the preintegration table [S2, unofficial].
*   **Measured Performance:** In [S1, unofficial], piecewise integration (related to this method) took 0.54 s for a $512^3$ volume, compared to 5.1 s for full integration. Local piecewise integration updates take milliseconds.
*   **Strengths:** Efficient approximation of in-scattering; interactive control of parameters; reduces noise with higher $R_\Omega$ and $I_{bias}$ [S1, unofficial].
*   **Limitations:** Approximation error increases with fewer rays; bias can introduce artificial brightness [S1, unofficial].
*   **When to Use:** When real-time or near-real-time in-scattering is needed and full integration is too slow [S1, unofficial].
*   **When Not to Use:** When high accuracy for complex scattering is required [S1, unofficial].

## Background
The method was introduced by Patric Ljung at Siemens Corporate Research in 2008 as part of "Advanced Illumination Techniques for GPU-Based Volume Ray-Casting" [S1, unofficial]. It was developed to improve the accuracy of direct light in volume ray-casting by approximating the emittance integral. The Ambient Volume Scattering variant was published in IEEE SciVis 2013 by Marco [S2, unofficial].

## Sources
[S1] GPU-Based Volume Ray-Casting with Advanced Illumination, https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2008/advanced_illumination.pdf, fetched 2026-10-29 (unofficial)
[S2] Volume Rendering, https://cg.ivd.kit.edu/publications/2015/marco/VolumeRendering.pdf, fetched 2026-10-28 (unofficial)
