# Single and multiple scattering

Also called: Light transport in participating media, Volumetric scattering

## Overview
Single and multiple scattering methods simulate the interaction of light with participating media (such as fog, smoke, or clouds) by modeling how photons are absorbed, emitted, and redirected by particles within the volume [G]. While single scattering approximations assume light undergoes at most one collision before reaching the camera, multiple scattering methods account for complex, multi-bounce light transport, enabling the rendering of dense, glowing, or highly reflective volumetric effects [G]. This family is distinguished from [Optical models](33-optical-models.md) by its focus on the stochastic or deterministic solution of the Radiative Transfer Equation (RTE) for light paths, and from [Local illumination, shadows and ambient occlusion](34-local-illumination-shadows-and-ambient-occlusion.md) by its explicit handling of in-scattering from distant light sources rather than just local shading [G].

## Shared principles
All methods in this family are derived from the conservation of energy for a differential cylinder of light passing through a medium [G]. They share the following core definitions and mechanisms:

**1. Fundamental Interactions**
Any volumetric effect is composed of three fundamental characteristics [G]:
*   **Absorption:** The loss of radiant energy along a ray due to energy conversion (e.g., to heat). It is described by the absorption coefficient $\sigma_a$ (units: $m^{-1}$) [G].
*   **Emission:** The addition of radiant energy where the medium is luminous (e.g., flames). It is described by the emissive term $L_e(\mathbf{p}, \boldsymbol{\omega})$, representing radiance emitted per unit length [G].
*   **Scattering:** The redirection of light rays upon collision with particles. It is split into:
    *   *In-scattering:* Light from other directions reflected into the view ray [G].
    *   *Out-scattering:* Light from the view ray reflected away [G].
In most lighting calculations, out-scattering is folded into the **extinction** term because its net effect on the view ray is identical to absorption [G].

**2. Notation**
*   $\mathbf{p}$: Position of interaction.
*   $\boldsymbol{\omega}$: Ray direction (unit vector).
*   $L$: Radiance quantity.
*   $L_i$: Incoming radiance.
*   $L_o$: Outgoing radiance.
*   $\sigma_t$: Total extinction coefficient, $\sigma_t = \sigma_a + \sigma_s$ [G].

**3. Differential Forms**
The change in radiance $dL$ over a differential path length $dt$ is governed by the balance of loss and gain terms [G]:
*   **Absorption Model:** $dL = -\sigma_a L_i dt$ [G].
*   **Emission Model:** $dL = \sigma_e dt$ (where $\sigma_e$ is the emission coefficient) [G].
*   **General RTE:** Combines extinction (loss) and source terms (gain from emission and in-scattering) [G].

**4. Transmittance**
The fraction of light surviving transmission through a medium is defined by the Beer-Lambert law [G]:
$$ T = e^{-\tau} $$
where $\tau$ is the optical depth (integrated extinction) [G].

## Subtopics

### [Volumetric Radiance Estimators](818-volumetric-radiance-estimators.md)
Volumetric Radiance Estimators are a family of nine distinct methods introduced by Jarosz et al. [2011a] that estimate equilibrium radiance in participating media by reconstructing values from discrete samples (photon points or photon beams) using spatial blurring. They differ from unbiased path integral estimators by lacking a general path contribution function and pdf, instead relying on specific radiometric quantities and kernel dimensions to approximate the radiance field.

### [Path Integral Framework](873-path-integral-framework.md)
The Path Integral Framework formulates light transport simulation as a Monte Carlo integration problem over the space of all possible light transport paths, expressing pixel intensity as a single non-recursive integral. It solves the problem of evaluating the recursive rendering equation by expanding it into a sum over path configurations, serving as the theoretical foundation for combining photon density estimation and Monte Carlo path integration.

### [Homogeneous Medium Sampling](980-homogeneous-medium-sampling.md)
Homogeneous Medium Sampling determines the distance a ray travels through a participating medium with uniform optical properties before interacting, avoiding explicit distance and spherical integrations by letting the ray randomly bounce within the medium before potentially hitting a surface. It differs from [Delta tracking](985-delta-tracking.md) by directly sampling the interaction distance from the extinction coefficient rather than using a majorant bound, and it serves as the foundational step for [Free-path sampling](1844-free-path-sampling.md).

