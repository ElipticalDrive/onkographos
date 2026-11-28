# In-scattering / Emittance Integral

Also called: Discretized Compositing Scheme, Progressive Refinement In-scattering

## Overview
The In-scattering / Emittance Integral method estimates light scattered into a volume sample from local light sources by discretizing the emittance integral into a compositing scheme [S1, unofficial]. It sets itself apart from its parent, Local Ambient Occlusion (LAO), by applying similar local sampling principles to compute in-scattered and emissive light contributions rather than just shadowing [S1, unofficial]. It builds on [Local Ambient Occlusion (LAO)](4502-local-ambient-occlusion-lao.md) [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume density field, transfer function, light source position/direction, optical properties, and parameters $R_\Omega$ (ray count) and $I_{bias}$ [S1, unofficial].
*   **Output:** In-scattered radiance contribution for each volume sample [S1, unofficial].

**Data Structures:**
*   **Accumulator:** A buffer storing the accumulated in-scattered light, initialized to zero [G].
*   **Direction Set:** A set of $R_\Omega$ sample directions around the light vector [G].

**Parameters:**
*   $R_\Omega$: Number of rays used to sample the hemisphere around the light vector. Typical values are 16 or 48 [S1, unofficial].
*   $I_{bias}$: Minimum light value clamped for a volume sample to prevent dark artifacts. Typical values are 0.0 or 0.2 [S1, unofficial].

**Steps:**
1.  **Initialization:** Initialize the in-scatter accumulator buffer to zero [G].
2.  **Progressive Refinement Loop:** For each rendering pass $k$:
    a.  Compute one light direction $\mathbf{d}_k$ per pass [S1, unofficial].
    b.  For each volume sample point $\mathbf{x}$ along the ray:
        i.  Estimate the in-scattered light contribution using the discretized compositing scheme [S1, unofficial].
        ii. Apply the bias by clamping the computed light value to a minimum of $I_{bias}$ [S1, unofficial].
        iii. Add the biased contribution to the accumulator buffer for sample $\mathbf{x}$ [G].
3.  **Compositing:** Composite the accumulated in-scattered light with direct lighting and other illumination terms [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_in_scattering(volume_density, light_directions, R_Omega, I_bias, sigma_t, step_size, light_source_position):
    # Initialize accumulator buffer to zero [G]
    in_scatter_buffer = [0.0] * len(volume_density)
    # Progressive refinement: one direction per pass [S1, unofficial]
    for k in range(R_Omega):
        direction = light_directions[k]
        # Iterate over volume samples along the ray direction [G]
        for i, x in enumerate(volume_density):
            # Compute transmittance from light source to x along direction [G]
            transmittance = 1.0
            for s in range(len(x)):
                transmittance *= (1.0 - sigma_t[x][s] * step_size)
            # Estimate in-scattered contribution using compositing scheme [S1, unofficial]
            density = volume_density[x]
            # Standard phase function evaluation [G]
            phase_val = max(0.0, dot(direction, normalize(x - light_source_position)))
            # Standard light intensity model [G]
            light_intensity = 1.0 / (dot(x - light_source_position, x - light_source_position) + 1e-6)
            L_in = light_intensity * phase_val * density * transmittance
            # Apply bias: clamp to minimum I_bias [S1, unofficial]
            L_final = max(L_in, I_bias)
            # Accumulate result [G]
            in_scatter_buffer[i] += L_final
    return in_scatter_buffer
```

**Edge Cases and Pitfalls:**
*   **Precision:** Accumulating small values over many passes may lead to precision loss if not using floating-point buffers [G].
*   **Boundaries:** Transmittance calculation near the volume boundary must handle exit/entry correctly [G].
*   **Noise:** Lower $R_\Omega$ values increase noise in the final image [S1, unofficial].

## Subtopics

### Ambient Volume Scattering
A simplified optical model that confines multiple scattering into spheres of finite radii around any given sample point, generalizing local shadowing techniques like volumetric ambient occlusion [S2, unofficial]. It preintegrates light transport using a Monte-Carlo simulation for material parameters (anisotropy, extinction) and stores results in a small preintegration table [S2, unofficial]. At rendering, ambient light transport is estimated via a simple texture lookup on this table [S2, unofficial]. The anisotropy parameter of the phase function can be controlled interactively [S2, unofficial]. This approach solves the issue of far-range scattering effects contributing little radiance due to exponential attenuation by limiting the scattering radius [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(R_\Omega \cdot N)$, where $N$ is the number of volume samples [G].
*   **Memory Complexity:** $O(N)$ for the accumulator buffer [G]. Ambient Volume Scattering requires $O(1)$ additional memory for the preintegration table [S2, unofficial].
*   **Measured Performance:** In [S1, unofficial], piecewise integration took 0.54 s for a $512^3$ volume, compared to 5.1 s for full integration. Local piecewise integration updates take milliseconds [S1, unofficial].
*   **Strengths:** Efficient approximation of in-scattering; interactive control of parameters; reduces noise with higher $R_\Omega$ and $I_{bias}$ [S1, unofficial].
*   **Limitations:** Approximation error increases with fewer rays; bias can introduce artificial brightness [S1, unofficial].
*   **When to Use:** When real-time or near-real-time in-scattering is needed and full integration is too slow [S1, unofficial].
*   **When Not to Use:** When high accuracy for complex scattering is required [S1, unofficial].

## Background
The method was introduced by Patric Ljung at Siemens Corporate Research as part of "Advanced Illumination Techniques for GPU-Based Volume Ray-Casting" [S1, unofficial]. It was developed to improve the accuracy of direct light in volume ray-casting by approximating the emittance integral [S1, unofficial]. The Ambient Volume Scattering variant was published in IEEE SciVis 2013 [S2, unofficial].

## Sources
[S1] GPU-Based Volume Ray-Casting with Advanced Illumination, https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2008/advanced_illumination.pdf, fetched 2026-10-29 (unofficial)
[S2] Volume Rendering, https://cg.ivd.kit.edu/publications/2015/marco/VolumeRendering.pdf, fetched 2026-10-28 (unofficial)
