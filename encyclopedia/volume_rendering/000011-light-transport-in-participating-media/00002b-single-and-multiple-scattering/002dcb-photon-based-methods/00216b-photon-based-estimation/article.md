# Photon-based Estimation

Also called: Photon-based Radiance Estimation

## Overview
Photon-based Estimation methods solve the Radiative Transfer Equation (RTE) by decoupling light transport simulation from the final image formation, estimating radiance at sample points using stored photon data rather than tracing paths to the camera [S1]. Unlike direct path tracing which accumulates radiance along camera rays, these methods first trace photons through the participating medium to build a spatial representation of light energy, then estimate the in-scattered radiance at rendering time by querying this representation [S1]. This family is distinguished from [Photon Beam Methods](4976-photon-beam-methods.md) by its use of point-based or volume-based density estimation rather than directional beam structures, and from [Compute Shader Photon Integration](2111-compute-shader-photon-integration.md) and [Volumetric Photon Mapping](2190-volumetric-photon-mapping.md) by the specific storage and querying mechanisms employed (e.g., BVHs, 3D textures, or point clouds) to handle the spatial distribution of photons.

## Shared principles
All methods in this family share the fundamental two-pass structure of photon-based rendering:
1.  **Photon Tracing (Preprocessing):** Photons are emitted from light sources and transported through the participating medium using Monte Carlo random walks. The transport accounts for absorption, scattering, and null collisions. The state of each photon (position, direction, power/flux) is recorded at interaction points.
2.  **Radiance Estimation (Rendering):** During the rendering pass, radiance is estimated at sample points (either along camera rays or at specific volume locations) by gathering contributions from nearby stored photons. This involves density estimation, where the flux from neighboring photons is weighted by a kernel function (e.g., based on distance) to approximate the local light field.

The core mathematical principle is the estimation of the scattered radiance $L_s(x', \omega_0)$ as a sum of contributions from stored photon power, often modeled as:
$$ L_{s}(x',\omega _{0}) \approx \sum_{i} \frac{P_i}{V_i} K(x' - x_i) $$
where $P_i$ is the power of photon $i$, $V_i$ is its volume of influence, and $K$ is a smoothing kernel [S1]. The specific implementation varies in how $V_i$ and $K$ are defined and how the photon data is stored for efficient querying.

## Subtopics

### [Compute Shader Photon Integration](2111-compute-shader-photon-integration.md)
A GPU-accelerated Monte Carlo technique that simulates photon transport using parallel compute shaders, explicitly using global invocation IDs for parallel processing and incremental radiance updates within the shader [2111-compute-shader-photon-integration.md]. It differs from other methods by maintaining photon state in buffers and updating radiance incrementally per step, rather than storing photons for later gathering. Choose when real-time, frame-by-frame photon accumulation is required without a separate preprocessing pass.

### [Volumetric Photon Mapping](2190-volumetric-photon-mapping.md)
A global illumination technique for participating media that separates direct and indirect illumination, storing photons in a 3D volume texture instead of a KD-tree to accelerate screen-space density estimation [2190-volumetric-photon-mapping.md]. It targets real-time performance in noisy contexts like medical imaging. Choose when real-time rendering of participating media is needed and KD-tree overhead is prohibitive.

### [Point-Based Light Transport in Participating Media](3763-point-based-light-transport-in-participating-media.md)
A method for homogeneous translucent materials with refractive boundaries that decorrelates scene complexity by replacing the scene with a shaded point cloud (volume samples) for indirect illumination [3763-point-based-light-transport-in-participating-media.md]. It handles refraction at boundaries which breaks screen-space coherence. Choose when rendering materials enclosed in refractive interfaces where screen-space methods fail.

#### Photon Beam Storage in BVH
Photon beams are stored in a Bounding Volume Hierarchy (BVH) constructed using the surface area heuristic [S1]. To improve partitioning, beams are split into sub-beams during construction, with the splitting length set automatically to produce sub-beams with approximately unit aspect ratio [S1]. At run-time, the camera ray is intersected with the BVH to quickly obtain all intersected beams for radiance estimation [S1].

## Background
The theoretical foundation for photon-based radiance estimation using photon points and beams was established by Amarau and Ohno (2002) and Apodaca and Gritz (1999) [S1]. The use of BVHs for storing photon beams was explored as an acceleration scheme, with MacDonald and Booth's surface area heuristic being adopted for construction [S1]. These methods have been validated on standard test scenes like the Cornell box and bumpy sphere [S1].

## Sources
[S1] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