### [Delta tracking](985-delta-tracking.md)
Delta tracking is a Monte Carlo method for sampling the distance to the next interaction (scattering or absorption) in a heterogeneous participating medium where the extinction coefficient varies spatially. It solves the problem that standard exponential distance sampling requires a constant extinction coefficient, which is rarely true in real-world volumes. Unlike ray marching, which introduces systematic bias by using fixed step sizes, delta tracking is unbiased and handles inhomogeneous media without requiring a structured grid.

### [Single-scattering homogeneous volume rendering](1135-single-scattering-homogeneous-volume-rendering.md)
This method renders a participating medium by solving the radiative transfer equation under the assumption that light scatters exactly once within a homogeneous volume before reaching the camera. It extends a basic absorption-only model by adding in-scattering from surface-emitting geometry, assuming the surfaces do not reflect or transmit light. It is distinguished from [Single-scattering atmosphere rendering](3289-single-scattering-atmosphere-rendering.md) by its general applicability to arbitrary scene manifolds rather than specific atmospheric layers, and from [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md) by its deterministic ray-marching approach.

### [Multiple Importance Sampling for Volumetric Path Tracing](1167-multiple-importance-sampling-for-volumetric-path-tracing.md)
Multiple Importance Sampling (MIS) for volumetric path tracing reduces variance in estimating direct lighting by combining contributions from phase function sampling and next event estimation (NEE). It solves the problem of high variance when sampling light sources in participating media, particularly when paths skip index-matching surfaces or involve delta distributions. Unlike standard path tracing which may rely on a single sampling strategy, this method uses the power heuristic to balance the probability density functions (PDFs) of both strategies, ensuring robust convergence.

### [Biased Sampling for Path Depth](1338-biased-sampling-for-path-depth.md)
Biased Sampling for Path Depth is a variance reduction technique for volumetric path tracing in dense, highly anisotropic participating media (such as clouds) that exponentially decays the medium's extinction coefficient and anisotropy parameter as the ray depth increases. By reducing the frequency of scattering events and the directionality of scattering at deeper bounces, the method significantly shortens path lengths and accelerates convergence, trading a slight, visually negligible bias for massive performance gains.

### [Unified Points, Beams, and Paths (UPBP)](1402-unified-points-beams-and-paths-upbp.md)
Unified Points, Beams, and Paths (UPBP) is a combined rendering algorithm that fuses volumetric density estimation estimators (Point-Point, Point-Beam, Beam-Beam) with unbiased Bidirectional Path Tracing (BPT) using Multiple Importance Sampling (MIS). It solves the problem of efficiently simulating light transport in diverse participating media by automatically selecting the most suitable representation—points for high-order scattering, beams for low-order scattering, and paths for specular surface interactions—thereby mitigating the weaknesses of individual estimators.

### [Sample Matching for Extinction Gradients](1884-sample-matching-for-extinction-gradients.md)
Sample Matching for Extinction Gradients is a differentiable volumetric path tracing method that reduces the variance of extinction coefficient gradients by evaluating the positive scattering term and the negative transmittance term at shared spatial locations along each path segment. Unlike the base [Volumetric Path Tracing](1863-volumetric-path-tracing.md) method, which estimates these two opposing terms at unrelated points, this technique exploits their negative correlation to cancel variance, while maintaining the same primal rendering output.

### [Zonal Method](1891-zonal-method.md)
The Zonal Method is a deterministic technique for computing multiple scattering in participating media by assuming isotropic scattering, where the source term depends only on position. It extends the diffuse radiosity method for interreflecting surfaces to volumes, solving for the isotropic scattering source term by setting up and solving a system of simultaneous linear equations. Unlike stochastic methods such as [Delta tracking](985-delta-tracking.md) or [Volumetric Radiance Estimators](818-volumetric-radiance-estimators.md), it provides a global solution for multiple bounces but is restricted to isotropic phase functions and suffers from high computational complexity for dense volumes.

### [Joint Path Vertex Sampling](2136-joint-path-vertex-sampling.md)
Joint Path Vertex Sampling (JPVS) is a method for constructing a full light transport path by connecting a light subpath endpoint $\mathbf{a}$ and an eye subpath endpoint $\mathbf{d}$ via a connection subpath containing two new vertices $\mathbf{b}$ and $\mathbf{c}$. It solves the high variance associated with connecting subpaths in participating media by using three random decisions (two distances and one direction) to sample the connection, rather than the zero, one, or two decisions used by methods like Equiangular Sampling or Virtual Ray Lights. This approach reduces variance by explicitly sampling the connection direction to cancel geometric singularities.

