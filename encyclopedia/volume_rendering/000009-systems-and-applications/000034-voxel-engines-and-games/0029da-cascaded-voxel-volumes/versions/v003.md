# Cascaded Voxel Volumes


## Overview
Cascaded Voxel Volumes is a spatial partitioning method for Voxelized Global Illumination (VXGI) that extends the concept of cascaded shadow maps to volumetric data [S1, S2, unofficial]. It addresses the problem of balancing memory usage and spatial resolution in large scenes by using multiple volumes with identical resolution but different scales, centered around the player [S1, unofficial]. Unlike single-volume approaches that must cover the entire scene or suffer from low resolution at distance, this method provides high-resolution coverage near the eye and lower-resolution coverage further away [S2, unofficial].

## How it works
**Inputs:** Camera position and orientation, scene geometry, and cascade configuration parameters [S1, S2, unofficial].

**Outputs:** A set of $N$ 3D textures (voxel grids), each representing a different spatial scale centered on the camera [S2, unofficial].

**Data Structures:** An array of $N$ volume instances, where each instance contains a 3D texture of fixed resolution $R \times R \times R$ (identical for all cascades) and a transformation matrix defining its world-space position and scale [S1, S2, unofficial].

**Steps:**
1. **Cascade Setup:** Define $N$ cascades. Cascade 0 is the smallest scale (highest resolution) centered tightly around the camera. Subsequent cascades $i$ have increasing scales $s_i$ (where $s_{i+1} > s_i$) and cover larger volumes of the scene [S1, S2, unofficial].
2. **Voxelization:** For each frame (or staggered frames):
   a. Position each cascade volume in world space centered on the camera, scaled according to $s_i$ [S1, S2, unofficial].
   b. Rasterize scene geometry into the 3D texture of cascade $i$ by projecting geometry onto the orthographic view of the cascade and filling the corresponding voxels [S2, unofficial].
   c. **Clamping:** Clamp the axis-aligned orthographic projections to world-space voxel-sized increments to prevent color swimming [S2, unofficial].
3. **Update Strategy:** Prioritize the cascade closest to the player (Cascade 0) for updates [S1, unofficial]. Stagger updates of other cascades across multiple frames to reduce per-frame computational load [S1, unofficial].
4. **Rendering:** During ray casting or cone tracing for GI, sample from the appropriate cascade based on the ray's distance from the camera [S2, unofficial].

**Formulas:**
*   **Cascade Scale:** The physical size of cascade $i$ is proportional to $s_i$ [S1, S2, unofficial].
*   **Clamping:** Let $P$ be the projected coordinate. The clamped coordinate $P'$ is:
    $$ P' = \text{floor}(P / \Delta) \cdot \Delta $$
    where $\Delta$ is the world-space voxel size of the cascade [S2, unofficial].

**Parameters:**
*   **Number of Cascades ($N$):** Not given in the sources.
*   **Resolution ($R$):** Identical for all cascades. Not given in the sources.
*   **Scale Factors ($s_i$):** Increasing values. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Color Swimming:** Moving the camera causes different fragments to be rendered, creating different voxels and visible color shifts [S2, unofficial]. Fixed by clamping projections to voxel-sized increments [S2, unofficial].
*   **Cascade Edges:** Discontinuities may occur at the boundaries between cascades. A linear interpolation between the two values at the boundary is suggested as a potential solution [S2, unofficial].
*   **Voxels Behind Eye:** Some voxels remain behind the eye because objects in front of the eye may still receive indirect color from objects behind the eye [S2, unofficial].

## Subtopics

### Centered Voxels (Cascaded Voxelization)
A specific implementation variant where multiple 3D textures create cascades that surround the eye, voxelizing only the geometry surrounding the eye rather than the entire scene [S2, unofficial]. It is chosen for open-world style games to make voxel-based global illumination possible by allowing farther areas to benefit from indirect illumination at a less-detailed level [S2, unofficial]. Key challenges include fixing voxel color swimming by clamping axis-aligned orthographic projections to world-space voxel-sized increments and handling edges between cascades, potentially via linear interpolation [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Staggering updates reduces per-frame computational load by updating cascades across multiple frames [S1, unofficial]. Exact complexity formulas are not given in the sources.
*   **Memory Complexity:** Memory usage scales with the number of cascades and fixed resolution, but exact complexity is not given in the sources [S1, S2, unofficial].
*   **Performance:** Used in 'The Tomorrow Children' on PS4 for efficient coverage of large scenes [S1, unofficial]. NVIDIA's VXGI in Unreal Engine 4 uses this because specular reflections lose quality at greater distances, justifying high resolution near the player [S1, unofficial].
*   **Strengths:** High resolution near the player; efficient coverage of large scenes [S1, S2, unofficial].
*   **Limitations:** Potential for aliasing/color swimming if not clamped; discontinuities at cascade boundaries [S1, S2, unofficial].

## Background
The method was proposed as an extension of cascaded shadow maps to VXGI [S1, unofficial]. It was implemented in 'The Tomorrow Children' by Q-Games on PS4 [S1, unofficial] and observed in NVIDIA's VXGI implementation in Unreal Engine 4 [S1, unofficial]. The concept of using multiple cascades to surround the eye was discussed in the context of Gigavoxels and sparse voxelization by Sean Lilley in 2012 [S2, unofficial].

## Sources
[S1] Voxel Cone Traced Global Illumination - Leif Node, https://leifnode.com/2015/05/voxel-cone-traced-global-illumination/, fetched 2026-10-27 (unofficial)
[S2] http://realtimevoxels.blogspot.fr/2012/09/introduction-to-gigavoxels-and-sparse.html, http://realtimevoxels.blogspot.fr/2012/09/introduction-to-gigavoxels-and-sparse.html, fetched 2026-10-28 (unofficial)
