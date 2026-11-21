# Empirical and Hybrid Models


## Overview
Empirical and Hybrid Models provide non-physical or semi-physical approximations for volumetric light transport, prioritizing visual fidelity and performance over strict adherence to the Radiative Transfer Equation (RTE) [G]. Unlike physically accurate methods that solve for multiple scattering, these models use heuristic shading terms, blurred indirect light, or hybrid surface-volume compositing to simulate translucency and atmospheric effects at interactive rates [4109, 12114]. They set themselves apart from [Absorption-only model](839-absorption-only-model.md) by including scattering-like effects, and from [Single Scattering Models](15290-single-scattering-models.md) by avoiding the computational cost of exact integration, often relying on precomputed gradients or empirical blur operations [G].

## Shared principles

## Subtopics

### [Empirical Translucent Volume Shading](4109-empirical-translucent-volume-shading.md)
An interactive optical model that captures the qualitative appearance of translucency by blending direct attenuation with a blurred indirect light contribution and surface shading. It solves the problem of rendering semi-transparent data with realistic lighting without simulating multiple scattering. Choose this when interactive rates are required for complex optical properties where physical accuracy is secondary to visual appeal.

### [Hybrid Surface-Volume Rendering](12114-hybrid-surface-volume-rendering.md)
A method for visualizing CFD data that combines physically-based atmospheric attenuation for low-albedo gases with a scanline-buffer algorithm for surface-defined objects. It sets itself apart by explicitly integrating surface fragments into the volume traversal loop to determine rendering extent, displaying the entirety of 3D data rather than just isosurfaces. Choose this when rendering mixed scenes with both dense volumetric gas and solid geometric objects.

### Multi-Volume Illumination Rendering
A rendering approach using separate volumes for emission, absorption, and reflection to simulate material light properties, with gradients computed on the fly by the GPU [S1]. It allows for fine-grained control over voxel properties using scaling factors ($k_{em}, k_{ab}, k_{ref}$) and boolean masks [S1]. It is chosen for detailed visualization of specific tissue types or markers in medical/scientific data where distinct light interaction properties need to be highlighted [S1]. The method sets emission and absorption volumes to the same data source, sets the reflection volume to a constant value of 1 for each voxel, and applies scaling factors globally or per-voxel [S1]. Boolean operations with masks select areas of interest for targeted voxel manipulation [S1]. Square root normalization is applied to combined volumes to amplify visibility after inversion, and final images are inverted to enhance print quality and contrast [S1]. Stereo rendering is optionally enabled by offsetting the camera along the x-axis to generate anaglyph images [S1]. The implementation uses a LUT resolution of $64 \times 64 \times 64$, a Henyey–Greenstein phase function with asymmetry parameter $g = 0.8$, a focal length of 3.0 scene units, a distance to object of 6 scene units, and an opacity threshold of 0.95 [S1].

## Background
The concepts stem from the need for real-time or near-real-time visualization of complex volumetric data, such as medical imaging (e.g., zebrafish embryos, brain tumors) and CFD simulations [S1, 12114]. The specific implementations cited originate from research published in *Digital* (2024) regarding GPU-enabled volume rendering [S1]. The hybrid approach addresses limitations of traditional isosurface rendering by capturing the full volume extent [12114]. The empirical model addresses the computational bottleneck of multiple scattering by using heuristic approximations [4109].

## Sources
[S1] GPU-Enabled Volume Renderer for Use with MATLAB, https://doi.org/10.3390/digital4040049, fetched 2026-10-29