### [Unidirectional Importance Sampling for Double-Scattering](2151-unidirectional-importance-sampling-for-double-scattering.md)
A fully analytic method to importance sample the geometry terms along 3-segment paths (double-scattering) in isotropic participating media, generalizing equiangular sampling for 2-segment paths. It constructs a joint probability density function (PDF) for the intermediate scattering vertices that cancels out the singularities inherent in the path contribution function, thereby reducing variance compared to standard uniform or single-scattering sampling strategies.

### [Walter's Method](2262-walter-s-method.md)
Walter's Method is a technique for computing refraction paths within triangle meshes to produce light shafts within complex refractive volumetric boundaries. It sets itself apart from siblings like [Volumetric Radiance Estimators](818-volumetric-radiance-estimators.md) and [Photon-based Methods](5620-photon-based-methods.md) by directly solving for valid refracted light paths without an intermediate lighting representation, thereby avoiding the blurring and approximation artifacts inherent in photon mapping. Unlike [Light Beam Tracing](link-to-light-beam-tracing), the solution is exact and does not suffer from blurring, allowing it to resolve fine details that standard photon mapping cannot.

### [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md)
This method computes the radiance contribution of a single scattering event in a participating medium by integrating the in-scattered light from a point light source along a view ray using Monte Carlo integration. It solves the radiative transfer equation for the single-scattering term by sampling interaction points along the ray and evaluating the light transport from the source to those points, attenuated by the medium's extinction. Unlike deterministic uniform ray marching methods, it provides an unbiased estimator for point-light in-scattering in homogeneous volumes.

### [Phase Function](3111-phase-function.md)
A phase function describes the angular distribution of light scattering at any point within a participating medium, defining the fraction of incoming light traveling along direction $\omega'$ that is scattered into direction $-\omega$ (towards the eye). It solves the problem of simplifying complex anisotropic scattering interactions into a normalized probability density function that can be evaluated or sampled during light transport simulation. Unlike [Isotropic Phase Function](5147-isotropic-phase-function.md), which scatters light equally in all directions, phase functions model directional preferences such as forward scattering in clouds or back-scattering in smoke.

### [Radiative Transfer and Integral Equation Solvers](3853-radiative-transfer-and-integral-equation-solvers.md)
This family encompasses deterministic methods for solving the Radiative Transfer Equation (RTE) in participating media, primarily by discretizing the angular or spatial domains or by formulating the transport problem as a system of linear or integral equations. Unlike the stochastic path-tracing siblings in [Single and multiple scattering](35-single-and-multiple-scattering.md), these methods aim to compute the global illumination solution (often steady-state) by propagating flux or solving for intensity distributions directly, thereby avoiding the noise inherent in Monte Carlo estimators.

### [Single Scattering Approximations](5340-single-scattering-approximations.md)
Single scattering approximations solve the Radiative Transfer Equation (RTE) by assuming that light undergoes exactly one scattering event within the participating medium before reaching the camera. This simplification ignores multiple scattering events and self-illumination (glow), making it applicable when the volume has low albedo or low density. It is distinguished from multiple scattering methods by its computational efficiency and from homogeneous-only models by its ability to handle heterogeneous media in some variants.

### [Photon-based Methods](5620-photon-based-methods.md)
Photon-based methods solve the problem of estimating radiance in participating media by decoupling light transport into a preprocessing phase (photon tracing) and a rendering phase (radiance estimation via density estimation or virtual light sources). Unlike direct integration methods that trace paths per pixel, these methods store photon interactions in spatial structures and estimate incident radiance at query points using kernel density estimation or explicit summation. What sets this family apart from its siblings is the specific use of stored photon data (points, beams, or spheres) to approximate the in-scattering term.

### [Art-Directable Multiple Volumetric Scattering](7442-art-directable-multiple-volumetric-scattering.md)
Art-Directable Multiple Volumetric Scattering is a path tracing method that computes efficient, physically accurate multi-bounce diffusion in participating media while allowing independent artistic control over image contrast. It distinguishes itself from standard volumetric path tracers by decoupling the physical diffusion (scale) from the visual contrast, enabling biased but convergent rendering of high-order scattering without pre-computation or caching.

### [Optimized Monte Carlo Integrator](9013-optimized-monte-carlo-integrator.md)
The Optimized Monte Carlo Integrator is a method for computing radiance scattered inside a participating medium from a collimated light source towards the camera, specifically designed for real-time rendering applications. It solves the problem of redundant geometric traversal by caching segment data, allowing samples to be drawn proportional to transmittance without repeated box outline calculations. It sets itself apart from standard ray marching by pre-computing analytic integration values for trapezoid-shaped segments of the medium, significantly reducing per-sample computational cost.

