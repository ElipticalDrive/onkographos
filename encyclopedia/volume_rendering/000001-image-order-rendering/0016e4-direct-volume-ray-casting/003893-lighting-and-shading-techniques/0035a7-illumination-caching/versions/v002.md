# Illumination Caching

Also called: Sundén and Ropinski implementation, TBRayMarcher Illumination Cache, TBRayMarcher Plugin

## Overview
Illumination caching is a rendering technique that pre-computes the lighting distribution inside a volume and stores it in a 3D texture, allowing for real-time rendering of complex lighting [S1, unofficial]. It assumes that volume material scatters light isotropically, enabling the attenuation factors (such as ambient occlusion and directional shadows) to be aggregated into a single lookup [S1, unofficial]. This method distinguishes itself from standard ray-casting by replacing per-sample shadow ray traversal with single-texture lookups of pre-aggregated extinction coefficients, achieving significant performance speedups while maintaining visual quality [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume dataset $V$ and a set of light sources (position, color, intensity).
*   **Output:** An illumination cache volume (3D texture) containing pre-computed lighting values, and the final rendered image.

**Data Structures:**
*   **Illumination Cache:** A "true" 3D volume texture [S1, unofficial]. It does not necessarily need to have the same extents as the original volume dataset [S1, unofficial].
*   **Volume Texture:** Stored in hardware-supported 3D texture formats.

**Steps:**
1.  **Pre-computation (Cache Generation):**
    *   For each light source, traverse the volume to compute the lighting contribution at sample points.
    *   The assumption of isotropic scattering allows the accumulation of light attenuation (shadows, ambient occlusion) into the cache [S1, unofficial].
    *   Store the resulting illumination values in the 3D texture.
2.  **Rendering:**
    *   During the ray-casting loop, for each sample point $\mathbf{p}$, retrieve the pre-computed illumination value $I_{cache}(\mathbf{p})$ from the cache texture [S1, unofficial].
    *   Combine $I_{cache}(\mathbf{p})$ with the base color $C_{base}$ derived from the transfer function.
    *   Integrate the shaded color into the front-to-back compositing loop as defined in [Lighting and Shading Techniques](8281-lighting-and-shading-techniques.md).

**Parameters:**
*   **Cache Resolution:** The resolution of the illumination cache texture. Not given in the sources.
*   **Light Sources:** Position, color, and intensity. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Chunking Artifacts:** If the volume dataset exceeds the maximum texture dimensions, it must be chunked. Lighting is computed per chunk, ignoring occlusion from adjacent chunks, resulting in incorrectly bright borders [S1, unofficial].
*   **Hardware Limits:** Maximum extents along each dimension for volume textures in DirectX 11 and DirectX 12 is 2048 voxels [S1, unofficial].
*   **VRAM Fragmentation:** Large volumes may require fragmented VRAM management, which is not supported by the plugin in DirectX 12 [S1, unofficial].

## Subtopics

### [Light Cache Rendering Pipeline](9159-light-cache-rendering-pipeline.md)
The Light Cache Rendering Pipeline is a direct volume ray-casting technique that pre-computes illumination attenuation factors (ambient occlusion and directional shadows) into a 3D texture, allowing for real-time rendering of complex lighting in volumetric data. It distinguishes itself from standard ray-casting by replacing per-sample shadow ray traversal with single-texture lookups of pre-aggregated extinction coefficients, achieving speedups of over 45 times while maintaining visual quality comparable to dense sampling.

### Radiance Caching (Jarosz et al.)
Radiance caching uses spherical harmonic functions to represent and store radiance distributions of certain sparse points and related gradient information [S2]. The illumination at a new position is interpolated from the existing distribution and gradient, thereby reducing the amount of calculation and providing a smoother result [S2]. Since the stored sample points are sparse, points to be calculated may be differently occluded compared to the stored points, resulting in different illumination; the authors did not consider visibility during interpolation [S2]. This problem was solved by Marco et al. [58], by using the second-order Hessian error metric to determine whether the interpolation error is acceptable. Since the visibility is considered, it produces results with higher quality than those of Jarosz et al. [57] [S2].

## Operating characteristics
**Time and Memory Complexity:**
*   **Pre-computation:** High initial cost proportional to the number of light sources and volume samples, as shadow rays must be marched for each cache sample [S1, unofficial].
*   **Rendering:** Low cost per frame, dominated by 3D texture lookups and compositing [S1, unofficial].

**Measured Performance:**
*   Evaluated on an NVIDIA RTX 3500 Ada GPU with a chunked Kolumbo dataset (5.9 billion voxels, chunked into nine sub-volumes of 512x935x1501 voxels) [S1, unofficial].
*   Achieves speedups of over 45 times compared to dense sampling methods [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Enables real-time rendering of complex lighting in large volumetric datasets [S1, unofficial].
*   **Limitations:**
    *   Chunking leads to incorrect lighting at chunk borders [S1, unofficial].
    *   Limited by maximum texture dimensions (2048 voxels per dimension in DX11/12) [S1, unofficial].
    *   DirectX 12 reserved resources feature is not supported, preventing the use of very large volume textures (up to 8 gigavoxels) that require fragmented VRAM management [S1, unofficial].
    *   Vulkan is reported to not work at all with the TBRayMarcher plugin [S1, unofficial].

**When to Use:**
*   When real-time performance is required for large volumetric datasets with complex lighting.
*   When the volume data can fit within texture dimension limits or when chunking artifacts are acceptable.

**When Not to Use:**
*   When high-fidelity lighting at chunk borders is critical.
*   When using DirectX 12 with very large datasets requiring fragmented VRAM management.
*   When using Vulkan as the rendering backend.

## Background
Illumination caching is based on an implementation by Sundén and Ropinski [16], as used in the TBRayMarcher Plugin for the Unreal Engine [S1, unofficial]. The technique relies on the simplifying assumption that the volume material scatters light isotropically, allowing the lighting distribution inside the volume to be pre-computed and stored in an illumination cache [S1, unofficial].

## Sources
[S1] Rendering Large Volume Datasets in Unreal Engine 5: A Survey, https://www.researchgate.net/publication/390671564_Rendering_Large_Volume_Datasets_in_Unreal_Engine_5_A_Survey, fetched 2026-10-28 (unofficial)
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
