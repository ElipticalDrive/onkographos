# In-scattering Model

Also called: In-Scattering Integration, Particle In-Scattering

## Overview
In-scattering is the phenomenon where light is scattered from other directions back into the viewing direction as it travels through a participating medium [S1, unofficial][S2, unofficial]. It represents the "light gained" component of radiative transfer, contrasting with out-scattering and absorption which constitute "light lost" [S1, unofficial]. While the [Phase Function](3111-phase-function.md) defines the angular distribution of scattering events, the In-scattering Model quantifies the accumulation of this scattered radiance along a ray, attenuated by the medium's extinction [S4, unofficial].

## How it works
The in-scattering model solves the Radiative Transfer Equation (RTE) to determine the radiance $L$ arriving at a point $p$ along a ray direction $\omega$ [S4, unofficial]. The method treats in-scattering as a source term $S(p)$ that adds radiance to the ray, counteracted by the total extinction coefficient $\sigma_t$ [S4, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   Ray origin $p(0)$ and direction $\omega$ [S4, unofficial].
    *   Medium properties at position $x$: scattering coefficient $\sigma_s(x, \omega)$, extinction coefficient $\sigma_t(x, \omega)$, and phase function $f_p(x, \omega, \omega')$ [S2, unofficial][S3, unofficial].
    *   Incident radiance field $L_i(x, \omega')$ from all directions $\omega'$ [S2, unofficial].
    *   True emission $q(x, \omega)$ (optional, often zero in non-emissive media) [S3, unofficial].
*   **Outputs**:
    *   In-scattered radiance $L(p, \omega)$ at the ray origin or any point $t$ along the ray [S4, unofficial].

### Data Structures
The implementation requires access to the medium's spatially varying coefficients and the phase function [S3, unofficial].
*   **Medium Coefficients**: A function or lookup table returning $\sigma_s(x)$ and $\sigma_t(x)$ for any 3D position $x$ [G].
*   **Phase Function**: A function $f_p(x, \omega, \omega')$ satisfying the normalization condition:
    $$ \forall \omega \in \Omega: \; \int_{S^2} f_p (x, \omega, \omega^\prime) \; d\omega^\prime = 1 $$
    [S2, unofficial]
*   **Incident Radiance**: In offline rendering, this is often the result of a previous pass or a precomputed environment map [G]. In real-time, it may be approximated or ignored (single scattering approximation) [G].

### Mathematical Formulation
The differential change in radiance along a ray parameterized by distance $t$ is governed by the first-order linear differential equation [S4, unofficial]:
$$ \frac{dL(p)}{dt} = -\sigma_t(p) L(p) + S(p) $$
where $S(p)$ is the in-scattering source term [S4, unofficial]. The source term $S(p)$ (also denoted as $j(x, \omega)$ in some literature) represents the number of photons scattered into the path per unit length [S3, unofficial]:
$$ S(p) = j(p, \omega) = \sigma_s(p, \omega) \int_{S^2} f_p(p, \omega, \omega') L_i(p, \omega') \, d\omega' + q(p, \omega) $$
[S3, unofficial][S2, unofficial]

To solve for the radiance at the ray origin $p(0)$, the equation is integrated from $t=0$ to $\infty$ [S4, unofficial]. The solution assumes that radiance at infinity is zero, $L(p(\infty)) = 0$ [S4, unofficial]. The general solution for radiance at any point $t$ along the ray is [S4, unofficial]:
$$ L(p(t)) = \int_t^{\infty} e^{- \int_{t}^{t'} \sigma_t(p) \, dt''} S(p(t')) \, dt' $$
Specifically, the in-scattered radiance at the start of the ray ($t=0$) is:
$$ L(p(0)) = \int_0^{\infty} T_r(p(t) \rightarrow p(0)) S(p(t)) \, dt $$
where $T_r(p(t) \rightarrow p(0)) = e^{- \int_{0}^{t} \sigma_t(p) \, dt'}$ is the transmittance from point $t$ to the origin [S4, unofficial].

### Algorithm Steps
1.  **Initialize**: Set total in-scattered radiance $L_{in} = 0$ and current transmittance $T = 1.0$ [G].
2.  **Discretize Ray**: Divide the ray segment of interest into $N$ steps with uniform length $\Delta t$ [G].
3.  **Iterate**: For each step $i$ from $0$ to $N-1$:
    a. Calculate position $p_i = p(0) + t_i \omega$ where $t_i = i \cdot \Delta t$ [G].
    b. Evaluate medium coefficients $\sigma_s(p_i)$ and $\sigma_t(p_i)$ [G].
    c. Evaluate the incident radiance $L_i(p_i, \omega')$ for all relevant incoming directions $\omega'$ [G].
    d. Compute the source term $S(p_i)$:
       $$ S(p_i) = \sigma_s(p_i) \int_{S^2} f_p(p_i, \omega, \omega') L_i(p_i, \omega') \, d\omega' + q(p_i) $$
       [S3, unofficial]
    e. Accumulate the contribution:
       $$ L_{in} \leftarrow L_{in} + T \cdot S(p_i) \cdot \Delta t $$
       [S4, unofficial]
    f. Update transmittance for the next step:
       $$ T \leftarrow T \cdot e^{-\sigma_t(p_i) \Delta t} $$
       [S4, unofficial]
4.  **Output**: Return $L_{in}$ [G].

### Parameters and Typical Values
*   **Scattering Coefficient $\sigma_s$**: Varies by medium [S3, unofficial]. Not given in the sources.
*   **Extinction Coefficient $\sigma_t$**: $\sigma_t = \sigma_a + \sigma_s$ [S2, unofficial]. In non-absorbing media, $\sigma_t = \sigma_s$ [G].
*   **Phase Function Anisotropy $g$**: For Henyey-Greenstein, $g \in [-1, 1]$ [G]. $g=0$ is isotropic, $g>0$ is forward-scattering [G].
*   **Integration Steps $N$**: Depends on optical depth [G]. High optical depth requires more steps to avoid aliasing in the transmittance integral [G].

### Edge Cases and Pitfalls
*   **Singularity at $t=0$**: The transmittance $T_r(p(0) \rightarrow p(0))$ is exactly $1$ [S4, unofficial]. Numerical integration must handle the start point carefully to avoid double-counting or division by zero if adaptive stepping is used [G].
*   **High Optical Depth**: If $\sigma_t$ is large, transmittance decays rapidly [S4, unofficial]. Contributions from distant points become negligible [G]. Truncating the integral at a small transmittance threshold (e.g., $10^{-6}$) is a valid optimization [G].
*   **Phase Function Normalization**: The phase function must be normalized to $1$ over the sphere [S2, unofficial].
*   **Incident Radiance Availability**: The source term $S(p)$ requires $L_i(p, \omega')$ [S2, unofficial]. In single-scattering approximations, $L_i$ is often approximated by direct illumination only [G].

## Subtopics

### [Out-Scattering Model](6803-out-scattering-model.md)
Out-scattering is the phenomenon where photons are scattered away from their incident direction, causing a reduction in radiance along a ray as light travels through a participating medium. It represents the loss of light from the current viewing direction due to collisions with particles that deflect energy into other directions. Unlike in-scattering, which adds radiance, out-scattering acts as an attenuation term in the radiative transfer equation, often grouped with true absorption under the total extinction coefficient.

### [In scattering](13241-in-scattering.md)
In scattering accounts for the increase in radiance along a ray due to light scattering from other directions into the ray's current direction. It serves as the volumetric analog to the Bidirectional Scattering Distribution Function (BSDF), describing how radiance from the surrounding environment is redistributed into the viewing direction. Unlike out-scattering, which removes radiance from the ray, in-scattering adds radiance.

## Operating characteristics
*   **Time Complexity**: $O(N \cdot M)$, where $N$ is the number of ray steps and $M$ is the number of directions sampled for the phase function integral [G]. For single-scattering approximations, $M$ is small (often $1$ for direct light) [G].
*   **Memory Complexity**: $O(N)$ for storing ray samples and medium coefficients [G]. If precomputing incident radiance fields, memory scales with the resolution of the field [G].
*   **Measured Performance**: Not given in the sources.
*   **Strengths**: Physically accurate representation of light transport in participating media [G]. Captures the "glow" and color bleeding effects characteristic of fog, smoke, and subsurface scattering [G].
*   **Limitations**: Computationally expensive for multiple scattering [G]. Requires accurate knowledge of the incident radiance field, which is difficult to obtain in dynamic scenes [G]. Single-scattering approximations fail in optically thick media [G].
*   **When to Use**: Essential for realistic rendering of fog, clouds, smoke, and subsurface scattering effects [G]. Avoid in real-time applications with strict performance constraints unless simplified approximations are acceptable [G].

## Background
The in-scattering model is derived from the Radiative Transfer Equation (RTE), a fundamental equation in atmospheric physics and astrophysics [S4, unofficial]. The term "in-scattering" distinguishes the gain of radiance from "out-scattering" and absorption, which cause loss [S1, unofficial]. The phase function, central to the model, was historically developed in atmospheric science to describe light scattering by molecules (Rayleigh) and larger particles (Mie) [S3, unofficial]. In computer graphics, the Henyey-Greenstein phase function became popular for its computational efficiency in modeling anisotropic scattering [S1, unofficial].

## Sources
[S1] Volumetric rendering fundamental concepts and terminology, https://computergraphics.stackexchange.com/questions/145/volumetric-rendering-fundamental-concepts-and-terminology, fetched 2026-10-27 (unofficial)
[S2] [Graphics] Volume Rendering - GitHub Pages, https://leeyngdo.github.io/blog/computer-graphics/2024-03-28-volume-rendering/, fetched 2026-10-27 (unofficial)
[S3] Volume Rendering Volume Rendering, https://docslib.org/doc/1400821/volume-rendering, fetched 2026-10-28 (unofficial)
[S4] Volume Rendering in Offline Renderer · A Graphics Guy's Note, https://agraphicsguynotes.com/posts/volume_rendering_in_offline_renderer/, fetched 2026-10-28 (unofficial)
