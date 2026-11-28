# Hybrid Surface-Volume and Mesh Representation

Also called: Hybrid Neural Representations, Implicit-Explicit NeRF Hybrids

## Overview
This family represents scenes by combining explicit geometric primitives (surfels, meshes) or implicit surface fields with neural radiance fields, aiming to decouple geometry precision from appearance modeling [7358-neural-surfel-radiance-fields.md][9732-unified-neural-implicit-surface-and-radiance-field-rendering.md][12715-structured-latent-codes-anchored-to-mesh.md]. Unlike standard NeRF which relies solely on volumetric density for both structure and color, these methods use explicit surfaces or occupancy boundaries to guide sampling or define the radiance evaluation points, enabling faster rendering and higher geometric fidelity [7358-neural-surfel-radiance-fields.md][9732-unified-neural-implicit-surface-and-radiance-field-rendering.md].

## Shared principles
All methods in this family replace or augment the standard volume density $\sigma$ with a mechanism that explicitly defines or approximates a surface boundary. This allows the rendering equation to be evaluated either directly on the surface or within a narrow band around it, reducing the number of samples required for convergence compared to full-volume sampling [9732-unified-neural-implicit-surface-and-radiance-field-rendering.md].

The core mechanism involves:
1.  **Surface Definition:** Defining the scene geometry via either a set of explicit primitives (surfels with position, normal, radius) [7358-neural-surfel-radiance-fields.md], a continuous occupancy field $o_\theta$ where the zero-level set defines the surface [9732-unified-neural-implicit-surface-and-radiance-field-rendering.md], or latent codes anchored to mesh vertices [12715-structured-latent-codes-anchored-to-mesh.md].
2.  **Hybrid Rendering:** Computing color $\hat{C}(\mathbf{r})$ by evaluating neural appearance networks at specific geometric locations (e.g., ray-surfel intersections [7358-neural-surfel-radiance-fields.md], surface roots [9732-unified-neural-implicit-surface-and-radiance-field-rendering.md]) rather than uniformly along the ray.
3.  **Differentiability:** Maintaining end-to-end differentiability through the geometric construction and rendering steps to optimize scene parameters from images [7358-neural-surfel-radiance-fields.md][9732-unified-neural-implicit-surface-and-radiance-field-rendering.md].

## Subtopics

### [Neural Surfel Radiance Fields](7358-neural-surfel-radiance-fields.md)
Represents the scene as a dynamic collection of neural surfels (position, normal, radius, feature vector). It uses differentiable rasterization to render only at ray-surfel intersections, enabling online, real-time reconstruction. Choose this for large-scale, dynamic indoor scenes requiring fast online integration.

### [Unified Neural Implicit Surface and Radiance Field Rendering](9732-unified-neural-implicit-surface-and-radiance-field-rendering.md)
Replaces density with a continuous occupancy field $o_\theta$. It finds surface intersections via root-finding and samples densely only near the surface, using a decaying interval schedule. Choose this for high-precision static reconstruction where mesh extraction is required.

### [Structured Latent Codes Anchored to Mesh](12715-structured-latent-codes-anchored-to-mesh.md)
Stores geometry and appearance latent codes at anchor nodes (mesh vertices) on a deformable mesh. Choose this for scenarios requiring local editing and deformation tracking of explicit mesh structures.

## Background
The motivation for hybrid representations arises from the limitations of pure volumetric NeRFs, which suffer from slow convergence due to empty space sampling and ambiguous geometry [7358-neural-surfel-radiance-fields.md][9732-unified-neural-implicit-surface-and-radiance-field-rendering.md]. SurfelNeRF [7358-neural-surfel-radiance-fields.md] was introduced to enable online photorealistic reconstruction by leveraging explicit surfels and differentiable rasterization. UNISURF [9732-unified-neural-implicit-surface-and-radiance-field-rendering.md] was developed to unify implicit surface rendering with volume rendering capabilities, allowing for accurate surface reconstruction without per-pixel masks. These methods build on [Neural Radiance Fields](4972-neural-radiance-fields.md) by introducing geometric priors to guide the radiance field optimization.

## Sources
[S1] Neural Surfel Radiance Fields: Online Photorealistic 3D Reconstruction, https://arxiv.org/abs/2103.14629, 2021-03-25
[S2] UNISURF: A Unified Mesh-Based Neural Implicit Surface and Radiance Field, https://arxiv.org/abs/2104.00631, 2021-04-01
