# Particle Tracing for Global Illumination
Also called: Photon Beam Tracing phase, Photon Mapping (Photon Beam Tracing phase)

## Overview
Particle Tracing for Global Illumination is a global illumination technique that tracks paths of scattered light energy through space starting at the light sources [S2, unofficial]. It is related to instant radiosity and photon mapping, which approximate irradiance by splatting energy around a scene [S1, unofficial]. Unlike unidirectional path tracing which traces from the camera, this method builds a density estimate of light transport by emitting particles from lights, making it particularly effective for caustics and complex indirect illumination in participating media.

## How it works
**Inputs and Outputs**
*   **Inputs:** Light source positions, intensities, and spectral properties; scene geometry; participating media properties (extinction $\sigma_t$, scattering $\sigma_s$, phase function $p$); spatial data structure for density estimation (e.g., k-d tree or voxel grid).
*   **Outputs:** A stored set of particles (or beams) representing the light field, which is later used to estimate radiance at surface points or volume locations during the rendering (backprojection) phase.

**Data Structures**
Each particle $P$ stores:
*   Position $\mathbf{p}$: 3D coordinates $(x, y, z)$.
*   Direction $\boldsymbol{\omega}$: Unit vector of propagation.
*   Power $w$: Scalar weight (radiant flux) carried by the particle.
*   Color $C$: RGB or spectral vector representing the energy distribution.

**Algorithm Steps**
1.  **Initialization:** Create an empty spatial data structure for storing particles.
2.  **Emission:** For each light source, emit $N$ particles. The initial power $w$ is distributed among them based on the light's total flux. The initial position $\mathbf{p}$ is sampled from the light's emission distribution. The initial direction $\boldsymbol{\omega}$ is sampled according to the light's angular distribution (e.g., cosine lobe for area lights).
3.  **Propagation:** While the particle's power $w$ is above a threshold $\epsilon$:
    a.  **Distance to Next Interaction:** Sample the distance $t$ to the next scattering or absorption event. For a homogeneous medium, $t = -\ln(u) / \sigma_t$, where $u \in [0,1]$ is a uniform random number [G]. For inhomogeneous media, use delta tracking or step-size adaptation based on local $\sigma_t(\mathbf{p})$.
    b.  **Movement:** Update position: $\mathbf{p} \leftarrow \mathbf{p} + t \cdot \boldsymbol{\omega}$.
    c.  **Boundary Check:** If $\mathbf{p}$ exits the scene or medium, terminate the particle.
    d.  **Interaction:** Determine if the event is scattering or absorption.
        *   If absorption (probability $\sigma_a / \sigma_t$): Terminate the particle.
        *   If scattering (probability $\sigma_s / \sigma_t$):
            i.  **Store Particle:** Insert $\mathbf{p}, \boldsymbol{\omega}, w, C$ into the spatial data structure.
            ii. **Scatter:** Sample new direction $\boldsymbol{\omega}'$ from the phase function $p(\boldsymbol{\omega} \to \boldsymbol{\omega}')$.
            iii. **Update Power:** Reduce power by the albedo factor: $w \leftarrow w \cdot \frac{\sigma_s}{\sigma_t}$. Update color $C$ if wavelength-dependent scattering is modeled.
            iv. **Russian Roulette:** If $w$ is very low, terminate with probability $1 - w/w_{max}$ to save computation, or scale $w$ and continue [G].
4.  **Termination:** Repeat until all light sources have emitted their allocated particles.

**Parameters**
*   $N$: Number of particles per light. Typical values range from $10^5$ to $10^7$ depending on desired noise levels.
*   $\epsilon$: Termination threshold. Typically $10^{-6}$ to $10^{-4}$ of initial power.
*   $\sigma_t, \sigma_s$: Extinction and scattering coefficients of the medium.

**Edge Cases and Pitfalls**
*   **Density Estimation Bias:** The choice of kernel radius for splatting affects bias-variance trade-off. Too small leads to noise; too large leads to blurring.
*   **Boundary Conditions:** Particles hitting boundaries must be handled correctly (reflection, refraction, or termination).
*   **Memory Usage:** Storing all particles can be memory-intensive for large scenes.

## Subtopics

### Photon Mapping
Stores particles on surfaces for density estimation. Chosen for surface-dominated scenes with caustics. [Link](5620-photon-based-methods.md)

### Photon Beam Tracing
Propagates beams instead of points, reducing variance for directional lights. Chosen for efficient handling of parallel light sources. [Link](1402-unified-points-beams-and-paths-upbp.md)

### Instant Radiosity
Approximates light transport using virtual point lights. Chosen for fast, approximate global illumination. [Link](3853-radiative-transfer-and-integral-equation-solvers.md)

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K)$, where $N$ is the number of particles and $K$ is the average number of interactions per particle. Density estimation adds $O(N \log N)$ or $O(N)$ depending on the spatial structure.
*   **Memory Complexity:** $O(N)$ to store all particles.
*   **Strengths:** Efficiently handles caustics and complex indirect illumination; decouples light transport from rendering.
*   **Limitations:** High memory usage; biased result due to density estimation; difficult to handle dynamic scenes.
*   **When to Use:** Static scenes with complex lighting, caustics, or participating media where direct path tracing is inefficient.
*   **When Not to Use:** Dynamic scenes, real-time applications, or scenes with simple lighting.

## Background
Particle tracing for global illumination was introduced as a method to track paths of scattered light energy through space starting at the light sources [S2, unofficial]. It is related to instant radiosity and photon mapping, which approximate irradiance by splatting energy around a scene [S1, unofficial]. The technique has been used in volume rendering and global illumination research to handle complex light transport in participating media.

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
