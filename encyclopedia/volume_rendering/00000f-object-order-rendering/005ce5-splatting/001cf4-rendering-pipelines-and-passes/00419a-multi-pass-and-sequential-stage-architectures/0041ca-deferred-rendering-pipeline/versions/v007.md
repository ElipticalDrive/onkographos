# Deferred Rendering Pipeline

Also called: 8-Pass Deferred Rendering Pipeline

## Overview
The Deferred Rendering Pipeline is a multi-pass sequence that decouples geometry processing from shading by storing per-pixel attributes in intermediate framebuffer objects (FBOs) before computing the final image. It solves the problem of efficiently applying complex lighting and post-processing effects (such as SSAO and shadows) to volumetric data by separating the traversal/rasterization stages from the evaluation stages. It differs from the [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md) by explicitly managing a fixed sequence of distinct render passes with intermediate storage, and from the [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md) by relying on screen-space quad passes for post-processing rather than explicit fragment lists.

## How it works
**Inputs and Outputs:**
*   **Input:** Scene geometry (typically an octree or voxel structure), camera matrices (`viewMat`, `projMat`, `viewMatInv`, `projMatInv`), light positions, and configuration flags (e.g., `_useMaterials`, `_filterNormals`, `_doShadows`, `_doSSAO`).
*   **Output:** A final color image rendered to the default framebuffer.

**Data Structures:**
The pipeline uses a sequence of Framebuffer Objects (`_glFBO[0]` to `_glFBO[6]`) and associated textures. The textures are bound to specific texture units (`TEX_NUM_*`) during state reset.
*   `_glFBO[0]`: Depth buffer (`_glTexDepth`).
*   `_glFBO[1]`: Position and Normal buffer (`_glTexHitPos`, `_glTexHitNorm`).
*   `_glFBO[2]`: Material buffer (`_glTexHitMaterial`).
*   `_glFBO[3]`: Filtered Normal buffer (`_glTexHitNormFiltered`).
*   `_glFBO[4]`: Shadow map buffer (`_glTexShadows`).
*   `_glFBO[5]`: Ambient Occlusion buffer (`_glTexAO`).
*   `_glFBO[6]`: Blurred AO buffer (`_glTexAOBlurred`).
*   Additional textures: `_glTexAONoise`, `_glTexTBOMaterials`.

**Numbered Steps:**
1.  **Pass 1 (Depth):** Bind `_glFBO[0]`. Clear the buffer. Set the renderer mode to `DEPTH`. Draw the scene geometry (e.g., using `OctreeDDARenderer`). This computes the depth of the first hit for each pixel.
2.  **Pass 2 (Position/Normal):** Bind `_glFBO[1]`. Clear the buffer. Use shader `_programPosNorm`. Set uniforms `viewMatInv`, `projMatInv`, and `useDif`. Draw a full-screen quad. This reconstructs world-space position and normal from the depth buffer.
3.  **Pass 3 (Materials):** If `_useMaterials` is true: Bind `_glFBO[2]`. Clear the buffer. Use shader `_programMaterial`. Set uniforms. Draw a full-screen quad. This fetches material properties for the hit fragments.
4.  **Pass 4 (Normal Filtering):** If `_filterNormals` is true: Bind `_glFBO[3]`. Clear the buffer. Use shader `_programNormFilter`. Set uniforms `sigma` and `bsigma`. Draw a full-screen quad. This smooths the normals to reduce noise.
5.  **Pass 5 (Shadows):** If `_doShadows` is true: Bind `_glFBO[4]`. Clear the buffer. Set `lightPos`. Set renderer mode to `SHADOW`. Draw the scene geometry. Else, clear `_glFBO[4]` to white (1.0), representing a fully lit state.
6.  **Pass 6 (SSAO):** If `_doSSAO` is true: Bind `_glFBO[5]`. Clear the buffer. Use shader `_programSSAO`. Set uniforms `projMat`, `viewMat`, `camPos`, and `radius`. Draw a full-screen quad. This computes screen-space ambient occlusion.
7.  **Pass 7 (AO Blur):** Bind `_glFBO[6]`. Clear the buffer. Use shader `_programBlur`. Draw a full-screen quad. This blurs the AO buffer to reduce noise and artifacts.
8.  **Pass 8 (Final Shading):** Bind the default framebuffer. Clear the buffer. Use shader `_programShading`. Set uniforms `isYUp`, `camPos`, `projMatInv`, `viewMatInv`, `lightPos`, `renderMode`, and `useMaterials`. Draw a full-screen quad. This combines all accumulated buffers (depth, position, normal, material, shadow, AO) to compute the final pixel color.

**Pitfalls:**
*   Pass 5 clears to white when shadows are disabled, which acts as a default fully lit state. If the shadow logic is not handled correctly, this may lead to incorrect lighting if the shader assumes a different default.
*   Pass 6 (SSAO) is often wrapped in conditional compilation (`#if 0/#else`) in reference implementations, indicating that the original octree-based AO path is disabled in favor of the screen-space quad pass. Implementers must ensure the correct path is active.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of passes (8) and the screen resolution. Each pass is typically a full-screen quad or a geometry draw call.
*   **Memory Complexity:** Proportional to the screen resolution times the number of intermediate buffers (8 FBOs). High-resolution scenes require significant VRAM for the depth, position, normal, material, shadow, and AO textures.
*   **Strengths:** Decouples geometry complexity from shading complexity. Allows for efficient application of screen-space effects like SSAO and blurring.
*   **Limitations:** Requires multiple render passes, increasing overhead. Depth buffer precision issues can affect position reconstruction. Not suitable for transparent objects without additional handling.
*   **When to use:** When complex lighting and post-processing effects are needed for volumetric or opaque geometry.
*   **When not to use:** For simple scenes where the overhead of multiple passes is unjustified, or for highly transparent scenes requiring order-independent transparency.

## Background
The deferred rendering pipeline is a well-established technique in computer graphics, originally developed to handle complex lighting scenarios efficiently. The specific 8-pass implementation described here is found in the `ssvdag` repository [S1, unofficial], which implements a sparse voxel octree renderer. It builds on the principles of deferred shading, adapting them for volumetric data traversal and rendering.

## Sources
[S1] ajaspe/ssvdag, https://github.com/ajaspe/ssvdag, fetched 2026-10-24 (unofficial)
