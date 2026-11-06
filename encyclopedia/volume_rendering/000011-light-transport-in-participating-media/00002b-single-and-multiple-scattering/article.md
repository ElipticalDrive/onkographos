# Single and multiple scattering

Also called: Light transport in participating media, Volumetric scattering

## Overview
Single and multiple scattering methods simulate the interaction of light with participating media by modeling how photons collide with particles, change direction, and are absorbed or emitted. This family solves the problem of computing the radiance distribution within a volume, ranging from the simplified case where light scatters only once (single scattering) to the physically accurate but computationally expensive case of infinite scattering bounces (multiple scattering) [S1, book]. What sets this family apart from its siblings is the explicit handling of the scattering phase function and the recursive nature of in-scattering; while [Optical models](33-optical-models.md) defines the static properties, and [Local illumination, shadows and ambient occlusion](34-local-illumination-shadows-and-ambient-occlusion.md) approximates lighting, this family provides the rigorous transport mechanisms (estimators, path tracing, and integral solvers) required to compute the equilibrium radiance field.

## Shared principles
All methods in this family are derived from the conservation of energy for a differential cylinder of light passing through a medium [S1, book]. They share the following core definitions and mechanisms:

**1. Fundamental Interactions**
Any volumetric effect is composed of three fundamental characteristics [S1, book]:
*   **Absorption:** The loss of radiant energy along a ray due to energy conversion (e.g., to heat). It is described by the absorption coefficient $\sigma_a$ (units: $m^{-1}$) [S1, book].
*   **Emission:** The addition of radiant energy where the medium is luminous (e.g., flames). It is described by the emissive term $L_e(\mathbf{p}, \boldsymbol{\omega})$, representing radiance emitted per unit length [S1, book].
*   **Scattering:** The redirection of light rays upon collision with particles. It is split into:
    *   *In-scattering:* Light from other directions reflected into the view ray.
    *   *Out-scattering:* Light from the view ray reflected away.
    In most lighting calculations, out-scattering is folded into the **extinction** term because its net effect on the view ray is identical to absorption [S1, book].

**2. Notation**
*   $\mathbf{p}$: Position of interaction.
*   $\boldsymbol{\omega}$: Ray direction (unit vector).
*   $L$: Radiance quantity.
*   $L_i$: Incoming radiance.
*   $L_o$: Outgoing radiance.
*   $\sigma_t$: Total extinction coefficient, $\sigma_t = \sigma_a + \sigma_s$ [S1, book].

**3. Differential Forms**
The change in radiance $dL$ over a differential path length $dt$ is governed by the balance of loss and gain terms [S1, book]:
*   **Absorption Model:** $dL = -\sigma_a L_i dt$ [S1, book].
*   **Emission Model:** $dL = \sigma_e dt$ (where $\sigma_e$ is the emission coefficient) [S1, book].
*   **General RTE:** Combines extinction (loss) and source terms (gain from emission and in-scattering) [S1, book].

**4. Transmittance**
The fraction of light surviving transmission through a medium is defined by the Beer-Lambert law [S1, book]:
$$ T = e^{-\tau} $$
where $\tau$ is the optical depth (integrated extinction) [S1, book].

**5. Phase Functions**
Scattering directionality is modeled by a phase function $p(\omega_i, \omega_o)$, defined in terms of $a = \cos(\theta)$, where $\theta$ is the angle between the incident direction $\omega_i$ and the scattered direction $\omega_o$ [S1, book]. The choice of phase function depends on the ratio of particle radius $r$ to wavelength $\lambda$ [S1, book]:
*   **Rayleigh Scattering:** Used when $r \ll \lambda$ (e.g., smoke, dust, water molecules).
    $$ P_R(a) = \frac{3}{4}(1+a)^2 $$
*   **Mie Scattering:** Used when $r \approx \lambda$ (e.g., fog, water droplets). Common variants include:
    *   Haziness: $P_{HM}(a) = 1 + 9 \left( \frac{1+a}{2} \right)^8$
    *   Murkiness: $P_{MM}(a) = 1 + 50 \left( \frac{1+a}{2} \right)^{32}$
*   **Henyey-Greenstein (HG):** An approximation to Mie scattering, widely used in graphics for its efficiency and asymmetry parameter $g$:
    $$ P_{HG}(a) = \frac{1 - g^2}{(1 + g^2 - 2ga)^{\frac{3}{2}}} $$
*   **Schlick's Approximation:** A computationally cheaper approximation to HG that avoids fractional exponentiation:
    $$ P_S(a) = \frac{1-g^2}{(1-ga)^2} $$

## Subtopics

### [Volumetric Radiance Estimators](818-volumetric-radiance-estimators.md)
Estimates in-scattered radiance by aggregating discrete photon samples using spatial blurring kernels. It differs from path integral methods by relying on pre-computed photon maps and density estimation rather than stochastic ray marching. Choose when simulating complex lighting with pre-computed global illumination data.

### [Path Integral Framework](873-path-integral-framework.md)
Formulates light transport as a Monte Carlo integration over the space of all possible paths. It differs from standard path tracing by allowing analytical approximations like the Most Probable Path (MPP) and WKB expansions. Choose when requiring a unified theoretical basis for combining photon density estimation and path integration.

### [Homogeneous Medium Sampling](980-homogeneous-medium-sampling.md)
Determines the distance to the first interaction in a medium with uniform optical properties using Beer's law. It differs from Delta Tracking by using the exact extinction coefficient, making it optimal for constant-density scenarios. Choose for simple, uniform volumes like fog or haze.

