# Illumination Caching

Also called: Sundén and Ropinski implementation, TBRayMarcher Illumination Cache, TBRayMarcher Plugin

## Overview
Illumination caching is a direct volume ray casting technique that pre-computes the lighting distribution inside a volume and stores it in a texture, allowing for real-time rendering of complex lighting [S1, unofficial]. It relies on the assumption that volume material scatters light isotropically, meaning the lighting at any point depends only on its position and not on the viewing direction or the path taken to reach it [S1, unofficial]. This method is distinct from standard per-sample shading (e.g., [Direct Volume Rendering with Phong Shading](3686-direct-volume-rendering-with-phong-shading.md)) by replacing expensive per-sample shadow ray traversals with single-texture lookups of pre-aggregated illumination data, achieving significant performance speedups [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume dataset $V$ and a set of light sources (position, color, intensity).
*   **Output:** An illumination cache texture (3D texture) storing pre-computed lighting values, and the final rendered image.

**Data Structures:**
The core data structure is a "true" 3D volume texture [S1, unofficial].
*   **Layout:** The texture stores lighting values (e.g., ambient occlusion factors, shadow attenuation, or full illumination terms) at discrete voxel coordinates.
*   **Extents:** The cache does not necessarily need to have the same spatial extents as the input volume dataset [S1, unofficial]. It may be smaller or larger depending on the region of interest or lighting influence.
*   **Format:** Typically stored in floating-point or fixed-point formats suitable for interpolation, though specific bit-depths are not detailed in the primary sources [S1, unofficial].

**Algorithm Steps:**
1.  **Pre-computation (Cache Generation):**
    *   For each voxel $\mathbf{p}$ in the cache domain:
        *   Compute the illumination contribution from all light sources.
        *   Assuming isotropic scattering, calculate the attenuation of light reaching $\mathbf{p}$ from each light source. This typically involves marching rays from the light source to $\mathbf{p}$ or from $\mathbf{p}$ to the light source to estimate extinction/shadowing [S1, unofficial].
        *   Store the resulting illumination value $L_{cache}(\mathbf{p})$ in the 3D texture at index corresponding to $\mathbf{p}$ [S1, unofficial].
2.  **Rendering (Ray Casting):**
    *   For each pixel in the image, cast a ray through the volume [19-direct-volume-ray-casting.md].
    *   For each sample point $\mathbf{p}_i$ along the ray:
        *   Sample the illumination cache texture at $\mathbf{p}_i$ to retrieve $L_{cache}(\mathbf{p}_i)$ [S1, unofficial].
        *   Compute the base color $C_{base}$ from the scalar value $V(\mathbf{p}_i)$ using transfer functions [19-direct-volume-ray-casting.md].
        *   Compute the shaded color: $C_i = C_{base} \cdot L_{cache}(\mathbf{p}_i)$ [S1, unofficial].
        *   Accumulate $C_i$ using front-to-back alpha compositing [19-direct-volume-ray-casting.md].

**Parameters:**
*   **Cache Resolution:** The resolution of the 3D texture storing the illumination. Higher resolution yields more accurate lighting but increases memory usage and pre-computation time. Not given in the sources.
*   **Light Source Count:** The number of light sources contributing to the cache. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Chunking Artifacts:** When the volume dataset exceeds the maximum texture dimensions (e.g., 2048 voxels per dimension in DirectX 11/12 [S1, unofficial]), the volume must be chunked into multiple sub-volumes. Each chunk is lit separately, ignoring occlusion from adjacent chunks. This results in incorrect lighting at chunk borders, making them appear incorrectly bright [S1, unofficial].
*   **Hardware Limitations:** The method relies on "true" volume textures. Features like DirectX 12 reserved resources (which allow managing very large, fragmented VRAM for textures up to 8 gigavoxels) are not supported by the primary implementation (TBRayMarcher), limiting the maximum usable texture size [S1, unofficial].
*   **Isotropic Assumption:** The method assumes isotropic scattering. If the volume exhibits strong anisotropic scattering, the pre-computed cache will be inaccurate [S1, unofficial].

## Subtopics

### [Light Cache Rendering Pipeline](9159-light-cache-rendering-pipeline.md)
The Light Cache Rendering Pipeline is a direct volume ray-casting technique that pre-computes illumination attenuation factors (ambient occlusion and directional shadows) into a 3D texture, allowing for real-time rendering of complex lighting in volumetric data. It distinguishes itself from standard ray-casting by replacing per-sample shadow ray traversal with single-texture lookups of pre-aggregated extinction coefficients, achieving speedups of over 45 times while maintaining visual quality comparable to dense sampling.

### Radiance Caching (Jarosz et al.)
Radiance caching uses spherical harmonic functions to represent and store radiance distributions of certain sparse points and related gradient information [S2]. The illumination at a new position is interpolated from the existing distribution and gradient, thereby reducing the amount of calculation and providing a smoother result [S2]. Since the stored sample points are sparse, points to be calculated may be differently occluded compared to the stored points, resulting in different illumination; the authors did not consider visibility during interpolation [S2]. This problem was later addressed by Marco et al. [58] using a second-order Hessian error metric to determine interpolation acceptability, which considers visibility and produces higher quality results [S2].

## Operating characteristics
**Time and Memory Complexity:**
*   **Pre-computation:** The pre-computation phase is expensive, as it requires evaluating lighting for every voxel in the cache domain. The complexity is proportional to the number of voxels in the cache times the number of light sources and the cost of shadow ray marching [S1, unofficial].
*   **Rendering:** The rendering phase is significantly faster than standard per-sample shading because it replaces shadow ray marching with a single texture lookup per sample [S1, unofficial].
*   **Memory:** The method requires additional memory for the illumination cache texture. The size depends on the cache resolution and the number of channels stored (e.g., RGB or scalar) [S1, unofficial].

**Measured Performance:**
*   Evaluated on an NVIDIA RTX 3500 Ada GPU with a chunked Kolumbo dataset (5.9 billion voxels, chunked into nine sub-volumes of 512x935x1501 voxels) [S1, unofficial].
*   The method achieves speedups of over 45 times compared to dense sampling methods [S2].

**Strengths and Limitations:**
*   **Strengths:** Enables real-time rendering of complex lighting in large volumes; reduces per-frame computational cost significantly [S1, unofficial].
*   **Limitations:** Requires pre-computation time; assumes isotropic scattering; suffers from lighting artifacts at chunk borders when large volumes are chunked; limited by maximum texture dimensions in some graphics APIs (e.g., DirectX 11/12 limit of 2048 voxels per dimension) [S1, unofficial].

**When to Use:**
*   Use when real-time performance is critical and the volume exhibits approximately isotropic scattering.
*   Suitable for large datasets where per-sample shadow ray marching is too slow.

**When Not to Use:**
*   Do not use if the volume exhibits strong anisotropic scattering.
*   Avoid if the volume cannot be chunked without unacceptable lighting artifacts at borders.
*   Avoid if the hardware does not support large "true" volume textures or if the required cache size exceeds available VRAM.

## Background
The illumination caching technique described here is based on an implementation by Sundén and Ropinski [16], which is used in the TBRayMarcher plugin for Unreal Engine [S1, unofficial]. The plugin was evaluated in the context of rendering large volume datasets in Unreal Engine 5 [S1, unofficial]. The underlying assumption of isotropic scattering allows the lighting distribution to be pre-computed and stored, a concept related to radiance caching methods such as those by Jarosz et al. [S2].

## Sources
[S1] Rendering Large Volume Datasets in Unreal Engine 5: A Survey, https://www.researchgate.net/publication/390671564_Rendering_Large_Volume_Datasets_in_Unreal_Engine_5_A_Survey, fetched 2026-10-28 (unofficial)
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
