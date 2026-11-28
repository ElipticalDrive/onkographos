# Enhancement Styles
Also called: Cartoon, Edge Enhanced, Tone-mapped, Shaded

## Overview
Enhancement Styles modify volume appearance by computing surface normals for voxels and applying lighting or color mapping to emphasize structural features [S1]. They differ from standard opacity mapping by using geometric orientation (normals) rather than just intensity to drive visual properties, allowing for edge highlighting, stylized cartoon rendering, or physically-based shading [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume data (intensity values per voxel), viewpoint direction, and style-specific parameters (e.g., number of color steps for Cartoon, material properties for Shaded).
*   **Output:** Rendered image with enhanced features based on the selected style.

**Data Structures:**
*   **Surface Normal:** A 3D vector $(n_x, n_y, n_z)$ computed for each voxel.
*   **View Direction:** A 3D vector $(v_x, v_y, v_z)$ pointing from the voxel to the camera.

**Method:**
The core mechanism involves computing surface normals and then applying a style-specific transformation based on the angle between the normal and the view direction.

1.  **Compute Surface Normals:**
    For each voxel $i$, compute the surface normal $\mathbf{n}_i$ based on local intensity gradients. This is typically done using finite differences on the intensity field $I(x,y,z)$ [S1].
    $$ \mathbf{n}_i = \nabla I_i = \left( \frac{\partial I}{\partial x}, \frac{\partial I}{\partial y}, \frac{\partial I}{\partial z} \right)_i $$
    The normal is normalized: $\hat{\mathbf{n}}_i = \frac{\mathbf{n}_i}{\|\mathbf{n}_i\|}$. If the gradient magnitude is zero, the normal is undefined or set to a default (e.g., $[0,0,1]$) [G].

2.  **Calculate View Angle:**
    Compute the dot product between the normalized normal $\hat{\mathbf{n}}_i$ and the normalized view direction $\hat{\mathbf{v}}$ [S1]:
    $$ \cos \theta_i = \hat{\mathbf{n}}_i \cdot \hat{\mathbf{v}} $$
    The angle $\theta_i$ represents the orientation of the voxel surface relative to the viewer. Voxels with normals orthogonal to the view have $\cos \theta_i \approx 0$ ($\theta_i \approx 90^\circ$) [S1].

3.  **Apply Style-Specific Mapping:**
    The final color/opacity is determined by the selected Enhancement Style:

    *   **Edge/Boundary/Silhouette Styles:**
        Color voxels based on the angle between the normal and the viewpoint. Voxels whose normals are closer to orthogonal to the view are highlighted (e.g., colored with `edgeColor`), while parallel voxels remain unchanged or are dimmed [S1]. The exact mapping function (e.g., linear falloff from orthogonal) is not specified in the sources [S1].

    *   **Cartoon Style:**
        Applies the same normal-based coloring as Edge/Boundary styles but quantizes the output into a user-specified number of discrete color steps [S1]. Let $N$ be the number of steps. The continuous color value $C$ derived from the angle is mapped to one of $N$ discrete levels [S1].

    *   **Tone-mapped Style:**
        Maps colors directly to intensity values instead of opacity values, functioning similarly to the default opacity map but driven by intensity rather than geometric orientation [S1]. This style does not use the normal-based highlighting; it is an alternative to the geometric enhancement styles [S1].

    *   **Shaded Style:**
        Applies full custom materials, lighting models, and shadow calculations to the volume data [S1]. This requires additional inputs for material properties (diffuse, specular, etc.) and light sources [S1].

**Parameters:**
*   **Number of Color Steps:** Integer, user-specified, for Cartoon style [S1].
*   **Edge Color:** Color value for highlighted orthogonal voxels, for Edge/Boundary/Silhouette styles [S1].
*   **Material Properties:** Diffuse, specular, ambient coefficients, for Shaded style [S1].

**Edge Cases and Pitfalls:**
*   **Zero Gradient:** Voxels with constant intensity have zero gradient and thus no defined normal. A default normal must be assigned [G].
*   **Noise:** Intensity noise can lead to erratic normals. Smoothing or thresholding may be required [G].
*   **View Direction:** The view direction must be normalized for correct dot product calculation [G].

## Operating characteristics
Not given in the sources.

## Background
Enhancement Styles are part of the X3D volume rendering capabilities, specifically designed to provide non-photorealistic and feature-emphasizing visualization techniques for medical and scientific data [S1]. They are often used in conjunction with IsoSurface Volumes to display isosurface contours at specific threshold values [S1].

## Sources
[S1] NEW PLATFORMS FOR HEALTH HYPERMEDIA, https://doi.org/10.48009/1_iis_2012_40-50, fetched 2026-10-29
