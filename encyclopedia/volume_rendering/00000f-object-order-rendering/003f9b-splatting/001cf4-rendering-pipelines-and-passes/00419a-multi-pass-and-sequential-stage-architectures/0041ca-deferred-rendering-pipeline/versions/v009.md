# Deferred Rendering Pipeline
Also called: 8-Pass Deferred Rendering Pipeline
## Overview
A multi-pass deferred rendering sequence that decouples geometry traversal, shading, and post-processing by storing intermediate results in separate framebuffer objects (FBOs) [S1, unofficial]. It computes depth, position/normal, materials, filtered normals, shadows, screen-space ambient occlusion (SSAO), blurred AO, and final shading in distinct stages [S1, unofficial]. It sets itself apart from object-order pipelines by deferring lighting calculations until after all geometry is processed, allowing for efficient handling of complex lighting and post-processing effects without re-traversing the volume structure for each light source [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Inputs:** View matrix, projection matrix, inverse matrices, camera position, light position, octree data (for depth/shadow passes), and configuration flags (`_useMaterials`, `_filterNormals`, `_doShadows`, `_doSSAO`) [S1, unofficial].
*   **Outputs:** Final shaded image rendered to the default framebuffer [S1, unofficial].

**Data Structures:**
The pipeline uses a sequence of 8 Framebuffer Objects (`_glFBO[0]` through `_glFBO[6]` and the default framebuffer) and associated textures [S1, unofficial]:
*   `_glFBO[0]`: Depth buffer (`_glTexDepth`) [S1, unofficial].
*   `_glFBO[1]`: Position and Normal buffer (`_glTexHitPos`, `_glTexHitNorm`) [S1, unofficial].
*   `_glFBO[2]`: Material buffer (`_glTexHitMaterial`) [S1, unofficial].
*   `_glFBO[3]`: Filtered Normal buffer (`_glTexHitNormFiltered`) [S1, unofficial].
*   `_glFBO[4]`: Shadow buffer (`_glTexShadows`) [S1, unofficial].
*   `_glFBO[5]`: AO buffer (`_glTexAO`) [S1, unofficial].
*   `_glFBO[6]`: Blurred AO buffer (`_glTexAOBlurred`) [S1, unofficial].
*   Additional textures: `_glTexAONoise`, `_glTexTBOMaterials` bound to texture units `TEX_NUM_*` during reset [S1, unofficial].

**Numbered Steps:**
1.  **Pass 1 (Depth):** Bind `_glFBO[0]`, clear buffers, set the `OctreeDDARenderer` mode to `DEPTH`, and draw the scene [S1, unofficial].
2.  **Pass 2 (Position/Normal):** Bind `_glFBO[1]`, clear buffers, use `_programPosNorm`, set uniforms `viewMatInv`, `projMatInv`, and `useDif`, and draw a full-screen quad [S1, unofficial].
3.  **Pass 3 (Materials):** If `_useMaterials` is true, bind `_glFBO[2]`, clear buffers, use `_programMaterial`, set uniforms, and draw a full-screen quad [S1, unofficial].
4.  **Pass 4 (Normal Filtering):** If `_filterNormals` is true, bind `_glFBO[3]`, clear buffers, use `_programNormFilter`, set uniforms `sigma` and `bsigma`, and draw a full-screen quad [S1, unofficial].
5.  **Pass 5 (Shadows):** If `_doShadows` is true, bind `_glFBO[4]`, clear buffers, set `lightPos`, set `OctreeDDARenderer` mode to `SHADOW`, and draw the scene. Else, clear `_glFBO[4]` to white (1.0) [S1, unofficial].
6.  **Pass 6 (SSAO):** If `_doSSAO` is true, bind `_glFBO[5]`, clear buffers, use `_programSSAO`, set uniforms `projMat`, `viewMat`, `camPos`, and `radius`, and draw a full-screen quad [S1, unofficial].
7.  **Pass 7 (AO Blur):** Bind `_glFBO[6]`, clear buffers, use `_programBlur`, and draw a full-screen quad [S1, unofficial].
8.  **Pass 8 (Final Shading):** Bind the default framebuffer, clear buffers, use `_programShading`, set uniforms `isYUp`, `camPos`, `projMatInv`, `viewMatInv`, `lightPos`, `renderMode`, and `useMaterials`, and draw a full-screen quad [S1, unofficial].

**Pitfalls:**
*   Pass 5 clears the shadow buffer to white (1.0) when shadows are disabled, which acts as a default fully lit state [S1, unofficial].
*   Pass 6 (SSAO) may be wrapped in `#if 0/#else` in some implementations, indicating the original octree-based AO path is disabled in favor of the screen-space quad pass [S1, unofficial].

## Operating characteristics
Not given in the sources.
## Background
Not given in the sources.
## Sources
[S1] ajaspe/ssvdag, https://github.com/ajaspe/ssvdag, fetched 2026-10-24 (unofficial)
