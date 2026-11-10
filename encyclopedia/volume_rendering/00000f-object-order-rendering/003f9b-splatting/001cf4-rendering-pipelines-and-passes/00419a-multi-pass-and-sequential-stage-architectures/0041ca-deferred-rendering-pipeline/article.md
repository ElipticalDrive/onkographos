# Deferred Rendering Pipeline

Also called: 8-Pass Deferred Rendering Pipeline

## Overview
The Deferred Rendering Pipeline is a multi-pass sequence that decouples geometric traversal and rasterization from final shading by storing intermediate state in Framebuffer Objects (FBOs) [S1, unofficial]. It solves the problem of efficiently applying complex lighting models (shadows, ambient occlusion) to large datasets by reusing screen-space data across passes, avoiding redundant volume traversal [S1, unofficial]. It differs from the [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md) by separating depth/position computation from shading, and from the [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md) by using screen-space buffers rather than explicit fragment lists for inter-pass communication.

## How it works
**Inputs and Outputs:**
*   **Inputs:** View matrix, projection matrix, inverse matrices, camera position, light position, octree data (for depth/shadow passes), and configuration flags (`_useMaterials`, `_filterNormals`, `_doShadows`, `_doSSAO`).
*   **Outputs:** Final shaded image in the default framebuffer.

**Data Structures:**
The pipeline uses 8 Framebuffer Objects (`_glFBO[0]` to `_glFBO[7]`, where `_glFBO[7]` is the default screen). Textures are bound to specific units during `resetState()`:
*   `_glTexDepth`: Bound to `TEX_NUM_DEPTH`.
*   `_glTexHitPos`: Bound to `TEX_NUM_HIT_POS`.
*   `_glTexHitNorm`: Bound to `TEX_NUM_HIT_NORM`.
*   `_glTexHitMaterial`: Bound to `TEX_NUM_HIT_MATERIAL`.
*   `_glTexHitNormFiltered`: Bound to `TEX_NUM_HIT_NORM_FILTERED`.
*   `_glTexShadows`: Bound to `TEX_NUM_SHADOWS`.
*   `_glTexAO`: Bound to `TEX_NUM_AO`.
*   `_glTexAOBlurred`: Bound to `TEX_NUM_AO_BLURRED`.
*   `_glTexAONoise`: Bound to `TEX_NUM_AO_NOISE`.
*   `_glTexTBOMaterials`: Bound to `TEX_NUM_TBO_MATERIALS`.

**Steps:**
1.  **Pass 1 (Depth):** Bind `_glFBO[0]`, clear buffers. Set the renderer mode to `DEPTH`. Draw the volume geometry (e.g., using an OctreeDDARenderer). This populates `_glTexDepth`.
2.  **Pass 2 (Position/Normal):** Bind `_glFBO[1]`, clear buffers. Use shader `_programPosNorm`. Set uniforms `viewMatInv`, `projMatInv`, and `useDif`. Draw a full-screen quad. This reconstructs world-space position and normal from depth, populating `_glTexHitPos` and `_glTexHitNorm`.
3.  **Pass 3 (Materials):** If `_useMaterials` is true: Bind `_glFBO[2]`, clear buffers. Use shader `_programMaterial`. Set uniforms. Draw a full-screen quad. Populates `_glTexHitMaterial`. If false, this step is skipped or buffers are cleared.
4.  **Pass 4 (Normal Filtering):** If `_filterNormals` is true: Bind `_glFBO[3]`, clear buffers. Use shader `_programNormFilter`. Set uniforms `sigma` and `bsigma`. Draw a full-screen quad. Populates `_glTexHitNormFiltered`.
5.  **Pass 5 (Shadows):** If `_doShadows` is true: Bind `_glFBO[4]`, clear buffers. Set `lightPos`. Set renderer mode to `SHADOW`. Draw the volume geometry. Populates `_glTexShadows`. Else: Clear `_glFBO[4]` to white (1.0), representing full illumination.
6.  **Pass 6 (SSAO):** If `_doSSAO` is true: Bind `_glFBO[5]`, clear buffers. Use shader `_programSSAO`. Set uniforms `projMat`, `viewMat`, `camPos`, and `radius`. Draw a full-screen quad. Populates `_glTexAO`. Note: The original octree-based AO path is disabled (`#if 0`), favoring this screen-space quad pass.
7.  **Pass 7 (AO Blur):** Bind `_glFBO[6]`, clear buffers. Use shader `_programBlur`. Draw a full-screen quad. Populates `_glTexAOBlurred`.
8.  **Pass 8 (Final Shading):** Bind the default framebuffer, clear buffers. Use shader `_programShading`. Set uniforms `isYUp`, `camPos`, `projMatInv`, `viewMatInv`, `lightPos`, `renderMode`, and `useMaterials`. Read from all intermediate textures (`_glTexHitPos`, `_glTexHitNormFiltered`, `_glTexHitMaterial`, `_glTexShadows`, `_glTexAOBlurred`). Draw a full-screen quad to produce the final image.

**Pitfalls:**
*   Pass 5 clears to white when shadows are disabled, which acts as a default fully lit state. If shadow logic expects a different default, this may cause incorrect lighting [S1, unofficial].
*   The SSAO pass (Pass 6) replaces an older octree-based implementation. Ensure the shader `_programSSAO` is compatible with the screen-space reconstruction data from Pass 2 [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
This pipeline is implemented in the `ssvdag` project by ajaspe, which uses a DAG4 octree structure for volume rendering [S1, unofficial]. It represents a screen-space deferred shading approach adapted for volume data, leveraging the fixed sequence of passes to handle complex lighting without per-fragment volume traversal in the final pass.

## Sources
[S1] ajaspe/ssvdag, https://github.com/ajaspe/ssvdag, fetched 2026-10-24 (unofficial)
