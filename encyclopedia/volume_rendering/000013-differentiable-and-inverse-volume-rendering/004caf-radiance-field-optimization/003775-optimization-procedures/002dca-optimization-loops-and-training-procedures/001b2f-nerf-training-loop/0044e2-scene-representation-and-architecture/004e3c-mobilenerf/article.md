# MobileNeRF

Also called: MobileNeRF (June 2022)

## Overview
MobileNeRF is a framework that trains a NeRF-like model based on a polygonal mesh with color, feature, and opacity MLPs attached to each mesh vertex [S1]. It is designed to render novel images efficiently on mobile devices such as phones and light laptops [S2, unofficial]. It sets itself apart from standard volumetric NeRFs by replacing the ray-marching integration with a rasterization-based approach, achieving a speed approximately 10 times faster than SNeRG [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of images for training; a polygonal mesh geometry.
*   **Output:** Rendered novel views of the scene.

**Data Structures:**
The core representation is a polygonal mesh. Attached to each mesh vertex are three Multi-Layer Perceptrons (MLPs):
1.  **Color MLP:** Predicts color.
2.  **Feature MLP:** Predicts feature vectors.
3.  **Opacity MLP:** Predicts opacity.
The textures associated with the polygons consist of binary opacities and feature vectors [S2, unofficial].

**Steps:**
1.  **Training:** The framework trains the NeRF-like model using the polygonal mesh structure [S1].
2.  **Discretization:** Alpha values are discretized [S1].
3.  **Anti-aliasing:** Features are super-sampled for anti-aliasing [S1].
4.  **Rendering:**
    a. The mesh with associated features and opacities is rasterized according to the viewing position [S1].
    b. A small MLP is used to shade each pixel [S1].

**Pitfalls:**
*   The method relies upon a high-end GPU to optimize the textured polygons-based NeRF representation [S2, unofficial].
*   The proposed textured polygons-based representation lost the view-dependent effectiveness [S2, unofficial].

## Operating characteristics
*   **Performance:** The method demonstrated a speed approximately 10 times faster than SNeRG [S1].
*   **Hardware Requirements:** While designed for mobile devices, the optimization process still relies upon a high-end GPU [S2, unofficial].
*   **Limitations:** The representation lost view-dependent effectiveness [S2, unofficial].

## Background
MobileNeRF was introduced in June 2022 [S1]. It was proposed as a new NeRF representation based on a set of polygons with textures that consist of binary opacities and feature vectors [S2, unofficial].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] NEPHELE: A Neural Platform for Highly Realistic Cloud Radiance Rendering, https://arxiv.org/abs/2303.04086v1, fetched 2026-10-27 (unofficial)
