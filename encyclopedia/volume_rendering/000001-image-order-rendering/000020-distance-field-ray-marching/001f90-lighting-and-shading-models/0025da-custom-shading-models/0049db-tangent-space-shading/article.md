# Tangent Space Shading

Also called: vzTangentSpaceShader, vzTMTangentSpaceShader

## Overview
Tangent Space Shading is a gradient-less lighting technique that approximates the diffuse lighting component of the Phong shading model without computing explicit gradients for every texel [S1]. It is distinguished from sibling methods like [GGX Microfacet BRDF](3715-ggx-microfacet-brdf.md) and [Light Evaluation for SDF Faces](3751-light-evaluation-for-sdf-faces.md) by its reliance on rendering artifacts (double-pass texture coordinate offsets) rather than geometric derivatives to simulate surface orientation [S1, S2].

## How it works
**Inputs:**
*   Volume data (voxels).
*   Light direction vector $\mathbf{l}$.
*   View direction (implicit via camera).

**Outputs:**
*   Rendered image with diffuse shading approximation.

**Data Structures:**
*   Volume texture (bricked or continuous).
*   Two render passes with modified texture coordinates.

**Steps:**
1.  **First Pass:** Draw the volume slice using standard texture coordinates.
2.  **Second Pass:** Draw the volume slice again using slightly offset texture coordinates [S2].
3.  **Gradient Approximation:** The difference between the two passes effectively computes the component of the gradient along the light vector [S2]. This difference is used to modulate the diffuse lighting intensity.
4.  **Lighting Model:** Apply an approximation of the diffuse component of the Phong shading model using the computed gradient component [S1].

**Parameters:**
*   Light direction: User-adjustable [S2].

**Edge Cases and Pitfalls:**
*   **Bricking Artifacts:** The technique produces visible seams for bricked shapes along the borders of the bricks [S1].
*   **Incorrect Lighting:** The shader does not generate physically correct lighting; it creates a visual effect via a system-dependent technique [S1].
*   **Recommendation:** Use [Voxel Shading Pipeline](3898-voxel-shading-pipeline.md) (specifically `vzTMGradientShader`) for correct volumetric lighting if bricking artifacts are a concern [S1].

## Operating characteristics
*   **Performance:** Efficient as it avoids per-texel gradient computation [S1].
*   **Quality:** Provides higher image quality and better depth cues than unshaded rendering, but lacks physical accuracy [S1].
*   **Limitations:** Not suitable for applications requiring correct volumetric lighting or where bricking seams are unacceptable [S1].

## Background
Tangent Space Shading was introduced by Brian Cabral in SIGGRAPH 1997 [S2]. It was implemented in the SGI OpenGL Volumizer library as `vzTangentSpaceShader` [S1].

## Sources
