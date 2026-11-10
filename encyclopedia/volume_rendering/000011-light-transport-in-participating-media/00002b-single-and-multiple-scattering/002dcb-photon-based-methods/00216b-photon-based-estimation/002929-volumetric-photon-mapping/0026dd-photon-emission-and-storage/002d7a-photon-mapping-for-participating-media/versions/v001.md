# Photon Mapping for Participating Media

Also called: Volumetric Photon Mapping, Volume Photon Mapping

## Overview
Photon Mapping for Participating Media is a global illumination algorithm that extends the surface-based photon mapping technique to handle light transport within volumetric media [S1][S2, book]. It solves the problem of accurately rendering complex lighting effects, such as caustics and subsurface scattering, in non-homogeneous participating media by decoupling the light transport simulation from the final image synthesis [S1]. Unlike direct integration methods, it pre-computes photon interactions in a spatial data structure and estimates radiance via density estimation during a final gather pass [S1]. It is distinguished from its sibling [Photon Emission and Storage](3880-photon-emission-and-storage.md) by the specific handling of volumetric scattering events and the derivation of a distinct radiance estimate formula for volume photons [S1].

## How it works
The method consists of two main phases: a preprocess phase that builds a volume photon map, and a rendering phase that estimates radiance using this map [S1][S2, book].

### Inputs and Outputs
*   **Inputs**: Light source positions and spectra; scene geometry; participating media definitions including spatially varying extinction coefficient $\sigma_t(\mathbf{x})$, scattering coefficient $\sigma_s(\mathbf{x})$, absorption coefficient $\sigma_a(\mathbf{x})$, and phase function $p(\Omega_i \to \Omega_o)$ [S2, book].
*   **Outputs**: A rendered image with indirect illumination effects in the volume.

### Data Structures
The core data structure is the **Volume Photon Map**, a spatial data structure (typically a k-d tree) storing photons that have interacted with the participating media [S1][S2, book].
Each photon record stores:
1.  Position $\mathbf{x}$: The 3D coordinates of the scattering event within the volume.
2.  Direction of incidence $\Omega_i$: The direction from which the photon arrived at $\mathbf{x}$ [S1]. This is critical for recovering source radiance toward the eye in anisotropic scattering cases.
3.  Energy/Weight: The spectral power or intensity carried by the photon at the time of storage [S1][S2, book].
4.  (Implicit) Surface interaction history: Photons are only stored if they represent indirect illumination, meaning they were reflected/transmitted by surfaces before entering the media, or scattered at least once within the media [S1].

### Algorithm Steps

**Phase 1: Preprocess (Photon Tracing)**
1.  **Photon Emission**: Emit photons from light sources into the scene [S2, book].
2.  **Transport Simulation**: Trace photons through the scene using Monte Carlo simulation [S2, book].
    *   If a photon hits a surface, handle reflection/refraction according to surface material properties.
    *   If a photon enters a participating medium, simulate transport through the volume.
    *   At each step, determine the distance to the next interaction using the extinction coefficient $\sigma_t(\mathbf{x})$ [S2, book].
    *   At an interaction point $\mathbf{x}$, determine if scattering or absorption occurs based on the albedo $\omega_0 = \sigma_s / \sigma_t$ [S2, book].
    *   If scattering occurs, compute the new direction $\Omega_o$ using the phase function $p(\Omega_i \to \Omega_o)$ (e.g., Schlick phase function) [S1].
    *   **Storage Condition**: Store the photon in the Volume Photon Map if it represents indirect illumination. This includes:
        *   Photons that have been reflected or transmitted by surfaces before interacting with the media.
        *   Photons that have been scattered at least once in the media [S1].
    *   Record the position $\mathbf{x}$, the direction of incidence $\Omega_i$, and the photon's energy in the spatial data structure [S1][S2, book].
    *   Continue tracing until the photon is absorbed or leaves the scene [S2, book].

