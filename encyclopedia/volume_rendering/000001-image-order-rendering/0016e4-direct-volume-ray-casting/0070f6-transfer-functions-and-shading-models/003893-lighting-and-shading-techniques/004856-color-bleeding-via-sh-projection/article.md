# Color Bleeding via SH Projection

Also called: Chromaticity Effects, Diffuse Shadowed Radiance Transfer

## Overview
This method computes indirect illumination (color bleeding) in direct volume rendering by projecting the accumulated radiance along random rays onto Spherical Harmonics (SH) basis functions [S1]. It solves the problem of capturing chromaticity effects—where the color of the medium changes as light travels through it—which simpler shading models ignore [S1]. Unlike sibling methods that rely solely on local gradient estimation or absorption-only models, this approach evaluates the full volume rendering integral to capture how the medium's color influences the scattered light [S1].

## How it works
The method extends the standard direct volume ray casting loop by adding a pre-computation or offline phase to estimate the indirect lighting term, which is then used during the final rendering pass [G].

### Inputs and Outputs
*   **Input:** Scalar volume data $V$, transfer functions for base color $C_{base}$ and opacity $\alpha$, optional material function $M$, and SH order $L$ [S1].
*   **Output:** SH coefficients representing the indirect illumination field [S1].

### Data Structures
*   **SH Coefficients:** An array storing coefficients $c_{lm}$ for the spherical harmonics basis functions $Y_{lm}$ at each spatial location [S1].
*   **Ray Data:** Accumulated color $L_{mi}$ and direction $\omega_i$ for each sampled ray [S1].

### Steps
1.  **Ray Casting:** For each voxel or sample point $\mathbf{x}$, cast $N$ rays in random directions $\omega_i$ based on a modified adaptive sampling scheme [S1].
2.  **Volume Integration:** For each ray $\omega_i$, evaluate the front-to-back compositing volume rendering integral to obtain the accumulated color $L_{mi}(\mathbf{x}, \omega_i)$ [S1]. This integral models the color and intensity changes (chromaticity) as the ray travels through the medium [S1]. The integration uses the base color $C_{base}$ and opacity $\alpha$ derived from the transfer functions [G]. Optionally, the material function $M$ can be included in the integration [S1].
3.  **SH Projection:** After completing the raycasting for the current sample, perform the SH projection [S1]. For each ray $i$ with direction $\omega_i$ (defined by angles $\vartheta_s, \phi_s$) and accumulated color $L_{mi}$, update the total sum of the SH projection [S1]. The accumulated color channel values are used as weights for the SH basis functions $Y_{lm}(\vartheta_s, \phi_s)$ [S1]. The update rule for the SH coefficients $c_{lm}$ at location $\mathbf{x}$ is:
    $$ c_{lm}(\mathbf{x}) \leftarrow c_{lm}(\mathbf{x}) + L_{mi}(\mathbf{x}, \omega_i) \cdot Y_{lm}(\vartheta_s, \phi_s) $$
    Note: The weighting with the clamped cosine term (depending on the angle between $\nabla \tau$ and the light vector) is omitted because the modified sampling approach already incorporates similar behavior [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_color_bleeding_sh(volume, transfer_functions, num_rays, sh_order):
    """
    Computes SH coefficients for color bleeding via volume ray casting.
    """
    import math
    import random

    def sample_uniform_sphere():
        u = random.random()
        v = random.random()
        theta = 2.0 * math.pi * u
        phi = math.acos(2.0 * v - 1.0)
        return (math.sin(phi) * math.cos(theta), math.sin(phi) * math.sin(theta), math.cos(phi))

    def cartesian_to_spherical(v):
        theta = math.atan2(v[1], v[0])
        r = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
        phi = math.acos(v[2] / r)
        return theta, phi

    def evaluate_sh(l, m, theta, phi):
        # Standard spherical harmonic basis function Y_lm evaluation [G]
        # Implementation relies on associated Legendre polynomials
        return math.sqrt((2 * l + 1) / (4 * math.pi)) * (math.cos(m * theta) if m >= 0 else math.sin(-m * theta))

    num_voxels = volume.shape[0] * volume.shape[1] * volume.shape[2]
    sh_coeffs = [[0.0] * ((sh_order + 1)**2) for _ in range(num_voxels)]

    for x_idx in range(num_voxels):
        local_sh_sum = [0.0] * ((sh_order + 1)**2)
        current_pos = (0.0, 0.0, 0.0)
        step_size = 1.0

        for _ in range(num_rays):
            omega_i = sample_uniform_sphere()
            L_mi = [0.0, 0.0, 0.0]
            transmittance = 1.0

            while transmittance > 1e-6:
                v_val = volume[current_pos]
                alpha = transfer_functions.opacity(v_val)
                color = transfer_functions.color(v_val)
                contribution = transmittance * alpha * color
                L_mi = [L_mi[j] + contribution[j] for j in range(3)]
                transmittance *= (1.0 - alpha)
                current_pos = tuple(p + omega_i[d] * step_size for d, p in enumerate(current_pos))

            theta, phi = cartesian_to_spherical(omega_i)
            for l in range(sh_order + 1):
                for m in range(-l, l + 1):
                    idx = l * (l + 1) + (m + l)
                    y_lm = evaluate_sh(l, m, theta, phi)
                    local_sh_sum[idx] += L_mi[0] * y_lm

        sh_coeffs[x_idx] = local_sh_sum

    return sh_coeffs
```

### Parameters and Typical Values
*   **Number of Rays ($N$):** Not given in the sources. [G]
*   **SH Order ($L$):** Not given in the sources. [G]
*   **Step Size:** Not given in the sources. [G]

### Edge Cases and Pitfalls
*   **Material Function Exchange:** If the material function $M$ is evaluated during ray marching, it is baked into the SH coefficients [S1]. Changing $M$ requires re-performing the SH projection [S1]. To allow interactive exchange, neglect $M$ during ray marching and use transfer function color, provided the new material has a similar hue [S1].
*   **Cosine Weighting:** Do not apply a clamped cosine weighting based on the angle between the gradient and light vector, as the modified sampling scheme already accounts for this [S1].
*   **Chromaticity:** Unlike methods that only consider absorption (e.g., Ritschel [Rit07]), this method captures chromaticity effects by evaluating the full volume rendering integral [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G]
*   **Memory Complexity:** Not given in the sources. [G]
*   **Strengths:** Captures chromaticity effects (color changes through the medium) which gradient-based shading misses [S1]. Provides better depth separation compared to conventional gradient-based shading [S1].
*   **Limitations:** High memory usage for storing SH coefficients [G]. Non-interactive if material functions are baked into the SH projection [S1]. Computationally expensive due to multiple ray casts per voxel [G].
*   **When to Use:** When accurate color bleeding and chromaticity effects are required, and interactive material editing is not needed or can be handled by using transfer function colors only [S1].
*   **When Not to Use:** When real-time interactive material changes are required and the performance cost of re-computing SH is prohibitive, or when memory is constrained [G].

## Background
This method was introduced by Lindemann and Ropinski in 2010 [S1]. It is an extension of diffuse shadowed radiance transfer, adapted for volume rendering [S1]. It differs from Ritschel's approach [Rit07] by considering the full volume rendering integral rather than just absorption, thereby capturing chromaticity effects [S1]. The method is part of the broader effort to improve lighting and shading in direct volume rendering beyond simple Phong models [G].

## Sources
[S1] Advanced Light Material Interaction for Direct Volume Rendering, https://doi.org/10.2312/vg/vg10/101-108, fetched 2026-10-26
