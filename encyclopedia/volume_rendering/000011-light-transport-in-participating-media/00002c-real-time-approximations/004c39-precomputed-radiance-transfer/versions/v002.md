# Precomputed Radiance Transfer


## Overview
Precomputed Radiance Transfer (PRT) approximates light transport integrals by decoupling radiative transfer calculations from the per-frame rendering loop, storing precomputed transfer functions or incident radiance for runtime lookup [G]. Unlike sibling methods that approximate scattering dynamically per frame (e.g., [Voxel Cone Tracing](1454-voxel-cone-tracing.md), [Froxel-based Volumetric Rendering](1919-froxel-based-volumetric-rendering.md)), PRT methods precompute the expensive light transport calculation, though the primary drawback noted in the literature is their high runtime cost [S1, unofficial]. It differs from [Precomputed Lighting (Baking)](2169-precomputed-lighting-baking.md) by storing transfer functions that map incident light to outgoing radiance, enabling dynamic lighting updates at the expense of increased runtime complexity [S1, unofficial].

## Shared principles
All methods in this family share the goal of approximating the integral of the radiative transfer equation for multiple scattering orders without solving it iteratively per frame [G]. They typically rely on one or more of the following mechanisms:
1. **Spatial Discretization:** The volume is discretized into a structured grid (voxels, froxels) or a hierarchical structure (octrees, light volumes) to allow efficient spatial queries and interpolation [G].
2. **Precomputation or Caching:** Lighting information (attenuation, visibility, or flux) is precomputed offline or updated incrementally per frame, stored in textures or data structures, and queried at runtime [G].
3. **Dimensionality Reduction:** Directional information is compressed using basis functions (e.g., Spherical Harmonics) or approximated using geometric primitives (e.g., cones, light volumes with blur widths) to reduce the cost of evaluating the scattering integral [G].
4. **Temporal Stability:** Many methods incorporate temporal accumulation (e.g., TAA reprojection) to stabilize noise and reduce per-frame computational load [G].

## Subtopics

### [Modular Transfer](1249-modular-transfer.md)
Modular Transfer is a precomputation-based rendering technique for volumetric media with repeated structures, designed to approximate high-order multiple scattering efficiently. It solves the problem of rendering complex, optically thick media with anisotropic scattering by splitting light transport paths into components that are precomputed offline and computed at runtime, making path integrals separable via isotropy and diffuser events. Unlike its siblings, it uses flux transfer matrices instead of light fields to handle the curse of dimensionality. Choose it for complex, optically thick media where diffusion approximations fail.

### [Precomputed Lighting (Baking)](2169-precomputed-lighting-baking.md)
Precomputed Lighting (Baking) is a volume rendering optimization technique that accelerates rendering by pre-computing the incident radiance term ($L_i$) for each voxel in a grid and storing it in the data structure, thereby skipping the evaluation of $L_i$ during the per-pixel raymarching pass. It solves the performance bottleneck of evaluating volumetric shadows or direct illumination for every sample point along a camera ray by trading memory and offline computation time for runtime speed. Unlike Modular Transfer, it is static and view-independent, requiring a full re-bake whenever the scene geometry, lighting, or density simulation changes. Choose it for static scenes where direct illumination is the primary bottleneck.

### [Real-time PRT Reconstruction via ZH Rotation](14304-real-time-prt-reconstruction-via-zh-rotation.md)
This method reconstructs lighting in real-time by rotating precomputed PRT values, stored in the Zernike Harmonic (ZH) basis, to match the local surface frame, projecting them back into the Spherical Harmonic (SH) basis, and computing the dot product with the lighting environment projected into SH. It sets itself apart from standard PRT by enabling per-pixel rotation of the transfer function on the GPU, allowing for dynamic surface orientations without the prohibitive cost of per-sample SH rotation. Choose it when dynamic surface normals require real-time PRT updates.

### [Volumetric Fur Rendering with LDPRT](14320-volumetric-fur-rendering-with-ldprt.md)
This method renders realistic microgeometry (specifically fur) with global lighting effects, such as self-shadowing, at real-time frame rates by combining a volumetric fur model with Local, Deformable Precomputed Radiance Transfer (LDPRT). It solves the problem of capturing microgeometric self-shadowing in a volumetric fur model by precomputing light attenuation to each voxel in the fur volume and rendering the volume using shell textures. Unlike general participating media methods, it specifically models the microgeometry of fur hairs using a multi-layered shell approach and defers the conversion of illuminance to luminous exitance to the reconstruction phase. Choose it for real-time fur rendering with self-shadowing.

### Modular Cell PRT
Utilize modular cells to precompute transfer, stitching multiple unit cells together and warping them to approximate scene geometry [S1, unofficial]. Propagate radiance first to cell borders acting as interfaces, then to neighboring cells using precomputed models [S1, unofficial].

## Background
Precomputed Radiance Transfer was introduced by Jensen et al. in 2001 [G]. The technique was originally developed for surface rendering to handle dynamic global illumination efficiently [G]. In volumetric rendering, it has been adapted to handle participating media by precomputing the transfer of radiance between different parts of the volume to address the computational cost of multiple scattering [G]. The approach has also been explored using modular cells to stitch and warp unit cells for approximating scene geometry [S1, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
