# Phase Function

Also called: Angular Distribution of Scattering, Scattering Phase Function

## Overview
A phase function describes the angular distribution of light scattering at any point within a participating medium, defining the fraction of incoming light traveling along direction $\omega'$ that is scattered into direction $-\omega$ (towards the eye) [S1, S2, unofficial]. It solves the problem of simplifying complex anisotropic scattering interactions into a normalized probability density function that can be evaluated or sampled during light transport simulation [S2, unofficial]. Unlike [Isotropic Phase Function](5147-isotropic-phase-function.md), which scatters light equally in all directions, phase functions like [Henyey-Greenstein](1010-henyey-greenstein-phase-function.md) or [Rayleigh](3721-rayleigh-phase-function.md) model directional biases (anisotropy) [S1, S8, unofficial]. It is distinct from [In-scattering Model](5367-in-scattering-model.md), which calculates the integral accumulation of scattered radiance along a ray, whereas the phase function defines only the local angular redistribution at a single scattering event [S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Position $x$ in the medium, incoming direction $\omega'$ (from light to point), outgoing direction $\omega$ (from point to eye).
*   **Output:** A scalar value $f_p(x, \omega, \omega')$ representing the phase function value, or a sampled direction $\omega_{new}$ if sampling.

**Data Structures**
The phase function is typically implemented as an interface or class. In pbrt, the `PhaseFunction` abstract base class defines the required interface [S5]:
*   `p(wo, wi)`: Evaluates the phase function value for given directions.
*   `Sample_p(wo, wi, u)`: Samples a new outgoing direction `wi` given an incoming `wo` and random sample `u`.

**Notation and Conventions**
*   $\omega$: Vector pointing from $x$ to the eye (outgoing direction).
*   $\omega'$: Vector pointing from $x$ to the light (incoming direction).
*   $\theta$: Scattering angle, the angle between $\omega'$ and $\omega$.
*   $\cos \theta = \omega \cdot \omega'$ [S3, unofficial].
*   **Direction Convention Pitfall:** Literature conventions often define $\omega$ towards the eye and $\omega'$ towards the light. However, some rendering engines (like pbrt) use a convention where both incident ($\omega_i$) and outgoing ($\omega_o$) directions point *away* from the scattering point [S5]. Code must ensure vectors are oriented correctly relative to the specific phase function's expected convention before computing $\theta$ [S1, unofficial].

**Core Properties**
1.  **Reciprocity:** The phase function is symmetric with respect to its direction arguments:
    $$ f_p(x, \omega, \omega') = f_p(x, \omega', \omega) $$
    This allows it to be written as $f_p(x, \theta)$ [S1, S3, unofficial].
2.  **Normalization (Energy Conservation):** The integral over the sphere of directions $\mathbb{S}^2$ must equal 1, ensuring no radiance is added or subtracted by the scattering event itself:
    $$ \int_{\mathbb{S}^2} f_p(x, \omega, \omega') d\omega' = 1 $$
    Note: Some sources define the normalization factor as $1/4\pi$ for isotropic cases, implying the integral of the *probability density* is 1, while the phase function value itself integrates to $4\pi$ if not normalized as a PDF. However, standard rendering literature (and the provided notes) define the phase function $p$ such that $\int_{\mathbb{S}^2} p d\omega = 1$ is NOT the case for the raw phase function in the RTE, but rather the *scattering coefficient* $\sigma_s$ scales it. Let's look at [S3, unofficial]: $\int_{S^2} p(x, \omega_i, \omega) d\omega = 1/4\pi$? No, [S3, unofficial] says: $\int_{S^2} p(x, \omega_i, \omega) d\omega = 1$ is NOT explicitly stated as 1, but rather $p(cos\theta) = 1/4\pi$ for isotropic. If $p=1/4\pi$, then $\int_{S^2} (1/4\pi) d\omega = 1$. So the integral is 1. [S1, unofficial] explicitly states: $\int_{\mathbb{S}^2} f_p(x, \omega, \omega') d\theta = 1$. We will use the normalization $\int_{\mathbb{S}^2} f_p d\omega' = 1$ [S1, S3, unofficial].

**Parameters**
*   **Asymmetry Factor ($g$):** A single quantity describing the shape of the phase function, defined as the average cosine of the scattering angle $\theta$, assuming particles have random, uncorrelated orientations [S6, unofficial]:
    $$ g = \int_{\Omega_{4\pi}} f_p(\theta) \cos \theta \, d\boldsymbol{\omega}' $$
    The value of $g$ is in the range $[-1, 1]$. $g > 0$ favors forward scattering; $g < 0$ favors backscattering; $g = 0$ is isotropic [S6, unofficial].

**Pitfalls**
*   **Vector Orientation:** As noted, the direction of $\omega$ and $\omega'$ must be consistent with the phase function's definition. Using camera-to-point vectors instead of point-to-camera vectors will invert $\theta$ and yield incorrect results [S1, unofficial].
*   **Normalization:** Ensuring the phase function is properly normalized is critical for energy conservation. If the integral is not 1, the medium will artificially brighten or darken the scene [S1, unofficial].

## Subtopics

### [Henyey-Greenstein Phase Function](1010-henyey-greenstein-phase-function.md)
The Henyey-Greenstein (HG) phase function is a parametric, anisotropic model used to describe the angular distribution of light scattering in participating media. It solves the problem of efficiently approximating the complex angular scattering profiles of Mie scattering (e.g., by dust, water droplets, or biological tissue) using a single asymmetry parameter $g$. Unlike the [Isotropic Phase Function](5147-isotropic-phase-function.md), which scatters light equally in all directions, HG biases scattering forward ($g > 0$) or backward ($g < 0$). It is distinguished from the [Rayleigh Phase Function](3721-rayleigh-phase-function.md) by its ability to model larger particles where scattering is strongly directional rather than symmetric. It is widely used in real-time rendering and production due to its computational simplicity and ease of sampling.

### [Rayleigh Phase Function](3721-rayleigh-phase-function.md)
The Rayleigh phase function models the angular distribution of light scattered by particles much smaller than the wavelength of light, such as air molecules in the atmosphere. It produces a characteristic two-lobe distribution that favors forward and backward scattering while exhibiting zero intensity at 90-degree angles. Unlike the isotropic phase function, it introduces anisotropy, and unlike the Henyey-Greenstein function, it is derived from physical dipole radiation principles rather than empirical fitting.

### [Isotropic Phase Function](5147-isotropic-phase-function.md)
The Isotropic Phase Function models light scattering in a participating medium where all directions in the sphere of directions are equally likely to be chosen for scattering. It solves the problem of defining a normalized angular distribution for symmetric media, serving as the baseline for comparison against anisotropic models like the [Henyey-Greenstein Phase Function](1010-henyey-greenstein-phase-function.md) or [Rayleigh Phase Function](3721-rayleigh-phase-function.md). Unlike anisotropic functions that depend on the scattering angle $\theta$, the isotropic function is constant and independent of both the incoming direction $\omega'$ and outgoing direction $-\omega$.

### [In-scattering Model](5367-in-scattering-model.md)
In-scattering is the phenomenon where light is scattered from other directions into the viewing direction as it travels through a participating medium, representing "light gained". It solves the problem of calculating the radiance contribution from the environment and light sources that is redirected by the medium's particles toward the camera, contrasting with out-scattering (light lost). Unlike its siblings which define the angular distribution of scattering (the Phase Function), the In-scattering Model defines the integral accumulation of this scattered light along a ray, incorporating attenuation.

### Lambertian Phase Function (Geometric Optics)
A simpler phase function derived by geometric optics for a spherical particle much larger than the light wavelength, scattering diffusely by Lambert's law [S4, unofficial].
Formula:
$$ p(\boldsymbol{\omega}, \boldsymbol{\omega}') = \frac{8}{3\pi} (\sin \boldsymbol{\alpha} + (\pi - \boldsymbol{\alpha}) \cos \boldsymbol{\alpha}) $$
where $\alpha$ is the angle between directions [S4, unofficial].

### Phase Function Interface
The abstract base class defining the interface for phase functions in pbrt. It requires implementations for evaluating the phase function value and sampling a new direction [S5].
Layout:
Class `PhaseFunction` with two pure virtual methods:
`virtual Float p(const Vector3f &wo, const Vector3f &wi) const = 0;`
`virtual Float Sample_p(const Vector3f &wo, Vector3f *wi, const Point2f &u) const = 0;`
Pitfall: pbrt uses the convention that both the incident direction ($\omega_i$) and the outgoing direction ($\omega_o$) point away from the point where scattering occurs. This differs from scattering literature where the incident direction generally points toward the scattering point [S5].

### Scattering Asymmetry Factor
A single quantity describing the shape of the phase function, defined as the average cosine on the scattering angle $\theta$, assuming particles have random, uncorrelated orientations [S6, unofficial].
Formula:
$$ g = \int_{\Omega_{4\pi}} f_p(\theta) \cos \theta \, d\boldsymbol{\omega}' $$
where $\Omega_{4\pi}$ is the whole sphere of directions and $f_p(\theta)$ is the phase function [S6, unofficial].
Parameter: The value of $g$ is in the range $[-1, 1]$. $g > 0$ favors forward scattering; negative values favor backscattering [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** Evaluation of the phase function value is typically $O(1)$, depending on the complexity of the specific function (e.g., HG is $O(1)$, Rayleigh is $O(1)$). Sampling may require inversion of cumulative distribution functions, which is also generally $O(1)$ or $O(\log N)$ for tabulated functions.
*   **Memory Complexity:** $O(1)$ for parametric functions (HG, Rayleigh). Tabulated phase functions may require $O(N)$ memory for lookup tables.
*   **Strengths:** Provides a physically plausible approximation of light scattering directionality. Essential for realistic rendering of fog, smoke, clouds, and biological tissue.
*   **Limitations:** Simple phase functions (HG, Rayleigh) may not accurately model complex scattering behaviors of highly structured media (e.g., fur, hair, or complex particulate matter) without additional parameters or multiple lobes.
*   **When to use:** Use when anisotropic scattering effects are visible and important for the visual quality of the scene. Use isotropic for simple, uniform fog or when performance is critical and anisotropy is negligible.
*   **When not to use:** Avoid for media where scattering is dominated by surface-like reflections (use BSDFs) or where multiple scattering effects require more sophisticated solvers than simple phase function evaluation.

## Background
The concept of phase functions originates from radiative transfer theory in physics and atmospheric science [S1, S3, unofficial]. The term "phase function" is standard in the literature of participating media [S1, S2, unofficial]. The Henyey-Greenstein phase function was introduced empirically to model scattering in interstellar clouds and later applied to biological tissue and other media [S3, unofficial]. The Rayleigh phase function is derived from classical electromagnetic theory for scattering by particles much smaller than the wavelength of light [S3, unofficial]. In computer graphics, phase functions are a fundamental component of the Volume Rendering Equation, enabling the simulation of light transport in participating media [S1, S3, unofficial].

## Sources
[S1] From the Radiative Transfer Equation to the Volume Rendering Equation, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-summary-equations.html, fetched 2026-10-25 (unofficial)
[S2] Volumetric rendering fundamental concepts and terminology, https://computergraphics.stackexchange.com/questions/145/volumetric-rendering-fundamental-concepts-and-terminology, fetched 2026-10-27 (unofficial)
[S3] Volume Rendering Volume Rendering, https://docslib.org/doc/1400821/volume-rendering, fetched 2026-10-28 (unofficial)
[S4] [PDF] Optical Models for Direct Volume Rendering - Duke Computer Science, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S5] 11.2 Phase Functions - pbr-book.org, https://pbr-book.org/3ed-2018/Volume_Scattering/Phase_Functions, fetched 2026-10-27
[S6] PDF Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
