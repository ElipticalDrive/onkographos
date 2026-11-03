# PBR Shader – Cook-Torrance & Kulla-Conty

A GLSL implementation of Physically Based Rendering (PBR), developed as part of my undergraduate project.

The shader use the **Cook-Torrance microfacet BRDF** with **Kulla-Conty multiple-scattering compensation** to improve energy conservation, particularly for rough surfaces.

## Features

- **Cook-Torrance BRDF**
  - GGX (Trowbridge-Reitz) Normal Distribution Function
  - Smith-Schlick-GGX Geometry Function
  - Fresnel-Schlick Approximation
- **Kulla-Conty Multiple Scattering** – Compensates for energy lost in the single-scattering microfacet model using precomputed lookup tables.
- **PBR Texture** – Supports Albedo, Normal, Metallic, Roughness and Ambient Occlusion maps.

## Precomputed LUTs

The Kulla-Conty approximation uses two lookup tables generated through numerical integration:

- **E LUT** – Directional reflectance as a function of viewing angle and surface roughness.
- **Eavg LUT** – Average reflectance as a function of surface roughness.

These LUTs are sampled in the fragment shader to approximate multiple-scattering contributions.

## Documentation

For a more detailed explanation of the Cook-Torrance BRDF, Kulla-Conty approximation, mathematical formulas and LUT precomputation, see **[Note.pdf(IT)](Note.pdf)**.
