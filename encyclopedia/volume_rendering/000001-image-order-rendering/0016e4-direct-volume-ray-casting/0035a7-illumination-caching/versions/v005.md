# Illumination Caching

Also called: Sundén and Ropinski implementation, TBRayMarcher Illumination Cache, TBRayMarcher Plugin

## Overview
Illumination caching is a rendering technique that pre-computes the lighting distribution inside a volume and stores it in a 3D texture, drastically improving rendering performance [S1, unofficial]. It assumes that volume material scatters light isotropically, allowing the lighting distribution to be aggregated into the cache [S1, unofficial]. It distinguishes itself from standard ray-casting by storing pre-computed lighting values in a texture rather than computing them per-sample during rendering [S1, unofficial].

## How it works
The method operates within the standard direct volume ray casting loop defined in [Direct volume ray casting](19-direct-volume-ray-casting.md) [G]. It replaces the on-the-fly computation of the lighting term with a lookup from a pre-computed illumination cache [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** The scalar volume dataset, light source positions and intensities, and the pre-computed illumination cache texture [S1, unofficial].
*   **Output:** The final image with lighting effects applied [S1, unofficial].

**Data Structures:**
*   **Illumination Cache:** A "true" 3D volume texture rather than a pseudo-volume texture [S1, unofficial].
    *   **Layout:** The cache stores pre-computed lighting values. The exact number of channels or bit-depth is not specified for the cache, though the source notes normalizing volume data to 8-bit precision [S1, unofficial].
    *   **Extents:** The cache does not necessarily need to have the same extents as the volume dataset itself [S1, unofficial].

**Steps:**
1.  **Pre-computation:** The lighting distribution inside the volume is pre-computed and stored in the illumination cache, assuming isotropic scattering [S1, unofficial]. *Note: The exact pre-computation algorithm is not detailed in the sources.*
2.  **Rendering:** During ray casting, the pre-computed lighting value is retrieved from the illumination cache texture for each sample point [S1, unofficial].

**Formulas:** Not given in the sources.

**Parameters:**
*   **Cache Resolution:** Not given in the sources.
*   **Chunk Size:** When chunking is required, example sub-volumes are $512 \times 935 \times 1501$ voxels [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Chunking Artifacts:** If the volume dataset exceeds maximum texture dimensions, it must be chunked into multiple volume textures. This results in incorrect lighting at chunk borders because each chunk is lit separately as if others do not exist, making borders incorrectly bright [S1, unofficial].
*   **Hardware Limits:** The maximum extents along each dimension for volume textures in DirectX 11 and DirectX 12 is 2048 voxels, requiring chunking for larger datasets [S1, unofficial].
*   **VRAM Fragmentation:** The implementation does not support DirectX 12 reserved resources, preventing the use of very large volume textures (up to 8 gigavoxels) that require fragmented VRAM management [S1, unofficial].
*   **API Compatibility:** The implementation works best with DirectX 11. It has reported issues with DirectX 12 and does not work with Vulkan [S1, unofficial].

## Subtopics

### Radiance Caching (Jarosz et al.)
Radiance caching uses spherical harmonic functions to represent and store radiance distributions of sparse points and related gradient information [S2]. The illumination at a new position is interpolated from the existing distribution and gradient, reducing calculation and providing a smoother result [S2]. However, since stored sample points are sparse, points to be calculated may be differently occluded compared to stored points, resulting in different illumination; the authors did not consider visibility during interpolation [S2]. This problem was later addressed by Marco et al. using a second-order Hessian error metric to determine if interpolation error is acceptable, considering visibility for higher quality [S2].

## Operating characteristics
*   **Time Complexity:** Rendering time is drastically improved because per-sample lighting computation is replaced by texture lookups [S1, unofficial].
*   **Memory Complexity:** Requires additional memory for the illumination cache texture [S1, unofficial].
*   **Measured Performance:** Evaluated on an NVIDIA RTX 3500 Ada GPU with a chunked Kolumbo dataset (5.9 billion voxels, chunked into nine sub-volumes of $512 \times 935 \times 1501$ voxels) [S1, unofficial].
*   **Strengths:** Drastically improves rendering performance for complex lighting [S1, unofficial].
*   **Limitations:** Chunking causes lighting artifacts at borders [S1, unofficial]. Limited by texture dimension limits (2048 voxels per dimension) and VRAM fragmentation issues in DirectX 12 [S1, unofficial].
*   **When to use:** When real-time rendering of complex lighting in volumetric data is required and the volume can be pre-computed [S1, unofficial].
*   **When not to use:** When the volume exceeds hardware texture limits without acceptable chunking artifacts [S1, unofficial].

## Background
The technique is based on the assumption that volume material scatters light isotropically [S1, unofficial]. It is implemented in the TBRayMarcher Plugin for Unreal Engine, which is based on an implementation by Sundén and Ropinski [S1, unofficial]. The plugin supports importing DICOM and MHD files and was evaluated on Unreal Engine 5.3 and 5.4 [S1, unofficial].

## Sources
[S1] Rendering Large Volume Datasets in Unreal Engine 5: A Survey, https://www.researchgate.net/publication/390671564_Rendering_Large_Volume_Datasets_in_Unreal_Engine_5_A_Survey, fetched 2026-10-28 (unofficial)
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
