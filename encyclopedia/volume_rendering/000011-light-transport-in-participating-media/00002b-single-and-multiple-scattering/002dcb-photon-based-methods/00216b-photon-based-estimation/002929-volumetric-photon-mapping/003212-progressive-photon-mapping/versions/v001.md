# Progressive Photon Mapping

Also called: Multipass Photon Mapping

## Overview
Progressive Photon Mapping is a multipass rendering technique that continuously emits photons into the image to gradually reduce illumination inaccuracy [S1]. It builds on [Volumetric Photon Mapping](2190-volumetric-photon-mapping.md) by addressing the memory limitations of storing a single huge volumetric photon map, which is costly for participating media [S2]. Unlike single-pass methods, it iteratively refines the estimate by discarding intermediate data structures after each pass [S2].

## How it works
**Inputs:** Light source parameters, participating media properties (absorption coefficient $\sigma_a$, scattering coefficient $\sigma_s$, phase function $p$), and scene geometry.
**Outputs:** Final radiance image.
**Data Structures:**
- **Photon Map:** A spatial index (e.g., k-d tree) storing photon positions, energies, and surface normals or volume interaction points. In the progressive variant, this is replaced by a temporary, small photon map generated per iteration [S2].
- **Shading Points:** Points on the volume or surface where radiance is estimated.

**Steps:**
1. **Initialize:** Set iteration counter $k = 0$. Initialize the final accumulated radiance buffer to zero [G].
2. **Photon Emission:** For iteration $k$, emit a batch of photons from light sources into the scene [S1].
3. **Photon Tracing:** Trace each photon through the participating media. At each interaction point (absorption or scattering), store the photon's position, energy, and direction in the current small photon map [S2]. If the photon is absorbed, terminate. If scattered, continue based on the phase function [S1, S2].
4. **Radiance Estimation:** For each shading point in the scene, query the current small photon map to estimate radiance. The estimator typically uses a kernel density estimation approach, summing contributions from nearby photons within a radius $r_k$ [S1, S2]. The radius $r_k$ is adjusted per iteration to control bias-variance trade-off [G].
   - Formula for radiance estimate $L$ at point $x$:
     $$ L(x) = \frac{1}{V(r_k)} \sum_{i \in N(x)} \frac{E_i}{\sigma_t} p(\omega_i \to \omega_o) $$
     Where $V(r_k)$ is the volume of the search sphere, $E_i$ is the energy of photon $i$, $\sigma_t$ is the extinction coefficient, $N(x)$ are photons within $r_k$, and $p$ is the phase function [G].
5. **Accumulation:** Add the estimated radiance to the final accumulated radiance buffer [G].
6. **Discard:** Discard the current small photon map to free memory [S2].
7. **Iterate:** Increment $k$. If convergence criteria are not met, return to Step 2. The number of photons or the search radius may be adjusted in subsequent passes to reduce variance [S1].

**Parameters:**
- Number of iterations: Not given in the sources.
- Photons per iteration: Not given in the sources.
- Search radius $r_k$: Decreases over iterations to reduce bias [G].

**Edge Cases:**
- **Boundary Conditions:** Photons escaping the volume must be handled (e.g., terminated or reflected) [G].
- **Precision:** Floating-point precision in energy accumulation may affect convergence [G].

## Subtopics

### Progressive Photon Beam Mapping
A progressive method that eliminates memory limitations by replacing the huge volumetric photon map with many small photon maps generated iteratively [S2]. It uses beam-to-beam estimators, which are suitable for low-order scattering objects like fog, offering faster convergence for such media [S2]. In each iteration, a small photon map is generated, used to update shading point contributions, and then discarded [S2]. This strategy greatly improves practicality by avoiding the storage of a single massive map [S2].

## Operating characteristics
**Time Complexity:** Linear with respect to the number of photons and shading points per iteration, but requires multiple passes [S1, S2].
**Memory Complexity:** Significantly reduced compared to standard Volumetric Photon Mapping because only small photon maps are stored per iteration [S2].
**Performance:** Convergence is gradual, improving accuracy with each pass [S1]. Beam-based variants converge faster for low-order scattering media like fog [S2].
**Strengths:** Memory efficiency; ability to handle large scenes with participating media; gradual convergence allows for early termination if quality is sufficient [S1, S2].
**Limitations:** Requires multiple passes, increasing total render time; bias-variance trade-off management is complex [S1, S2].
**When to use:** When memory is constrained or when rendering participating media with low-order scattering where beam-based estimators are effective [S2].
**When not to use:** When real-time single-pass performance is required and memory is not a constraint [S1].

## Background
Progressive Photon Mapping was developed by Hachisuka et al. [S1]. It was proposed as a multipass rendering technique to continuously emit photons and gradually reduce inaccuracy [S1]. The method builds on the foundational photon mapping technique and addresses its scalability issues in participating media [S1, S2].

## Sources
[S1] Real-time volume rendering for three-dimensional fetal ultrasound using volumetric photon mapping., https://doi.org/10.1186/s42492-024-00177-4, fetched 2026-10-26
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