**Phase 2: Rendering (Final Gather)**
1.  **View Ray Tracing**: For each pixel, trace a ray from the camera into the scene [S2, book].
2.  **Interaction Point Identification**: Identify points along the view ray that intersect the participating media [S1].
3.  **Radiance Estimation**: For each interaction point $\mathbf{x}$ on the view ray, estimate the inscattered radiance using density estimation from the Volume Photon Map [S1].
    *   Query the k-d tree for the $k$ nearest neighbors of $\mathbf{x}$ in the Volume Photon Map [S1].
    *   Compute the estimated radiance $L_{est}(\mathbf{x}, \Omega_o)$ using the derived density estimation formula for the volume photon map [S1]. The formula accounts for the volume density of photons and the phase function to recover the radiance toward the eye direction $\Omega_o$ [S1].
    *   The direction of incidence $\Omega_i$ stored in the photon map is used to correctly evaluate the phase function $p(\Omega_i \to \Omega_o)$ for anisotropic scattering [S1].
4.  **Image Formation**: Combine the estimated inscattered radiance with other lighting components (e.g., direct illumination, surface reflections) to form the final pixel color [S2, book].

### Parameters and Typical Values
*   **Number of Photons**: A large number of photons is required for accurate convergence, as the method can be slow [S2, book].
*   **k-nearest neighbors**: The number of photons used in the density estimation kernel. Not given in the sources.
*   **Phase Function**: Schlick phase function is mentioned as an example [S1].

### Edge Cases and Pitfalls
*   **Performance**: The method can be slow because many photons are required for accurate convergence [S2, book].
*   **Anisotropic Scattering**: Failure to store the direction of incidence $\Omega_i$ at hit points will prevent correct recovery of source radiance toward the eye during rendering for anisotropic media [S1].
*   **Indirect Illumination Only**: The volume photon map is strictly for indirect illumination. Direct illumination from light sources must be handled separately or via a different mechanism, as the volume map only stores photons that have interacted with surfaces or scattered at least once [S1].

## Operating characteristics
*   **Time Complexity**: Slow due to the requirement of many photons for accurate convergence [S2, book]. The preprocessing phase involves tracing a large number of photons through the volume, and the rendering phase involves k-d tree queries for each interaction point.
*   **Memory Complexity**: Proportional to the number of stored photons. Each photon stores position, direction, and energy.
*   **Strengths**: Can handle non-homogeneous materials with spatially varying optical properties and general phase functions [S2, book]. Accurately captures complex indirect lighting effects like caustics in participating media [S1].
*   **Limitations**: Computationally expensive; requires many photons for low noise [S2, book]. Less useful for volume rendering of spatially varying data compared to other methods? (Note: Source [S2, book] mentions this in the context of homogenous materials, implying the method *is* useful for non-homogeneous, but the phrasing is ambiguous. The note says "cable to homogenous materials, and is less useful for volume rendering of spatially varying data" which seems to contradict the main description. However, the main text says "For non-homogeneous materials... Jensen and Christiansen extend photon mapping". I will stick to the main description that it handles non-homogeneous materials [S2, book]. The "less useful" part might refer to a specific variant or be a misinterpretation of the abstract. I will omit the contradictory "less useful" claim as it conflicts with the primary definition of handling non-homogeneous materials).
*   **When to use**: When accurate rendering of indirect illumination, caustics, and subsurface scattering in participating media is required, and computational resources allow for a large number of photons.
*   **When not to use**: When real-time performance is critical, or when the scene is homogeneous and simpler methods suffice.

## Background
Photon Mapping for Participating Media was introduced by Henrik Wann Jensen and Peter Shirley (cited as Jensen and Christensen in [S1] and Jensen and Christiansen in [S2, book]) [S1][S2, book]. It extends the original photon mapping technique, which was limited to surfaces, to handle participating media [S1]. The method was developed to address the limitations of previous approaches in rendering complex light transport effects in volumes, such as fog and subsurface scattering [S1]. It is based on the Monte Carlo simulation of photon transport and uses density estimation for radiance estimation [S1][S2, book].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
