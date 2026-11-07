# Anisotropic Scattering via Importance Sampling

Also called: Importance Sampling for Anisotropic Scattering, Phase Function Importance Sampling

## Overview
A technique to sample scattering directions according to a volume's phase function to avoid artifacts associated with uniform sampling [S1, unofficial]. It sets itself apart from [Uniform Sphere Sampling](1669-uniform-sphere-sampling.md) by replacing the uniform distribution with one weighted by the phase function, thereby reducing variance in anisotropic media [S1, unofficial].

## How it works
**Inputs:** Incident direction $\omega_i$, phase function $p(\omega_i, \omega_o)$, asymmetry parameter $g$.
**Outputs:** Outgoing direction $\omega_o$.

**Data Structures:**
The phase function is typically defined by the Henyey-Greenstein model. The asymmetry parameter $g$ controls the scattering direction: $g=0$ is isotropic, $g>0$ is forward-peaked, and $g<0$ is backward-peaked [S1, unofficial].

**Steps:**
1.  Generate a random number $u_1 \in [0, 1)$ [S1, unofficial].
2.  Compute the cosine of the scattering angle $\cos \theta$ using the inverse cumulative distribution function (CDF) of the Henyey-Greenstein phase function:
    $$ \cos \theta = \frac{1}{2g} \left( 1 + g^2 - \left( \frac{1 - g^2}{1 - g + 2g u_1} \right)^2 \right) $$
    If $g=0$, set $\cos \theta = 1 - 2u_1$ [S1, unofficial].
3.  Generate a random azimuthal angle $\phi = 2\pi u_2$ where $u_2 \in [0, 1)$ [S1, unofficial].
4.  Construct the local coordinate system where the z-axis aligns with $\omega_i$.
5.  Compute the new direction $\omega_o$ in the local frame:
    $$ \omega_o = (\sin \theta \cos \phi, \sin \theta \sin \phi, \cos \theta) $$
6.  Transform $\omega_o$ back to the world coordinate system using the basis defined by $\omega_i$ [S1, unofficial].

**Parameters:**
*   $g$: Henyey-Greenstein asymmetry parameter. Typical values tested include $g=0.8$ (forward scattering), $g=0$ (isotropic), and $g=1$ (fully forward) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Uniform Sampling Artifacts:** Initially sampling over a uniform sphere and weighting the resulting directions by the phase function creates visible artifacts [S1, unofficial]. Importance sampling the direction directly based on the phase function is required to avoid these [S1, unofficial].
*   **Boundary Conditions:** The formula for $\cos \theta$ assumes $g \neq 0$. A separate branch is needed for isotropic scattering [S1, unofficial].

## Subtopics

### Importance Sampling Strategy
In the context of inverse rendering caches, secondary samples are split evenly between diffuse color (due to $f_{\text{diffuse}}$) and specular color (due to $f_{\text{specular}}$) [S2]. For diffuse color, multiple importance sampling is used with half the samples from an occlusion-aware importance sampler and half from cosine-weighted hemisphere sampling [S2]. For specular color, importance sampling is performed according to the distribution function $D$ (specifically the Trowbridge-Reitz NDF) [S2].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Eliminates artifacts caused by uniform sampling in anisotropic media [S1, unofficial].
*   **Limitations:** Requires the ability to invert the phase function's CDF, which may not be analytically available for all phase functions [G].

**When to Use:**
Use when rendering volumes with significant anisotropy ($g \neq 0$) where uniform sampling leads to high variance or artifacts [S1, unofficial].

## Background
This method was developed for the CS348b Final Project on rendering underwater scenes, where the pbrt language was extended to handle anisotropic scattering for ray marches [S1, unofficial]. The implementation initially attempted uniform sampling with phase function weighting but switched to direct importance sampling after observing artifacts [S1, unofficial].

## Sources
[S1] CS348b Final Project - Rendering Underwater Scenes, http://www.graphics.stanford.edu/courses/cs348b-competition/cs348b-04/underwater/index.html, fetched 2026-10-30 (unofficial)
[S2] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
