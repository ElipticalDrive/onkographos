# Custom Pass Rendering

Also called: DemoRenderer pipeline, Multi-pass Octree Rendering Pipeline, NanoVolume Custom Pass Rendering, SSVDAG rendering passes

## Overview
Custom Pass Rendering is a multi-pass, object-order rendering architecture that decouples volumetric traversal and shading from the standard graphics pipeline by executing a sequence of specialized compute or fragment shaders as "custom passes" [S1, S2, unofficial]. It solves the problem of rendering large, sparse volumetric datasets (such as VDB grids) in real-time by leveraging hardware-accelerated rasterization for post-traversal effects (SSAO, shadows, normal filtering) while using GPU-based ray traversal for the primary volume sampling [S1, S2, unofficial]. It differs from pure ray-marching methods by separating the expensive volume traversal into a dedicated pass that outputs hit positions and normals to screen-space buffers, which are then consumed by subsequent passes for lighting and compositing [S1, S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Volumetric data structure (e.g., NanoVDB octree) [S1, S2, unofficial].
*   Camera parameters: View-Projection matrix, View matrix, Camera position, Screen resolution, Field of View (FOV) [S1, S2, unofficial].
*   Scene bounding box and octree metadata (center, half-side, levels) [S1, S2, unofficial].
*   Optional material lookup table [S1, unofficial].

**Outputs:**
*   Final composited color buffer [S1, S2, unofficial].
*   Intermediate screen-space buffers: Depth, Hit Position, Hit Normal, Material ID, Shadow map, Ambient Occlusion (AO) [S1, unofficial].

### Data Structures
The pipeline relies on a set of Frame Buffer Objects (FBOs) or Render Textures, each attached to a specific render target. All textures are sized to the screen resolution ($W \times H$).

1.  **FBO[0] (Primary Rays/Depth):**
    *   Attachment 0: `_glTexDepth` (GL_TEXTURE_2D, GL_RGBA32F, GL_LINEAR filtering) [S1, unofficial].
2.  **FBO[1] (Hit Pos/Norm):**
    *   Attachment 0: `_glTexHitPos` (GL_TEXTURE_2D, GL_RGB32F, GL_NEAREST filtering) [S1, unofficial].
    *   Attachment 1: `_glTexHitNorm` (GL_TEXTURE_2D, GL_RGB32F, GL_NEAREST filtering) [S1, unofficial].
3.  **FBO[2] (Materials, optional):**
    *   Attachment 0: `_glTexHitMaterial` (GL_TEXTURE_2D, GL_R8UI, GL_NEAREST filtering) [S1, unofficial].
4.  **FBO[3] (Normal Filtering):**
    *   Attachment 0: `_glTexHitNormFiltered` (GL_TEXTURE_2D, GL_RGB32F, GL_NEAREST filtering) [S1, unofficial].
5.  **FBO[4] (Shadows):**
    *   Attachment 0: `_glTexShadows` (GL_TEXTURE_2D, GL_R32F, GL_NEAREST filtering) [S1, unofficial].
6.  **FBO[5] (AO):**
    *   Attachment 0: `_glTexAO` (GL_TEXTURE_2D, GL_R32F, GL_NEAREST filtering) [S1, unofficial].
7.  **FBO[6] (AO Blurred):**
    *   Attachment 0: `_glTexAOBlurred` (GL_TEXTURE_2D, GL_R32F, GL_NEAREST filtering) [S1, unofficial].

### Numbered Steps

1.  **Initialization:** Create `NUM_PASSES` RendererMonitor instances [S1, unofficial]. Load the volumetric data (e.g., NanoVDB) into GPU-accessible memory (e.g., `ComputeBuffer` or `GL_TEXTURE_BUFFER`) [S1, S2, unofficial].
2.  **Pass 1: Primary Ray Traversal (Pos/Norm Computation):**
    *   Compile and link a shader (e.g., `SHADER_POS_NORM_FILE`) [S1, unofficial].
    *   Set uniforms: `screenRes` (vec2), `depth` (int, TEX_NUM_DEPTH), `projectionFactor` (float) [S1, unofficial].
    *   Execute a full-screen pass. For each pixel, compute the camera ray direction $\mathbf{r}_d$ from screen coordinates [S1, unofficial]:
        $$ \mathbf{r}_d = \text{normalize}(\text{fromHomog}(\mathbf{P}^{-1} \cdot \mathbf{v}_1) - \text{fromHomog}(\mathbf{P}^{-1} \cdot \mathbf{v}_0)) $$
        where $\mathbf{v}_0, \mathbf{v}_1$ are clip-space rays at $z=0$ and $z=1$ [S1, unofficial].
    *   Perform GPU-based ray traversal (e.g., DAG/SDAG traversal) against the volume octree to find the first hit [S1, unofficial].
    *   If a hit is found, write the hit position $\mathbf{p}_{hit}$ and normal $\mathbf{n}_{hit}$ to `_glTexHitPos` and `_glTexHitNorm` respectively [S1, unofficial].
    *   Calculate a "diffuseness" metric `dif` for normal estimation:
        $$ \text{dif} = \max(0.001, \frac{2.0 \cdot (\mathbf{n}_{hit} \cdot \mathbf{projectionFactor})}{\mathbf{p}_{hit}.x}) $$
        Apply clamping: if $\text{dif} > 5.0$, mix with 1.0 using a smoothstep over range $[5.0, 20.0]$ [S1, unofficial].
    *   If no hit, discard or write background [S1, unofficial].
3.  **Pass 2: Material Lookup (Optional):**
    *   If `_useMaterials` is true, compile `SHADER_MATERIAL_FILE` [S1, unofficial].
    *   Bind the material octree buffer to `TEX_NUM_MATOCTREE` [S1, unofficial].
    *   Uniforms: `matOctree`, `screenRes`, `depth`, `matOctreehalfSide`, `matOctreeCenter`, `matOctreeLevels` [S1, unofficial].
    *   For each pixel with a valid hit position, query the material ID from the octree and write to `_glTexHitMaterial` [S1, unofficial].
4.  **Pass 3: Normal Filtering:**
    *   Compile `SHADER_NORMAL_FILTER_FILE` [S1, unofficial].
    *   Uniform: `inputTex` (TEX_NUM_HITNORM) [S1, unofficial].
    *   Apply a bilateral or Gaussian filter with kernel size `MSIZE` (default 10) and sigma `_sigma` (default 4.0) or `_bsigma` (default 0.4) to `_glTexHitNorm` to produce `_glTexHitNormFiltered` [S1, unofficial].
5.  **Pass 4: SSAO (Screen-Space Ambient Occlusion):**
    *   Compile `SHADER_SSAO_FILE` [S1, unofficial].
    *   Uniforms: `hitPosTex` (TEX_NUM_HITPOS), `hitNormTex` (TEX_NUM_HITNORMFILTERED), `depthTex` (TEX_NUM_DEPTH), `noiseTex` (TEX_NUM_AONOISE) [S1, unofficial].
    *   Generate `_ssaoNSamples` (default 32) 3D samples in a hemisphere oriented by the surface normal [S1, unofficial].
    *   Generate a noise texture of size `_ssaoNoiseTexSide` (default 64) [S1, unofficial].
    *   For each pixel, sample the hemisphere, project samples to screen space, and compare against the depth buffer to calculate occlusion [S1, unofficial].
    *   Write visibility factor to `_glTexAO` [S1, unofficial].
6.  **Pass 5: AO Blur:**
    *   Compile `SHADER_BLUR_FILE` [S1, unofficial].
    *   Uniform: `inputTex` (TEX_NUM_AO) [S1, unofficial].
    *   Apply a blur filter to `_glTexAO` to produce `_glTexAOBlurred` [S1, unofficial].
7.  **Pass 6: Shading:**
    *   Compile `SHADER_SHADING_FILE` [S1, unofficial].
    *   Uniforms: `screenRes`, `hitPosTex`, `hitNormTex`, `shadowsTex`, `aoTex`, `depthTex`, `materialTex` [S1, unofficial].
    *   If materials are enabled, define `USE_MATERIALS` and `NUM_MATERIALS` [S1, unofficial].
    *   For each pixel:
        *   Retrieve hit position, normal, shadow factor, and AO value [S1, unofficial].
        *   Determine diffuse color: use material color if `useMaterials` is true, else default (e.g., 0.8, 0.8, 0.8) [S1, unofficial].
        *   Compute final color using lighting model (diffuse, ambient) and apply AO [S1, unofficial].
        *   Output to final color buffer [S1, unofficial].
8.  **Pass 7: Volume Rendering (NanoVolume Pass):**
    *   In the Unity implementation, a separate `NanoVolumePass` is executed [S2, unofficial].
    *   Include headers: `CustomPassCommon.hlsl`, `PseudoRandom.hlsl`, `NanoVolumePass.hlsl` [S2, unofficial].
    *   Setup stereo eye index if in stereo mode [S2, unofficial].
    *   Load camera depth at `varyings.positionCS.xy` [S2, unofficial].
    *   Get position inputs using `GetPositionInput` [S2, unofficial].
    *   Calculate normalized view direction [S2, unofficial].
    *   Call `NanoVolumePass` with camera position and negated view direction [S2, unofficial].
    *   Return color with alpha blending: `float4(color.rgb, color.a)` [S2, unofficial].
    *   Render State: ZWrite Off, Cull Off, Blend One OneMinusSrcAlpha [S2, unofficial].

### Formulas
*   **Projection Factor:**
    $$ \text{projectionFactor} = \frac{\text{inv\_2tan\_half\_fovy}}{\text{screen\_tolerance}} $$
    where $\text{inv\_2tan\_half\_fovy} = \frac{1.0}{2.0 \cdot \tan(0.5 \cdot \text{fovy})}$ and $\text{screen\_tolerance} = \frac{1.0}{\text{screenRes}[1]}$ [S1, unofficial].
    Typical FOV is 60 degrees [S1, unofficial].

*   **Diffuseness Metric (for Normal Estimation):**
    $$ \text{dif} = \max(0.001, \frac{2.0 \cdot (\mathbf{n} \cdot \text{projectionFactor})}{\mathbf{p}.x}) $$
    Clamped via:
    $$ \text{dif}_{final} = \begin{cases} \text{dif} & \text{if } \text{dif} \le 5.0 \\ \text{mix}(\text{dif}, 1.0, \frac{\text{dif}-5.0}{15.0}) & \text{if } 5.0 < \text{dif} < 20.0 \\ 1.0 & \text{if } \text{dif} \ge 20.0 \end{cases} $$
    [S1, unofficial].

*   **SSAO Occlusion:**
    $$ \text{occlusion} = \sum_{i=0}^{N-1} \left( \frac{\text{sampleDepth} < \|\mathbf{c}_{cam} - \mathbf{s}_i\|}{\|\mathbf{c}_{cam} - \mathbf{s}_i\|} \cdot \text{rangeCheck} \right) $$
    where $\mathbf{s}_i$ is the $i$-th sample in tangent space, transformed to world and then clip space, and $\text{rangeCheck}$ is 1.0 if the sample is within radius, 0.0 otherwise [S1, unofficial].
    Final visibility:
    $$ \text{visibility} = 1.0 - \frac{\text{occlusion}}{\text{count}} $$
    [S1, unofficial].

### Parameters
*   `_sigma`: 4.0f (Normal filtering/SSAO smoothing) [S1, unofficial].
*   `_bsigma`: 0.4f (Bilateral filtering/SSAO smoothing) [S1, unofficial].
*   `_msize`: 10 (Kernel size for normal filtering) [S1, unofficial].
*   `_ssaoRadius`: Calculated as `eo->getHalfSide(eo->getNLevels()) * 100.0f` [S1, unofficial].
*   `_ssaoNSamples`: 32 (Number of SSAO samples) [S1, unofficial].
*   `_ssaoNoiseTexSide`: 64 (Noise texture side length) [S1, unofficial].
*   `_renderMode`: 0 (Default render mode) [S1, unofficial].
*   `_useDif`: true (Enable diffuse shading) [S1, unofficial].
*   `_filterNormals`: true (Enable normal filtering) [S1, unofficial].
*   `_doSSAO`: true (Enable SSAO) [S1, unofficial].
*   `_doShadows`: true (Enable shadows) [S1, unofficial].
*   `_useMaterials`: false (Enable material lookup) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision:** The use of `GL_RGBA32F` for depth and position buffers ensures high precision for ray origin/normal storage, preventing z-fighting or normal artifacts in large scenes [S1, unofficial].
*   **Boundaries:** The SSAO pass discards pixels where `hitPos == vec3(0,0,0)` (background) [S1, unofficial].
*   **Degenerate Input:** If the volume octree is not loaded (`_materialOctree` is NULL), the material pass is skipped [S1, unofficial].
*   **Stereo:** The Unity implementation explicitly handles stereo eye indexing via `UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX` [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the GPU ray traversal pass (Pass 1) and the SSAO pass (Pass 4). Ray traversal is $O(\log N)$ per pixel for sparse octrees, where $N$ is the number of nodes [S1, unofficial]. SSAO is $O(W \cdot H \cdot N_{samples})$ [S1, unofficial].
*   **Memory Complexity:** Requires multiple full-screen textures (Depth, Pos, Norm, Material, Shadow, AO, AO Blurred). Total memory $\approx 7 \times W \times H \times \text{bytesPerPixel}$ [S1, unofficial]. For 1080p, this is roughly 7 * 1920 * 1080 * 4 bytes $\approx 58$ MB for 32-bit floats, plus overhead [S1, unofficial].
*   **Performance:** Real-time performance is achievable on modern GPUs due to hardware-accelerated rasterization for post-processing passes and efficient GPU-based traversal [S1, S2, unofficial].
*   **Strengths:** Decouples volume traversal from lighting, allowing for efficient screen-space lighting techniques (SSAO, Shadows) [S1, S2, unofficial]. Supports material lookups and complex shading models [S1, unofficial].
*   **Limitations:** Requires multiple render passes, increasing overhead. Screen-space techniques (SSAO) suffer from edge artifacts and require depth/normal buffers [S1, unofficial].

## Background
Custom Pass Rendering originated from the SSVDAG (Sparse Sparse Volume DAG) project and was adapted for Unity via the NanoVolume renderer [S1, S2, unofficial]. It was developed to enable real-time rendering of large, sparse volumetric datasets (like those from medical imaging or scientific simulations) by leveraging the efficiency of GPU-based ray traversal combined with standard graphics pipeline features for lighting [S1, S2, unofficial]. The method is used in research and demo applications for visualizing VDB-based volumes [S1, S2, unofficial].

## Sources
[S1] ajaspe/ssvdag, https://github.com/ajaspe/ssvdag, fetched 2026-10-24 (unofficial)
[S2] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
