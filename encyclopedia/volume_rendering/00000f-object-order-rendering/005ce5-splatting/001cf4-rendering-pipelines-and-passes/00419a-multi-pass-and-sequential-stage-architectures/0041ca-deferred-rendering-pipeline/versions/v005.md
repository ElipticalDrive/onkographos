# Deferred Rendering Pipeline

Also called: 8-Pass Deferred Rendering Pipeline

## Overview
A multi-pass deferred rendering sequence that decouples geometry processing from lighting and post-processing by storing intermediate scene attributes in separate framebuffer objects (FBOs) [S1, unofficial]. It differs from forward rendering by deferring shading until after depth and position are known, and from fragment-list approaches by relying on screen-space quad passes for filtering and lighting rather than explicit geometry lists.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene geometry (OctreeDDARenderer), camera matrices (`viewMat`, `projMat`, inverses), light position, material data, and configuration flags (`_useMaterials`, `_filterNormals`, `_doShadows`, `_doSSAO`).
*   **Outputs:** Final shaded image in the default framebuffer.

**Data Structures:**
The pipeline uses a sequence of 8 Framebuffer Objects (`_glFBO[0]` to `_glFBO[6]` and the default framebuffer). Textures are bound to specific texture units (`TEX_NUM_*`) during `resetState()`:
1.  `_glTexDepth` (Depth buffer)
2.  `_glTexHitPos` (Position buffer)
3.  `_glTexHitNorm` (Normal buffer)
4.  `_glTexHitMaterial` (Material buffer)
5.  `_glTexHitNormFiltered` (Filtered Normal buffer)
6.  `_glTexShadows` (Shadow map buffer)
7.  `_glTexAO` (Ambient Occlusion buffer)
8.  `_glTexAOBlurred` (Blurred AO buffer)
9.  `_glTexAONoise` (Noise texture for AO)
10. `_glTexTBOMaterials` (TBO Materials)

**Numbered Steps:**

1.  **Depth Pass:** Bind `_glFBO[0]`. Clear buffers. Set `OctreeDDARenderer` mode to `DEPTH`. Draw scene geometry.
2.  **Position/Normal Reconstruction:** Bind `_glFBO[1]`. Clear buffers. Use `_programPosNorm`. Set uniforms: `viewMatInv`, `projMatInv`, `useDif`. Draw a full-screen quad.
3.  **Material Pass (Conditional):** If `_useMaterials` is true: Bind `_glFBO[2]`. Clear buffers. Use `_programMaterial`. Set uniforms. Draw a full-screen quad.
4.  **Normal Filtering (Conditional):** If `_filterNormals` is true: Bind `_glFBO[3]`. Clear buffers. Use `_programNormFilter`. Set uniforms: `sigma`, `bsigma`. Draw a full-screen quad.
5.  **Shadow Pass (Conditional):** If `_doShadows` is true: Bind `_glFBO[4]`. Clear buffers. Set `lightPos`. Set `OctreeDDARenderer` mode to `SHADOW`. Draw scene geometry. Else: Clear `_glFBO[4]` to white (1.0), representing a fully lit state.
6.  **SSAO Pass (Conditional):** If `_doSSAO` is true: Bind `_glFBO[5]`. Clear buffers. Use `_programSSAO`. Set uniforms: `projMat`, `viewMat`, `camPos`, `radius`. Draw a full-screen quad.
7.  **AO Blurring:** Bind `_glFBO[6]`. Clear buffers. Use `_programBlur`. Draw a full-screen quad.
8.  **Final Shading:** Bind default framebuffer. Clear buffers. Use `_programShading`. Set uniforms: `isYUp`, `camPos`, `projMatInv`, `viewMatInv`, `lightPos`, `renderMode`, `useMaterials`. Draw a full-screen quad.

**Pitfalls:**
*   Pass 5 (Shadows) clears to white (1.0) when disabled, which acts as a default fully lit state. This must be handled correctly in the shading pass to avoid darkening the scene if shadow logic assumes a 0.0 default [S1, unofficial].
*   Pass 6 (SSAO) is often wrapped in `#if 0/#else` in implementations, indicating the original octree-based AO path is disabled in favor of the screen-space quad pass [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
This specific 8-pass implementation is part of the `ssvdag` project, an unofficial software renderer for sparse voxel DAGs [S1, unofficial]. It adapts deferred shading techniques to handle large-scale voxel data structures efficiently by separating traversal (depth/shadow) from screen-space post-processing.

## Sources
[S1] ajaspe/ssvdag, https://github.com/ajaspe/ssvdag, fetched 2026-10-24 (unofficial)
