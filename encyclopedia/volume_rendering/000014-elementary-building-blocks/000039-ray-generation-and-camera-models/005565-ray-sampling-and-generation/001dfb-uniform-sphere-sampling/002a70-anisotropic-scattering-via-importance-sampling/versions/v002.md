# Anisotropic Scattering via Importance Sampling

Also called: Importance Sampling for Anisotropic Scattering, Phase Function Importance Sampling

## Overview
A technique to sample scattering directions according to a volume's phase function to avoid artifacts associated with uniform sampling [S1, unofficial]. It sets itself apart from [Uniform Sphere Sampling](1669-uniform-sphere-sampling.md) by replacing the uniform distribution with one weighted by the phase function, thereby reducing variance in anisotropic media [S1, unofficial].

## How it works
**Inputs and outputs:** Not given in the sources.

**Data structures:** The phase function is typically defined by the Henyey-Greenstein model [G]. The asymmetry parameter $g$ controls scattering directionality: $g=0$ is isotropic, $g>0$ is forward-peaked, and values like $g=0.8$ and $g=1$ are tested [S1, unofficial].

**Numbered steps:** 
1. Evaluate the phase function $p(\omega_i, \omega_o)$ for the incident direction $\omega_i$ and candidate outgoing directions [G].
2. Sample the outgoing direction $\omega_o$ directly from the phase function's distribution instead of using uniform sphere sampling [S1, unofficial].
3. Pass the sampled direction to the ray marcher for integration [S1, unofficial].

**Formulas:** Not given in the sources.

**Parameters:** $g$: Henyey-Greenstein asymmetry parameter. Tested values include $g=0.8$, $g=0$ (isotropic), and $g=1$ [S1, unofficial].

**Edge cases and pitfalls:** Uniform sphere sampling with phase function weighting creates visible artifacts [S1, unofficial]. Direct importance sampling based on the phase function is required to avoid these [S1, unofficial].

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