### [Continuous Media Convergence](9653-continuous-media-convergence.md)
Continuous Media Convergence describes the phenomenon where a discrete medium composed of a large number of scattering particles renders with an appearance statistically indistinguishable from a homogeneous continuous medium. It solves the problem of efficiently rendering dense particulate scenes by allowing the use of traditional volumetric path tracing with global, averaged radiative transfer parameters instead of explicit particle geometry. It differs from [Delta tracking](985-delta-tracking.md) by replacing spatial queries with uniform or analytically derived coefficients, and from discrete particle rendering by avoiding explicit geometry traversal.

### Composite Medium Definition
A method to define participating media by summing multiple phase terms, each with its own density, phase function, and absorption, to produce a composite medium [S3, unofficial]. The total optical depth is the sum of the optical depths of each term in RGB. Phase functions are applied per-channel, and the Henyey-Greenstein eccentricity $g$ ranges from $(-1, 1)$, where $g < 0$ is back-scattering, $g = 0$ is isotropic, and $g > 0$ is forward scattering [S3, unofficial].

### Skylight Approximation
Extending the shading method to multiple light sources to approximate global illumination from skylight (sunlight scattered by atmosphere) [S4, unofficial]. The technique shades clouds from multiple light sources, stores resulting particle colors from all shading iterations, and at render time evaluates the phase function at each particle once per light. An ambient term is added to compensate for lost scattered light due to the scattering approximation [S4, unofficial].

### Poisson Scattering Probability and Phase Function Convolution
Computes the phase function for $i$ scattering events by convolving the base phase function $p(\omega, \omega')$ with itself $i-1$ times via integration over the unit sphere [S5, unofficial]. For the Henyey-Greenstein phase function, the result of $i$ convolutions is the same formula with $g$ replaced by $g^i$, precomputed and saved for each pixel on the viewing cube faces [S5, unofficial]. The probability $q_{ij}$ of exactly $i$ scattering events along segment $S_j$ is given by $q_{ij} = \exp(-D_j) D_j^i / i!$, where $D_j$ is the optical depth [S5, unofficial]. The contribution $E_j$ is computed as $\sum_i q_{ij} K_{ij}$, where $K_{ij}$ is the convolution sample [S5, unofficial]. This Poisson distribution is an approximation valid for small water droplets where scattering is highly peaked forward [S5, unofficial].

## Background
The mathematical foundation of light transport in participating media is the Radiative Transfer Equation (RTE), which describes the conservation of energy for a differential cylinder of light [G]. Early work in computer graphics focused on single scattering approximations due to computational constraints, often assuming homogeneous media and simple phase functions like Rayleigh or Mie scattering [S2, book]. Rayleigh scattering is appropriate when particle radius $r \ll \lambda$ (e.g., smoke, dust), while Mie scattering is used when $r \approx \lambda$ (e.g., fog, water droplets) [S2, book]. Schlick's approximation to the Henyey-Greenstein phase function became standard for its efficiency in avoiding fractional exponentiation [S2, book].

The development of multiple scattering methods was driven by the need to render dense, glowing volumes where light bounces multiple times [G]. Deterministic solvers, such as the Zonal Method, offer alternative approaches for specific isotropic cases, though they suffer from high complexity [S1][S2, book]. Modern approaches often combine density estimation techniques with unbiased path tracing, leveraging variance reduction strategies like Multiple Importance Sampling (MIS) and biased sampling for path depth to handle complex media efficiently [S1][S2, book].

## Sources
[S1] A lighting model for general participating media, https://doi.org/10.1145/1053427.1053446, fetched 2026-11-02
[S2] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
[S3] Volumetric Light Scattering in Fallout 4, https://www.scribd.com/document/748135006/Fast-Flexible-Physically-Based-Volumetric-Light-Scattering, fetched 2026-10-29 (unofficial)
[S4] Real-Time Cloud Rendering, https://www.cs.unc.edu/techreports/01-005.pdf, fetched 2026-10-28 (unofficial)
[S5] Diffusion and multiple anisotropic scattering for global illumination in clouds, https://digital.library.unt.edu/ark:/67531/metadc1410406/?q=multiple scattering rendering, fetched 2026-10-29 (unofficial)
