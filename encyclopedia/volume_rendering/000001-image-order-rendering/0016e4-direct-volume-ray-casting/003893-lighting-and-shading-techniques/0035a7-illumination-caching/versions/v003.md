# Illumination Caching

Also called: Sundén and Ropinski implementation, TBRayMarcher Illumination Cache, TBRayMarcher Plugin

## Overview
Illumination caching is a rendering technique that pre-computes the lighting distribution inside a volume and stores it in a 3D texture, allowing for real-time rendering of complex lighting effects such as shadows and ambient occlusion [S1, unofficial]. It distinguishes itself from analytic shading methods (e.g., [Direct Volume Rendering with Phong Shading](3686-direct-volume-rendering-with-phong-shading.md)) by replacing expensive per-sample shadow ray traversals with single-texture lookups of pre-aggregated extinction coefficients [S1, unofficial]. This approach assumes isotropic light scattering to enable the pre-computation of attenuation factors [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset (scalar field) and a set of light sources (position, color, intensity).
*   **Output:** An illumination cache volume (3D texture) storing pre-computed lighting values, which is then used during the ray casting loop defined in [Direct volume ray casting](19-direct-volume-ray-casting.md).

**Data Structures:**
*   **Illumination Cache:** A "true" 3D volume texture (not a pseudo-volume texture) [S1, unofficial].
    *   **Layout:** The cache does not necessarily need to have the same extents as the source volume dataset [S1, unofficial].
    *   **Content:** Stores pre-computed illumination attenuation factors (ambient occlusion and directional shadows) [S1, unofficial].
    *   **Resolution:** Limited by hardware constraints; maximum extents along each dimension are typically 2048 voxels in DirectX 11 and DirectX 12 [S1, unofficial].

**Steps:**
1.  **Assumption:** Assume the volume material scatters light isotropically (equally in all directions) [S1, unofficial].
2.  **Pre-computation:** For each light source, traverse the volume to compute the lighting distribution. This involves calculating the attenuation of light as it passes through the volume, effectively simulating shadows and ambient occlusion [S1, unofficial].
3.  **Storage:** Store the computed lighting values in the illumination cache 3D texture [S1, unofficial].
4.  **Rendering:** During the ray casting loop, for each sample point $\mathbf{p}_i$, retrieve the pre-computed lighting term $L(\mathbf{p}_i)$ from the illumination cache via texture lookup [S1, unofficial].
5.  **Shading:** Apply the retrieved lighting term to the base color $C_{base}(\mathbf{p}_i)$ as per the standard ray casting shading model:
    $$
    C_i = C_{base}(\mathbf{p}_i) \cdot L(\mathbf{p}_i)
    $$
    [S1, S8307, unofficial].

**Parameters:**
*   **Cache Extents:** Can differ from the volume dataset extents [S1, unofficial].
*   **Chunk Size:** If the volume exceeds texture limits, it must be chunked. Typical chunk sizes observed: $512 \times 935 \times 1501$ voxels [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Chunking Artifacts:** When large volumes are chunked into multiple textures (due to the 2048 voxel limit per dimension), each chunk is lit separately as if other chunks do not exist [S1, unofficial]. This results in incorrect lighting at chunk borders, making them appear incorrectly bright [S1, unofficial].
*   **Hardware Limits:** DirectX 12 reserved resources feature is not supported by the TBRayMarcher plugin, preventing the use of very large volume textures (up to 8 gigavoxels) that require fragmented VRAM management [S1, unofficial].
*   **Driver/Backend Issues:** The plugin works best with DirectX 11. DirectX 12 support is problematic, and Vulkan is reported to not work at all [S1, unofficial].
*   **VRAM Constraints:** VRAM fragmentation and available free memory can cause crashes well below the theoretical 2048 voxel limit per dimension [S1, unofficial].

## Subtopics

### Radiance Caching (Jarosz et al.)
Radiance caching uses spherical harmonic functions to represent and store radiance distributions of sparse points along with gradient information [S2]. Illumination at a new position is interpolated from the existing distribution and gradient, reducing calculation cost and providing smoother results [S2]. However, because stored sample points are sparse, points to be calculated may be differently occluded compared to stored points, leading to incorrect illumination; the original authors did not consider visibility during interpolation [S2]. This visibility issue was later addressed by Marco et al. using a second-order Hessian error metric [S2].

## Operating characteristics
*   **Performance:** Drastically improves rendering performance by replacing secondary shadow ray marches with texture lookups [S1, unofficial]. Speedups of over 45 times have been reported while maintaining visual quality comparable to dense sampling [S1, unofficial].
*   **Hardware:** Evaluated on an NVIDIA RTX 3500 Ada GPU [S1, unofficial].
*   **Memory:** VRAM is a significant limiting factor, especially for large datasets requiring chunking [S1, unofficial].
*   **Scalability:** Limited by the 2048 voxel dimension limit in DirectX 11/12, requiring chunking for larger datasets, which introduces visual artifacts at borders [S1, unofficial].
*   **Strengths:** Real-time rendering of complex lighting (shadows, AO) in large volumes [S1, unofficial].
*   **Limitations:** Chunking artifacts, lack of DirectX 12 reserved resource support, and potential for incorrect lighting at chunk boundaries [S1, unofficial].

## Background
The technique is based on the implementation by Sundén and Ropinski [S1, unofficial]. It is utilized in the TBRayMarcher plugin for Unreal Engine [S1, unofficial]. The method relies on the simplifying assumption that volume material scatters light isotropically [S1, unofficial].

## Sources
[S1] Rendering Large Volume Datasets in Unreal Engine 5: A Survey, https://www.researchgate.net/publication/390671564_Rendering_Large_Volume_Datasets_in_Unreal_Engine_5_A_Survey, fetched 2026-10-28 (unofficial)
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