### [Delta tracking](985-delta-tracking.md)
Samples free path lengths in heterogeneous media by converting the medium into a homogeneous one via "fake" particles (null-scattering). It differs from Homogeneous Medium Sampling by handling spatially varying density without subdivision. Choose for complex, heterogeneous volumes where exact sampling is difficult.

### [Multiple Importance Sampling for Volumetric Path Tracing](1167-multiple-importance-sampling-for-volumetric-path-tracing.md)
Reduces variance by combining unidirectional phase function sampling and direct light source sampling using the power heuristic. It differs from single-strategy path tracing by dynamically balancing PDFs. Choose when paths skip index-matching surfaces or in heterogeneous media where one sampling strategy is inefficient.

### [Biased Sampling for Path Depth](1338-biased-sampling-for-path-depth.md)
Reduces variance in dense, highly anisotropic media by exponentially decaying the extinction coefficient and anisotropy parameter as ray depth increases. It differs from unbiased methods by explicitly altering physical properties to produce shorter paths. Choose for clouds and dense media where convergence speed is prioritized over strict physical accuracy.

### [Unified Points, Beams, and Paths (UPBP)](1402-unified-points-beams-and-paths-upbp.md)
Fuses volumetric radiance estimators (Point-Point, Point-Beam, Beam-Beam) with unbiased bidirectional path tracing (BPT). It differs from siblings by subsuming photon mapping and BPT into a single framework. Choose for high-quality offline rendering requiring both biased density estimation and unbiased path connections.

### [Volumetric Path Tracing](1863-volumetric-path-tracing.md)
A basic path tracer using simple sampling (e.g., delta tracking) without explicit light sampling. It differs from advanced methods by lacking variance reduction techniques. Choose for debugging and computing ground-truth results.

### [Sample Matching for Extinction Gradients](1884-sample-matching-for-extinction-gradients.md)
Differentiates volumetric transport by evaluating scattering and transmittance terms at shared locations to reduce variance in gradient estimation. It differs from standard estimators by aligning sampling locations. Choose when optimizing medium parameters or performing differentiable rendering.

### [Joint Path Vertex Sampling](2136-joint-path-vertex-sampling.md)
Connects light and eye subpath endpoints by constructing a connection subpath with two new vertices using three random decisions. It differs from Equiangular Sampling by sampling geometry proportional to the path contribution function. Choose for high-variance single and multiple scattering scenarios in bidirectional path tracing.

### [Unidirectional Importance Sampling for Double-Scattering](2151-unidirectional-importance-sampling-for-double-scattering.md)
Analytically importance samples geometry terms along 3-segment paths (double-scattering) in isotropic media. It differs from standard techniques by canceling out singularities in the path contribution function. Choose for reducing variance in double-scattering events.

### [Walter's Method](2262-walter-s-method.md)
Computes refracted light paths within participating media bounded by triangle meshes to produce light shafts. It differs from photon-based methods by solving for valid refracted paths without intermediate lighting representations. Choose for complex refractive volumetric boundaries like glass volumes.

### [Phase Function](3111-phase-function.md)
A function describing the angular distribution of light scattering at any point within a medium. It defines the fraction of incoming light scattered into the viewing direction. Choose when defining the scattering behavior of a participating medium.

### [Radiative Transfer and Integral Equation Solvers](3853-radiative-transfer-and-integral-equation-solvers.md)
Solves the Radiative Transfer Equation (RTE) using integral equation methods. It differs from Monte Carlo methods by providing deterministic solutions. Choose for specific analytical or semi-analytical volume rendering tasks.

### [Volumetric-Primitives Path Tracer](4107-volumetric-primitives-path-tracer.md)
A forward rendering method modeling the scene as 3D kernel-based volumetric primitives. It differs from voxel-based methods by using kernel primitives. Choose for efficient multiple scattering via Next-Event Estimation in primitive-based scenes.

### [Microflake Model](4739-microflake-model.md)
A physically-based model where the medium is a cloud of microflakes with statistically described orientations. It differs from isotropic models by providing direction-dependent phase functions and anisotropic attenuation. Choose for materials like velvet or fur simulated as volumes.

### [Heterogeneous In-scattering with Multiple Scattering](4934-heterogeneous-in-scattering-with-multiple-scattering.md)
Computes radiance from direct and indirect (recursive) lighting in heterogeneous volumes. It differs from single-scattering approximations by handling multiple bounces. Choose for physically accurate multiple scattering in complex media.

### [Single Scattering Approximations](5340-single-scattering-approximations.md)
Approximates volumetric lighting by considering only the first scattering event. It differs from multiple scattering methods by ignoring indirect light bounces. Choose for real-time applications or simple atmospheric effects where performance is critical.

### [Photon-based Methods](5620-photon-based-methods.md)
Uses photon mapping or beam tracing to estimate volumetric lighting. It differs from path tracing by storing light transport data for later density estimation. Choose for complex global illumination in participating media.

## Background
The theoretical foundation of single and multiple scattering in participating media is rooted in the Radiative Transfer Equation (RTE), which describes the transport of energy through a medium [S1, book]. The selection of appropriate phase functions (Rayleigh, Mie, Henyey-Greenstein) is based on physical optics principles relating particle size to wavelength, as detailed in classical texts like Bohren and Huffman [S1, book]. In computer graphics, these principles were adapted for volume rendering to simulate atmospheric effects, smoke, and fire. Early implementations often relied on single-scattering approximations for performance, but modern techniques utilize path tracing, delta tracking, and photon mapping to achieve physically accurate multiple scattering [S1, book]. The field has evolved from simple ray marching to sophisticated unbiased and biased estimators that balance visual fidelity with computational cost.

## Sources
[S1] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
