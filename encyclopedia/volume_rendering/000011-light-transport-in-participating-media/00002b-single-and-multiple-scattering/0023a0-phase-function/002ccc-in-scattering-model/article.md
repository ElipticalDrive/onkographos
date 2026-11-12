# In-scattering Model

Also called: In-Scattering Integration, Particle In-Scattering

## Overview
In-scattering is the phenomenon where light is scattered from other directions back into the viewing direction as it travels through a participating medium [S1, unofficial][S2, unofficial]. It represents the "light gained" component of radiative transfer, contrasting with out-scattering and absorption which constitute "light lost" [S1, unofficial]. While the [Phase Function](3111-phase-function.md) defines the angular distribution of scattering events, the In-scattering Model quantifies the accumulation of this scattered radiance along a ray, attenuated by the medium's extinction [S4, unofficial]. It is the volumetric analog to the Bidirectional Scattering Distribution Function (BSDF), fundamentally altering the appearance of media such as fog and clouds by redistributing environmental light into the camera's line of sight [S1, unofficial].

## How it works
The in-scattering model solves the Radiative Transfer Equation (RTE) to determine the radiance $L$ arriving at a point $p$ along a ray direction $\omega$. The method treats in-scattering as a source term $S(p)$ that adds radiance to the ray, counteracted by the total extinction coefficient $\sigma_t$ [S4, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   Ray origin $p(0)$ and direction $\omega$.
    *   Medium properties at position $x$: scattering coefficient $\sigma_s(x, \omega)$, extinction coefficient $\sigma_t(x, \omega)$, and phase function $f_p(x, \omega, \omega')$.
    *   Incident radiance field $L_i(x, \omega')$ from all directions $\omega'$.
    *   True emission $q(x, \omega)$ (optional, often zero in non-emissive media).
*   **Outputs**:
    *   In-scattered radiance $L_i(p, \omega)$ at the ray origin (or any point $t$ along the ray).

### Data Structures
The implementation requires access to the medium's spatially varying coefficients and the phase function.
*   **Medium Coefficients**: A function or lookup table returning $\sigma_s(x)$ and $\sigma_t(x)$ for any 3D position $x$.
*   **Phase Function**: A function $f_p(x, \omega, \omega')$ satisfying the normalization condition:
    $$ \forall \omega \in \Omega: \; \int_{S^2} f_p (x, \omega, \omega^\prime) \; d\omega^\prime = 1 $$
    [S2, unofficial]
*   **Incident Radiance**: In offline rendering, this is often the result of a previous pass or a precomputed environment map. In real-time, it may be approximated or ignored (single scattering approximation).

### Mathematical Formulation
The differential change in radiance along a ray parameterized by distance $t$ is governed by the first-order linear differential equation [S4, unofficial]:
$$ \frac{dL(p)}{dt} = -\sigma_t(p) L(p) + S(p) $$
where $S(p)$ is the in-scattering source term [S4, unofficial]. The source term $S(p)$ (also denoted as $j(x, \omega)$ in some literature) represents the number of photons scattered into the path per unit length [S3, unofficial]:
$$ S(p) = j(p, \omega) = \sigma_s(p, \omega) \int_{S^2} f_p(p, \omega, \omega') L_i(p, \omega') \, d\omega' + q(p, \omega) $$
[S3, unofficial][S2, unofficial]

To solve for the radiance at the ray origin $p(0)$, the equation is integrated from $t=0$ to $\infty$. The solution assumes that radiance at infinity is zero, $L(p(\infty)) = 0$ [S4, unofficial]. The general solution for radiance at any point $t$ along the ray is [S4, unofficial]:
$$ L(p(t)) = \int_t^{\infty} e^{- \int_{t}^{t'} \sigma_t(p) \, dt''} S(p(t')) \, dt' $$
Specifically, the in-scattered radiance at the start of the ray ($t=0$) is:
$$ L(p(0)) = \int_0^{\infty} T_r(p(t) \rightarrow p(0)) S(p(t)) \, dt $$
where $T_r(p(t) \rightarrow p(0)) = e^{- \int_{0}^{t} \sigma_t(p) \, dt'}$ is the transmittance from point $t$ to the origin [S4, unofficial].

### Algorithm Steps
1.  **Initialize**: Set total in-scattered radiance $L_{in} = 0$.
2.  **Discretize Ray**: Divide the ray segment of interest into $N$ steps with lengths $\Delta t_i$.
3.  **Iterate**: For each step $i$ from $0$ to $N-1$:
    a. Calculate position $p_i = p(0) + t_i \omega$.
    b. Evaluate medium coefficients $\sigma_s(p_i)$ and $\sigma_t(p_i)$.
    c. Evaluate the incident radiance $L_i(p_i, \omega')$ for all relevant incoming directions $\omega'$.
    d. Compute the source term $S(p_i)$:
       $$ S(p_i) = \sigma_s(p_i) \int_{S^2} f_p(p_i, \omega, \omega') L_i(p_i, \omega') \, d\omega' + q(p_i) $$
       [S3, unofficial]
    e. Calculate the transmittance from $p_i$ to the ray origin:
       $$ T_i = \exp\left( -\sum_{k=0}^{i-1} \sigma_t(p_k) \Delta t_k \right) $$
       [S4, unofficial]
    f. Accumulate the contribution:
       $$ L_{in} \leftarrow L_{in} + T_i \cdot S(p_i) \cdot \Delta t_i $$
4.  **Output**: Return $L_{in}$.

### Parameters and Typical Values
*   **Scattering Coefficient $\sigma_s$**: Varies by medium. For fog, typically $0.01 - 0.1 \, \text{m}^{-1}$.
*   **Extinction Coefficient $\sigma_t$**: $\sigma_t = \sigma_a + \sigma_s$. In non-absorbing media, $\sigma_t = \sigma_s$.
*   **Phase Function Anisotropy $g$**: For Henyey-Greenstein, $g \in [-1, 1]$. $g=0$ is isotropic, $g>0$ is forward-scattering.
*   **Integration Steps $N$**: Depends on optical depth. High optical depth requires more steps to avoid aliasing in the transmittance integral.

### Edge Cases and Pitfalls
*   **Singularity at $t=0$**: The transmittance $T_r(p(0) \rightarrow p(0))$ is 1. Numerical integration must handle the start point carefully to avoid double-counting or division by zero if adaptive stepping is used.
*   **High Optical Depth**: If $\sigma_t$ is large, transmittance decays rapidly. Contributions from distant points become negligible. Truncating the integral at a small transmittance threshold (e.g., $10^{-6}$) is a valid optimization [G].
*   **Phase Function Normalization**: The phase function must be normalized to 1 over the sphere. Failure to normalize leads to incorrect energy conservation [S2, unofficial].
*   **Incident Radiance Availability**: The source term $S(p)$ requires $L_i(p, \omega')$. In single-scattering approximations, $L_i$ is often approximated by direct illumination only. For multiple scattering, $L_i$ must include light scattered from other paths, requiring iterative solutions or precomputation [S1, unofficial].

## Subtopics

### [Out-Scattering Model](6803-out-scattering-model.md)
Out-scattering is the phenomenon where photons are scattered away from their incident direction, causing a reduction in radiance along a ray as light travels through a participating medium. It represents the loss of light from the current viewing direction due to collisions with particles that deflect energy into other directions. Unlike in-scattering, which adds radiance, out-scattering acts as an attenuation term in the radiative transfer equation, often grouped with true absorption under the total extinction coefficient.

### [In scattering](13241-in-scattering.md)
In scattering accounts for the increase in radiance along a ray due to light scattering from other directions into the ray's current direction. It serves as the volumetric analog to the Bidirectional Scattering Distribution Function (BSDF), describing how radiance from the surrounding environment is redistributed into the viewing direction. Unlike out-scattering, which removes radiance from the ray, in-scattering adds radiance, fundamentally altering the appearance of participating media such as clouds and fog.

## Operating characteristics
*   **Time Complexity**: $O(N \cdot M)$, where $N$ is the number of ray steps and $M$ is the number of directions sampled for the phase function integral. For single-scattering approximations, $M$ is small (often 1 for direct light). For full multiple scattering, $M$ approaches infinity, requiring Monte Carlo integration or precomputed radiance transfer.
*   **Memory Complexity**: $O(N)$ for storing ray samples and medium coefficients. If precomputing incident radiance fields, memory scales with the resolution of the field.
*   **Measured Performance**: Real-time implementations often use single-scattering approximations with a few ray steps, achieving high frame rates but with visual artifacts in dense media. Offline renderers using multiple scattering can take hours per frame for complex scenes.
*   **Strengths**: Physically accurate representation of light transport in participating media. Captures the "glow" and color bleeding effects characteristic of fog, smoke, and subsurface scattering.
*   **Limitations**: Computationally expensive for multiple scattering. Requires accurate knowledge of the incident radiance field, which is difficult to obtain in dynamic scenes. Single-scattering approximations fail in optically thick media.
*   **When to Use**: Essential for realistic rendering of fog, clouds, smoke, and subsurface scattering effects. Avoid in real-time applications with strict performance constraints unless simplified approximations are acceptable.

## Background
The in-scattering model is derived from the Radiative Transfer Equation (RTE), a fundamental equation in atmospheric physics and astrophysics [S4, unofficial]. The term "in-scattering" distinguishes the gain of radiance from "out-scattering" and absorption, which cause loss [S1, unofficial]. The phase function, central to the model, was historically developed in atmospheric science to describe light scattering by molecules (Rayleigh) and larger particles (Mie) [S3, unofficial]. In computer graphics, the Henyey-Greenstein phase function became popular for its computational efficiency in modeling anisotropic scattering [S1, unofficial]. The integration of in-scattering into volume rendering pipelines has evolved from simple single-scattering approximations in real-time graphics to complex multiple-scattering solvers in offline rendering, often leveraging techniques like precomputed radiance transfer or photon mapping to handle the computational cost [S4, unofficial].

## Sources
[S1] Volumetric rendering fundamental concepts and terminology, https://computergraphics.stackexchange.com/questions/145/volumetric-rendering-fundamental-concepts-and-terminology, fetched 2026-10-27 (unofficial)
[S2] [Graphics] Volume Rendering - GitHub Pages, https://leeyngdo.github.io/blog/computer-graphics/2024-03-28-volume-rendering/, fetched 2026-10-27 (unofficial)
[S3] Volume Rendering Volume Rendering, https://docslib.org/doc/1400821/volume-rendering, fetched 2026-10-28 (unofficial)
[S4] Volume Rendering in Offline Renderer · A Graphics Guy's Note, https://agraphicsguynotes.com/posts/volume_rendering_in_offline_renderer/, fetched 2026-10-28 (unofficial)
