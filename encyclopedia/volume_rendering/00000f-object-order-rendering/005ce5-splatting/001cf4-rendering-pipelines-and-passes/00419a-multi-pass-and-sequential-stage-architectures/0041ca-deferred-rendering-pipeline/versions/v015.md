# Deferred Rendering Pipeline
Also called: 8-Pass Deferred Rendering Pipeline
## Overview
A multi-pass deferred rendering sequence that computes depth, position/normal, materials, filtered normals, shadows, SSAO, blurred AO, and final shading in separate framebuffer objects [S1, unofficial]. It separates geometry processing from lighting calculations, allowing complex lighting effects to be applied per-pixel without re-rasterizing geometry [S1, unofficial]. It is distinct from the [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md) by deferring shading to a post-process stage, and from the [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md) by using fixed-screen-space framebuffers rather than dynamic fragment lists.

## How it works
**Inputs and Outputs:**
- Input: Volume data (accessed via OctreeDDARenderer), camera matrices (view, projection, inverse), light position, material parameters.
- Output: Final shaded image in the default framebuffer.

**Data Structures:**
The method uses a sequence of Framebuffer Objects (FBOs) and associated textures. The following textures are bound to texture units `TEX_NUM_*` during `resetState()` [S1, unofficial]:
1. `_glTexDepth`: Depth buffer.
2. `_glTexHitPos`: Position buffer.
3. `_glTexHitNorm`: Normal buffer.
4. `_glTexHitMaterial`: Material buffer.
5. `_glTexHitNormFiltered`: Filtered normal buffer.
6. `_glTexShadows`: Shadow map buffer.
7. `_glTexAO`: Ambient Occlusion buffer.
8. `_glTexAOBlurred`: Blurred AO buffer.
9. `_glTexAONoise`: Noise texture for AO.
10. `_glTexTBOMaterials`: Texture for TBO materials.

Framebuffers `_glFBO[0]` through `_glFBO[6]` are used sequentially [S1, unofficial].

**Steps:**
1. **Depth Pass:** Bind `_glFBO[0]`. Clear. Set `OctreeDDARenderer` mode to `DEPTH`. Draw the volume geometry.
2. **Position/Normal Pass:** Bind `_glFBO[1]`. Clear. Use shader `_programPosNorm`. Set uniforms `viewMatInv`, `projMatInv`, `useDif`. Draw a full-screen quad.
3. **Material Pass:** If `_useMaterials` is true: Bind `_glFBO[2]`. Clear. Use shader `_programMaterial`. Set uniforms. Draw a full-screen quad.
4. **Normal Filtering Pass:** If `_filterNormals` is true: Bind `_glFBO[3]`. Clear. Use shader `_programNormFilter`. Set uniforms `sigma`, `bsigma`. Draw a full-screen quad.
5. **Shadow Pass:** If `_doShadows` is true: Bind `_glFBO[4]`. Clear. Set `lightPos`. Set `OctreeDDARenderer` mode to `SHADOW`. Draw the volume geometry. Else, clear `_glFBO[4]` to white (1.0) [S1, unofficial].
6. **SSAO Pass:** If `_doSSAO` is true: Bind `_glFBO[5]`. Clear. Use shader `_programSSAO`. Set uniforms `projMat`, `viewMat`, `camPos`, `radius`. Draw a full-screen quad.
7. **AO Blur Pass:** Bind `_glFBO[6]`. Clear. Use shader `_programBlur`. Draw a full-screen quad.
8. **Final Shading Pass:** Bind the default framebuffer. Clear. Use shader `_programShading`. Set uniforms `isYUp`, `camPos`, `projMatInv`, `viewMatInv`, `lightPos`, `renderMode`, `useMaterials`. Draw a full-screen quad.

**Pitfalls:**
- Pass 5 (Shadows) clears to white (1.0) when disabled, which acts as a default fully lit state [S1, unofficial].
- Pass 6 (SSAO) is wrapped in `#if 0/#else` in the source, indicating the original octree-based AO path is disabled in favor of the screen-space quad pass [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
This specific 8-pass implementation is part of the `ssvdag` project, which uses an octree-based DDA renderer for volume visualization [S1, unofficial].

## Sources
[S1] ajaspe/ssvdag, https://github.com/ajaspe/ssvdag, fetched 2026-10-24 (unofficial)
