# Custom Pass Rendering

Also called: DemoRenderer pipeline, Multi-pass Octree Rendering Pipeline, NanoVolume Custom Pass Rendering, SSVDAG rendering passes

## Overview
Custom Pass Rendering is a multi-pass, object-order rendering architecture that decomposes volumetric visualization into a sequence of specialized GPU compute and fragment shader stages. Unlike single-pass ray marchers, it separates geometric reconstruction (depth/position/normal estimation) from shading and post-processing effects (SSAO, shadows, material lookup) [S1, S2, unofficial]. It is distinct from [DirectXSplat Renderer API](8504-directxsplat-renderer-api.md) by relying on explicit frame buffer object (FBO) attachments for intermediate data rather than a unified API abstraction, and from [Time-Sliced Pipelined Architecture](1435-time-sliced-pipelined-architecture.md) by executing all passes synchronously per frame rather than asynchronously across frames [S1, S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Volumetric data: Stored in a GPU buffer (e.g., `ComputeBuffer` or `usamplerBuffer`) representing an octree or sparse voxel grid [S1, S2, unofficial].
*   Camera parameters: View matrix, projection matrix, inverse matrices, camera position, screen resolution (`screenRes`), and field of view (FOV) [S1, S2, unofficial].
*   Scene bounds: `sceneBBoxMin`, `sceneBBoxMax`, `sceneCenter`, `rootHalfSide` [S1, unofficial].
*   Material data: Optional material octree encoded in a texture buffer (`matOctree`) [S1, unofficial].

**Outputs:**
*   Final image: A color buffer with alpha blending [S2, unofficial].
*   Intermediate buffers: Depth, Hit Position, Hit Normal, Material ID, Shadow map, Ambient Occlusion (AO), and Blurred AO [S1, unofficial].

### Data Structures
The pipeline utilizes a series of Frame Buffer Objects (FBOs) with specific attachments [S1, unofficial]:

*   **FBO[0] (Primary Rays/Depth):**
    *   Attachment 0: `_glTexDepth` (`GL_TEXTURE_2D`, `GL_RGBA32F`, screen resolution, `GL_LINEAR`). Stores depth (`x`), derivative info (`y`), and iteration count (`z`) [S1, unofficial].
*   **FBO[1] (Hit Pos/Norm):**
    *   Attachment 0: `_glTexHitPos` (`GL_TEXTURE_2D`, `GL_RGB32F`, screen resolution, `GL_NEAREST`). Stores world-space hit position [S1, unofficial].
    *   Attachment 1: `_glTexHitNorm` (`GL_TEXTURE_2D`, `GL_RGB32F`, screen resolution, `GL_NEAREST`). Stores world-space hit normal [S1, unofficial].
*   **FBO[2] (Materials, optional):**
    *   Attachment 0: `_glTexHitMaterial` (`GL_TEXTURE_2D`, `GL_R8UI`, screen resolution, `GL_NEAREST`). Stores material ID [S1, unofficial].
*   **FBO[3] (Normal Filtering):**
    *   Attachment 0: `_glTexHitNormFiltered` (`GL_TEXTURE_2D`, `GL_RGB32F`, screen resolution, `GL_NEAREST`) [S1, unofficial].
*   **FBO[4] (Shadows):**
    *   Attachment 0: `_glTexShadows` (`GL_TEXTURE_2D`, `GL_R32F`, screen resolution, `GL_NEAREST`) [S1, unofficial].
*   **FBO[5] (AO):**
    *   Attachment 0: `_glTexAO` (`GL_TEXTURE_2D`, `GL_R32F`, screen resolution, `GL_NEAREST`) [S1, unofficial].
*   **FBO[6] (AO Blurred):**
    *   Attachment 0: `_glTexAOBlurred` (`GL_TEXTURE_2D`, `GL_R32F`, screen resolution, `GL_NEAREST`) [S1, unofficial].

### Numbered Steps

1.  **Initialization:** Create `NUM_PASSES` `RendererMonitor` instances [S1, unofficial]. Load volumetric data into a GPU buffer (e.g., `ComputeBuffer`) [S2, unofficial].
2.  **Pass 1: Position and Normal Computation:**
    *   Compile a fragment shader (`SHADER_POS_NORM_FILE`) with `#pragma target 5.0` [S2, unofficial].
    *   Set uniforms: `screenRes` (vec2), `depth` (int, `TEX_NUM_DEPTH`), `projectionFactor` (float) [S1, unofficial].
    *   If materials are enabled, compile a material lookup pass (`SHADER_MATERIAL_FILE`) with uniforms: `matOctree` (int, `TEX_NUM_MATOCTREE`), `screenRes`, `depth`, `matOctreehalfSide`, `matOctreeCenter`, `matOctreeLevels` [S1, unofficial]. Bind the material octree data to `TEX_NUM_MATOCTREE` [S1, unofficial].
    *   Execute the pass. For each pixel, compute the camera ray using screen coordinates and inverse projection matrix [S1, unofficial].
    *   Sample depth from `_glTexDepth`. If depth is valid, compute world-space hit position: $\mathbf{p} = \mathbf{o} + t \cdot \mathbf{d}$, where $\mathbf{o}$ is camera origin, $\mathbf{d}$ is ray direction, and $t$ is depth [S1, unofficial].
    *   Compute normal using finite differences or geometric properties. The derivative factor is calculated as:
        $$ \text{dif} = \max(0.001, 2.0 \cdot \frac{y_{\text{depth}} \cdot \text{projectionFactor}}{x_{\text{depth}}}) $$
        where $x_{\text{depth}}$ and $y_{\text{depth}}$ are components of the depth texture texel [S1, unofficial].
    *   Apply smoothing if $\text{dif} > 5.0$:
        $$ \text{dif}_{\text{smooth}} = \text{mix}(\text{dif}, 1.0, \min(\frac{\text{dif}-5.0}{20.0-5.0}, 1.0)) $$
        [S1, unofficial].
    *   Calculate normal $\mathbf{n}$ based on $\text{dif}$ and ray direction [S1, unofficial]. Ensure normal points towards camera: if $\mathbf{n} \cdot \mathbf{camDir} > 0$, negate $\mathbf{n}$ [S1, unofficial].
    *   Write position to `_glTexHitPos` and normal to `_glTexHitNorm` [S1, unofficial].
3.  **Pass 2: Normal Filtering:**
    *   Compile blur shader (`SHADER_NORMAL_FILTER_FILE`) [S1, unofficial].
    *   Set uniform: `inputTex` (int, `TEX_NUM_HITNORM`) [S1, unofficial].
    *   Define kernel size `MSIZE` (default 10) [S1, unofficial].
    *   Apply bilateral or Gaussian filtering to `_glTexHitNorm` using parameter `_sigma` (default 4.0) or `_bsigma` (default 0.4) [S1, unofficial].
    *   Write result to `_glTexHitNormFiltered` [S1, unofficial].
4.  **Pass 3: SSAO (Screen Space Ambient Occlusion):**
    *   Compile SSAO shader (`SHADER_SSAO_FILE`) [S1, unofficial].
    *   Set uniforms: `hitPosTex` (`TEX_NUM_HITPOS`), `hitNormTex` (`TEX_NUM_HITNORMFILTERED`), `depthTex` (`TEX_NUM_DEPTH`), `noiseTex` (`TEX_NUM_AONOISE`) [S1, unofficial].
    *   Generate `NUM_SAMPLES` (default 32) 3D sample vectors using `generateSSAOSamples` and upload to `samples` uniform [S1, unofficial].
    *   Generate a noise texture of size `_ssaoNoiseTexSide` (default 64) x 64 using `generateNoiseTex` and bind to `TEX_NUM_AONOISE` [S1, unofficial].
    *   For each pixel, retrieve hit position $\mathbf{p}$ and normal $\mathbf{n}$ [S1, unofficial].
    *   Construct TBN matrix using random tangent vector from noise texture [S1, unofficial].
    *   For each sample $i$:
        *   Compute sample position: $\mathbf{s}_i = \mathbf{p} + (\mathbf{T}\mathbf{s}_i^{\text{local}} + \mathbf{B}\mathbf{s}_i^{\text{local}} + \mathbf{N}\mathbf{s}_i^{\text{local}}) \cdot \text{radius}$ [S1, unofficial].
        *   Project $\mathbf{s}_i$ to clip space and then to screen space [S1, unofficial].
        *   Sample depth at projected coordinates [S1, unofficial].
        *   Calculate occlusion contribution based on depth difference and range check [S1, unofficial].
    *   Compute final AO value: $V = 1.0 - \frac{\sum \text{occlusion}}{\text{count}}$ [S1, unofficial].
    *   Write to `_glTexAO` [S1, unofficial].
5.  **Pass 4: AO Blur:**
    *   Compile blur shader (`SHADER_BLUR_FILE`) [S1, unofficial].
    *   Set uniform: `inputTex` (int, `TEX_NUM_AO`) [S1, unofficial].
    *   Apply Gaussian blur to `_glTexAO` [S1, unofficial].
    *   Write result to `_glTexAOBlurred` [S1, unofficial].
6.  **Pass 5: Shading:**
    *   Compile shading shader (`SHADER_SHADING_FILE`) [S1, unofficial].
    *   Set uniforms: `screenRes`, `hitPosTex` (`TEX_NUM_HITPOS`), `hitNormTex` (`TEX_NUM_HITNORMFILTERED`), `shadowsTex` (`TEX_NUM_SHADOW`), `aoTex` (`TEX_NUM_AOBLURRED`), `depthTex` (`TEX_NUM_DEPTH`), `materialTex` (`TEX_NUM_MATERIAL`) [S1, unofficial].
    *   If materials are used, define `USE_MATERIALS` and `NUM_MATERIALS`, and upload material colors to `materials` uniform [S1, unofficial].
    *   For each pixel:
        *   Retrieve position, normal, shadow, and AO values [S1, unofficial].
        *   If material lookup is enabled, fetch material ID from `_glTexHitMaterial` and retrieve diffuse color from `materials` array [S1, unofficial].
        *   Compute lighting model (diffuse, specular, etc.) using light position, normal, and view direction [S1, unofficial].
        *   Apply AO and shadow factors [S1, unofficial].
        *   Output final color [S1, unofficial].
7.  **Volume Rendering (NanoVolume Pass):**
    *   In Unity implementation, include `NanoVolumePass.hlsl` [S2, unofficial].
    *   Call `NanoVolumePass` with camera position and negated view direction [S2, unofficial].
    *   Return color with alpha blending: `float4(color.rgb, color.a)` [S2, unofficial].

### Formulas
*   **Projection Factor:**
    $$ \text{projectionFactor} = \frac{\text{inv\_2tan\_half\_fovy}}{\text{screen\_tolerance}} $$
    where $\text{inv\_2tan\_half\_fovy} = \frac{1.0}{2.0 \cdot \tan(0.5 \cdot \text{fovy})}$ and $\text{screen\_tolerance} = \frac{1.0}{\text{screenRes}[1]}$ [S1, unofficial].
*   **Camera Ray Direction:**
    $$ \mathbf{rd} = \text{normalize}(\mathbf{e}' - \mathbf{o}') $$
    where $\mathbf{o}' = \text{viewMatInv} \cdot [0,0,0,1]^T$ and $\mathbf{e}' = \text{viewMatInv} \cdot [\mathbf{rd}_{\text{clip}}, 1]^T$ [S1, unofficial].
*   **Normal Derivative Factor:**
    $$ \text{dif} = \max(0.001, 2.0 \cdot \frac{y_{\text{depth}} \cdot \text{projectionFactor}}{x_{\text{depth}}}) $$
    [S1, unofficial].

### Parameters
*   `_sigma`: 4.0 (Normal filtering/SSAO smoothing) [S1, unofficial].
*   `_bsigma`: 0.4 (Bilateral filtering/SSAO smoothing) [S1, unofficial].
*   `_msize`: 10 (Normal filtering kernel size) [S1, unofficial].
*   `_ssaoRadius`: `eo->getHalfSide(eo->getNLevels()) * 100.0` (SSAO sample radius) [S1, unofficial].
*   `_ssaoNSamples`: 32 (SSAO sample count) [S1, unofficial].
*   `_ssaoNoiseTexSide`: 64 (SSAO noise texture side length) [S1, unofficial].
*   `_renderMode`: 0 (Render mode selector) [S1, unofficial].
*   `_useDif`: true (Enable diffuse shading) [S1, unofficial].
*   `_filterNormals`: true (Enable normal filtering) [S1, unofficial].
*   `_doSSAO`: true (Enable SSAO) [S1, unofficial].
*   `_doShadows`: true (Enable shadows) [S1, unofficial].
*   `_useMaterials`: false (Enable material lookup) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Background Pixels:** If depth value is $\le 0$ or hit position is $(0,0,0)$, discard pixel or render sky color [S1, unofficial].
*   **Normal Orientation:** Normals must be flipped if they point away from the camera ($\mathbf{n} \cdot \mathbf{camDir} > 0$) [S1, unofficial].
*   **Precision:** Use `GL_RGBA32F` for depth and position buffers to avoid precision loss in large scenes [S1, unofficial].
*   **Material IDs:** Handle invalid material IDs (e.g., 255) with debug colors [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of pixels and $P$ is the number of passes (typically 5-7) [S1, S2, unofficial]. Each pass is $O(N)$ for screen-space operations, except volume traversal which depends on tree depth [S1, unofficial].
*   **Memory Complexity:** $O(N \cdot B)$, where $B$ is the number of bytes per pixel per buffer. With 7 FBOs and 32-bit floats, this is approximately $7 \times 4 \times W \times H$ bytes [S1, unofficial].
*   **Performance:** Real-time on modern GPUs for resolutions up to 4K, depending on scene complexity and SSAO sample count [S1, S2, unofficial].
*   **Strengths:** Modular, allows independent tuning of effects (SSAO, shadows, materials), leverages hardware rasterization for depth/position [S1, S2, unofficial].
*   **Limitations:** High memory bandwidth usage due to multiple FBOs, not suitable for extremely large scenes without tiling, SSAO quality depends on sample count [S1, S2, unofficial].

## Background
Custom Pass Rendering originated from the SSVDAG (Sparse Sparse Voxel DAG) project by ajaspe [S1, unofficial] and was adapted for Unity by andersblomqvist [S2, unofficial]. It is used in real-time volumetric visualization tools, particularly for scientific data and game engines requiring high-quality volumetric effects [S1, S2, unofficial].

## Sources
[S1] ajaspe/ssvdag, https://github.com/ajaspe/ssvdag, fetched 2026-10-24 (unofficial)
[S2] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
