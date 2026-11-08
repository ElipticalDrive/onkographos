# Deferred Rendering Pipeline
Also called: 8-Pass Deferred Rendering Pipeline
## Overview
A multi-pass deferred rendering sequence that computes depth, position/normal, materials, filtered normals, shadows, SSAO, blurred AO, and final shading in separate framebuffer objects [S1, unofficial]. It separates geometric projection and shading logic, allowing independent optimization of visual appearance and geometry traversal [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene geometry (OctreeDDARenderer), camera matrices (`viewMat`, `projMat`, `viewMatInv`, `projMatInv`), light position, material data, and configuration flags (`_useMaterials`, `_filterNormals`, `_doShadows`, `_doSSAO`).
*   **Outputs:** Final shaded image in the default framebuffer.

**Data Structures:**
*   **Framebuffers:** `_glFBO[0]` through `_glFBO[6]` are used sequentially [S1, unofficial].
*   **Textures:** Bound to texture units `TEX_NUM_*` during `resetState()` [S1, unofficial]:
    *   `_glTexDepth`: Depth buffer.
    *   `_glTexHitPos`: Hit position.
    *   `_glTexHitNorm`: Hit normal.
    *   `_glTexHitMaterial`: Material ID.
    *   `_glTexHitNormFiltered`: Filtered normal.
    *   `_glTexShadows`: Shadow map.
    *   `_glTexAO`: Ambient Occlusion.
    *   `_glTexAOBlurred`: Blurred AO.
    *   `_glTexAONoise`: Noise texture for AO.
    *   `_glTexTBOMaterials`: Texture-based material data.

**Steps:**
1.  **Depth Pass:** Bind `_glFBO[0]`, clear, set `OctreeDDARenderer` mode to `DEPTH`, draw scene [S1, unofficial].
2.  **Position/Normal Pass:** Bind `_glFBO[1]`, clear, use `_programPosNorm`, set uniforms `viewMatInv`, `projMatInv`, `useDif`, draw quad [S1, unofficial].
3.  **Material Pass (Conditional):** If `_useMaterials` is true, bind `_glFBO[2]`, clear, use `_programMaterial`, set uniforms, draw quad [S1, unofficial].
4.  **Normal Filtering Pass (Conditional):** If `_filterNormals` is true, bind `_glFBO[3]`, clear, use `_programNormFilter`, set uniforms `sigma`, `bsigma`, draw quad [S1, unofficial].
5.  **Shadow Pass (Conditional):** If `_doShadows` is true, bind `_glFBO[4]`, clear, set `lightPos`, set `OctreeDDARenderer` mode to `SHADOW`, draw scene. Else, clear `_glFBO[4]` to white (1.0) [S1, unofficial].
6.  **SSAO Pass (Conditional):** If `_doSSAO` is true, bind `_glFBO[5]`, clear, use `_programSSAO`, set uniforms `projMat`, `viewMat`, `camPos`, `radius`, draw quad [S1, unofficial].
7.  **AO Blur Pass:** Bind `_glFBO[6]`, clear, use `_programBlur`, draw quad [S1, unofficial].
8.  **Final Shading Pass:** Bind default framebuffer, clear, use `_programShading`, set uniforms `isYUp`, `camPos`, `projMatInv`, `viewMatInv`, `lightPos`, `renderMode`, `useMaterials`, draw quad [S1, unofficial].

**Pitfalls:**
*   Pass 5 (Shadows) clears to white (1.0) when disabled, which acts as a default fully lit state [S1, unofficial].
*   Pass 6 (SSAO) is wrapped in `#if 0/#else`, indicating the original octree-based AO path is disabled in favor of the screen-space quad pass [S1, unofficial].

## Operating characteristics
Not given in the sources.
## Background
Origin: Implemented in the `ssvdag` project by ajaspe [S1, unofficial].
## Sources
[S1] ajaspe/ssvdag, https://github.com/ajaspe/ssvdag, fetched 2026-10-24 (unofficial)
