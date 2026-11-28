# Unbounded Scene Handling (NDC and Background Model)
Also called: Background Spheres, NDC and Background Model

## Overview
This method extends Plenoxels to render unbounded scenes, addressing the limitation of standard bounded voxel grids. It distinguishes between forward-facing and 360° scenes: forward-facing scenes use Normalized Device Coordinates (NDC) to handle depth, while 360° scenes augment the foreground voxel grid with a Multisphere Image (MSI) background model to represent distant geometry [S1][S2, unofficial].

## How it works
The method adapts the discrete voxel representation based on the scene's camera trajectory.

### Inputs and Outputs
*   **Inputs:** Camera poses, image observations, scene type (forward-facing or 360°).
*   **Outputs:** Radiance and density fields represented by sparse voxel grids (foreground) and potentially multisphere layers (background).

### Data Structures
1.  **Foreground Voxel Grid:** A sparse 3D grid storing RGB colors and opacity/density values.
    *   **Forward-facing:** Coordinates are mapped to NDC space.
    *   **360°:** Coordinates are standard 3D, but the background is handled separately.
2.  **Multisphere Image (MSI) Layers (360° only):**
    *   Consists of 64 concentric spheres.
    *   Each sphere stores a 2D texture map (voxel grid) of RGB colors and opacities.
    *   **Storage:** Only RGB channels (zero-order spherical harmonics) are stored to conserve memory.
    *   **Sparsity:** Layers are stored sparsely using opacity/density thresholding [S1][S2, unofficial].

### Steps

1.  **Scene Classification:** Determine if the scene is forward-facing or 360°.
2.  **Coordinate Transformation:**
    *   **If Forward-Facing:** Transform camera ray origins and directions into Normalized Device Coordinates (NDC) as defined in the original NeRF paper [S1][S2, unofficial]. This maps the unbounded depth range to a finite interval, allowing the use of a standard bounded voxel grid [S1][S2, unofficial].
    *   **If 360°:** Keep coordinates in standard Euclidean space for the foreground.
3.  **Background Model Construction (360° only):**
    *   Pre-scale the inner scene so it is approximately contained within the unit sphere [S1][S2, unofficial].
    *   Place 64 spheres linearly in inverse radius from $r=1$ to $r=\infty$. The radius of the $i$-th sphere ($i=0 \dots 63$) is determined by the inverse radius spacing: $1/r_i$ is linearly spaced between $1/1$ and $1/\infty$ (i.e., $1/r_i = 1 - i/64$) [S1][S2, unofficial].
    *   For each sphere, voxels are indexed by spherical angles $\theta$ and $\phi$ using an equirectangular projection [S1][S2, unofficial].
    *   Learn voxel colors (RGB) and opacities for each sphere layer using trilinear interpolation within and between spheres [S1][S2, unofficial].
4.  **Rendering:**
    *   For each pixel, cast a ray.
    *   If the ray intersects the foreground voxel grid, sample radiance/density using trilinear interpolation.
    *   If the ray extends beyond the foreground (or for 360° scenes), intersect the MSI layers. Sample the appropriate sphere based on distance, using equirectangular coordinates to index the 2D voxel grid.
    *   Composite foreground and background contributions using standard volume rendering integration [S1][S2, unofficial].

### Parameters
*   **Number of Background Spheres:** 64 [S1][S2, unofficial].
*   **Radius Distribution:** Linear in inverse radius from 1 to $\infty$ [S1][S2, unofficial].
*   **Color Representation:** RGB only (zero-order SH) [S1][S2, unofficial].
*   **Sparsity Threshold:** Opacity/density thresholding is used to prune empty background layers [S1][S2, unofficial].

### Edge Cases and Pitfalls
*   **Precision:** The transition from foreground to background must be smooth to avoid visible seams. The pre-scaling of the inner scene to the unit sphere is critical for correct sphere placement [S1][S2, unofficial].
*   **Memory:** Storing 64 full-resolution spheres is memory-intensive. The use of sparsity (opacity thresholding) and zero-order SH is essential to keep memory feasible [S1][S2, unofficial].
*   **Forward-Facing Assumption:** NDC is only valid for forward-facing scenes where the camera does not orbit 360° around the center. Using NDC for 360° scenes causes geometric distortions [S1][S2, unofficial].

## Subtopics

### Plenoxel Unbounded Scene Handling
This variant specifically refers to the implementation details in the Plenoxels paper for handling unbounded scenes. It distinguishes two modes: (1) Forward-facing scenes using NDC coordinates [S2, unofficial], and (2) 360° scenes using a Multisphere Image (MSI) background model. The MSI model uses 64 spheres linearly spaced in inverse radius from 1 to $\infty$, with voxels warped into spheres via equirectangular projection. Colors are stored as RGB (zero-order SH) and layers are stored sparsely using density thresholding [S2, unofficial].

## Operating characteristics
*   **Memory:** The background model adds significant memory overhead compared to bounded scenes. However, storing only RGB and using sparsity mitigates this. The foreground grid remains sparse [S1][S2, unofficial].
*   **Performance:** Rendering speed is comparable to bounded Plenoxels, with the added cost of checking background spheres. The use of zero-order SH simplifies the color evaluation [S1][S2, unofficial].
*   **Strengths:** Enables training on real-world unbounded datasets (e.g., forward-facing street scenes, 360° indoor/outdoor captures) without requiring manual bounding box estimation [S1][S2, unofficial].
*   **Limitations:** The NDC approach is limited to forward-facing cameras. The 360° background model assumes the background is roughly spherical and distant; it may not handle complex, close-range unbounded geometry well [S1][S2, unofficial].

## Background
This method was introduced in the Plenoxels paper (Fridovich-Keil et al., 2022) to extend the efficiency of radiance field representation without neural networks to real-world unbounded scenes [S1][S2, unofficial]. It builds on the NDC coordinate system from the original NeRF paper [S1][S2, unofficial] and the multisphere background model from NeRF++ [S1].

## Sources
[S1] Plenoxels: Radiance Fields without Neural Networks, https://doi.org/10.1109/cvpr52688.2022.00542, fetched 2026-10-26
[S2] CVPR 2022 Open Access Repository, https://openaccess.thecvf.com/content/CVPR2022/html/Fridovich-Keil_Plenoxels_Radiance_Fields_Without_Neural_Networks_CVPR_2022_paper.html, fetched 2026-10-27 (unofficial)
