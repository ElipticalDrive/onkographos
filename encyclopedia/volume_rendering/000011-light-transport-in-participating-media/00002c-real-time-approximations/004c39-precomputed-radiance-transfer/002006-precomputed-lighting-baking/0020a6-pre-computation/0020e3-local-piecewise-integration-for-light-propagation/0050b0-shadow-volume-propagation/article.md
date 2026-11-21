# Shadow Volume Propagation

Also called: Ropinski et al

## Overview
Shadow volume propagation is an illumination model for volume rendering that approximates light propagation direction to generate shadows and simulate scattering within volumetric datasets [S1, unofficial]. It decouples shadow computation from scattering to achieve hard shadow borders, which improves depth perception [S2, unofficial]. Unlike half-angle slicing, it specifically separates chromaticity and luminance processing [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volumetric dataset (density, absorption, scattering coefficients), light direction $\vec{\omega}_l$, viewing direction $\vec{\omega}_o$, cone apex angle $\theta$.
*   **Outputs:** Final radiance $L(x, \vec{\omega}_o)$ for each sample point $x$ along the viewing ray.

**Data Structures:**
*   **Auxiliary Volume:** A 3D volume storing precomputed luminance and scattering information. This volume is computed once per light direction and reused for all viewing rays [S1, unofficial].

**Formulas:**
The radiance $L(x, \vec{\omega}_o)$ at point $x$ in viewing direction $\vec{\omega}_o$ is given by [S2, unofficial]:

$$
L(x, \vec{\omega}_o) = L(x_0, \vec{\omega}_o)\tau(x, x_0) + \int_{x_0}^{x} \tau(x, x')L_e(x', \vec{\omega}_o)dx' + \int_{x_0}^{x} \tau(x, x')L_i(x', \vec{\omega}_o)dx'
$$

Where:
*   $x_0$ is the entry point of the viewing ray into the volume.
*   $\tau(x, x')$ is the transmittance between $x$ and $x'$.
*   $L_e$ is emitted radiance.
*   $L_i$ is in-scattered radiance.

The in-scattered radiance $L_i$ is approximated using the auxiliary volume [S1, unofficial]:

$$
L_i(x', \vec{\omega}_o) = \lambda(L_b(x', \vec{\omega}_l))
$$

Where:
*   $L_b$ is the blurred incoming light.
*   $\lambda$ extracts luminance.
*   $\gamma$ extracts chromaticity (used in the blurring step).

**Blurring Model:**
The method assumes a cone phase function to limit incoming light directions to a cone with apex angle $\theta$ centered on the light direction $\vec{\omega}_l$ [S2, unofficial]:

$$
f(x, \vec{\omega}_i, \vec{\omega}_o) = 
\begin{cases} 
\frac{1}{2\pi(1-\cos(\theta))} & \text{if } \vec{\omega}_i \cdot \vec{\omega}_l < \cos(\theta) \\
0 & \text{otherwise}
\end{cases}
$$

**Algorithm Steps:**
1.  **Precomputation:** Compute the auxiliary volume storing luminance and scattering information for the given light direction [S1, unofficial].
2.  **Chromaticity-Luminance Split:** Decompose the incoming light into luminance and chromaticity components [S2, unofficial].
3.  **Blurring:** Apply blurring to the chromaticity part of the incoming light within the cone defined by $\theta$. Do **not** blur the luminance component [S2, unofficial].
4.  **Rendering:** For each viewing ray:
    a. Sample the volume along the ray.
    b. Retrieve the precomputed luminance/scattering data from the auxiliary volume [S1, unofficial].
    c. Modulate the Phong shading at each sample using the retrieved data [S1, unofficial].
    d. Combine the unblurred luminance with the blurred chromaticity to compute final radiance [S2, unofficial].

**Parameters:**
*   $\theta$ (cone apex angle): Not given in the sources.

**Pitfalls:**
*   Decoupling luminance and chromaticity is critical for maintaining hard shadow borders. Blurring luminance would soften shadows and reduce depth perception [S2, unofficial].

## Operating characteristics
*   **Strengths:** Produces hard shadow borders which improve depth perception [S2, unofficial]. Decouples shadowing from scattering, allowing for efficient approximation of global illumination effects like scattering [S1, unofficial].
*   **Limitations:** Approximates scattering using a cone model rather than full spherical integration, which may reduce accuracy for highly anisotropic scattering [S2, unofficial]. Requires precomputation of an auxiliary volume [S1, unofficial].
*   **Performance:** Not given in the sources.

## Background
Shadow volume propagation was introduced by Ropinski et al. [85] as a technique to simulate global illumination effects like scattering in volume rendering [S1, unofficial]. It is similar to half-angle slicing but differs in its decoupling of chromaticity and luminance [S2, unofficial]. The method is used to enhance perception in volume visualization by providing realistic shadowing and scattering effects [S1, unofficial].

## Sources
[S1] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
[S2] A Comparison of Volumetric Illumination Methods by Considering their ..., https://ep.liu.se/ecp/094/005/ecp13094005.pdf, fetched 2026-10-28 (unofficial)
