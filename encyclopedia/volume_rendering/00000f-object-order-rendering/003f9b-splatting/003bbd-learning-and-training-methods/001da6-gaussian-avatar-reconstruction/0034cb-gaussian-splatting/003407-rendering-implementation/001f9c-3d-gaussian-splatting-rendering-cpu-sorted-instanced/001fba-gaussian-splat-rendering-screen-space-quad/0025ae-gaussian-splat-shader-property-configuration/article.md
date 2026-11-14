# Gaussian Splat Shader Property Configuration
Also called: Render property block setup, Splat instance property binding

## Overview
This method configures the material property block with splat count, spherical harmonic (SH) degree, brightness, scale, antialiasing, transform matrix, and optional skinning or volume lighting data for the GPU render pass [S1, unofficial]. It differs from sibling methods by handling the CPU-side setup of the `MaterialPropertyBlock` and buffer bindings, whereas [Gaussian Splat Fragment Shader](3258-gaussian-splat-fragment-shader.md) handles per-pixel evaluation and [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md) handles draw call issuance.

## How it works
The configuration process initializes buffers, binds textures to shader uniforms, and sets scalar parameters that control the appearance and evaluation of the Gaussian splats [S1, S2, S3, unofficial].

### Inputs and Outputs
*   **Inputs**: Splat data buffers, renderer-specific parameters (brightness, scale factor, SH degree, gamma), and global state flags (dirty buffers, total splat count, active SH bands) [S3, unofficial].
*   **Outputs**: A populated `MaterialPropertyBlock` ready for the draw call, and bound GPU buffers accessible by the fragment shader [S1, S3, unofficial].

### Data Structures
#### RendererParams
A struct passed to the shader to control per-renderer appearance [S3, unofficial].
*   `Brightness`: `float` (Default: $1.0$) [S1, S3, unofficial].
*   `ScaleFactor`: `float` (Default: $1.0$) [S1, S3, unofficial].
*   `GammaToLinear`: `uint` ($1$ if gamma correction is enabled, $0$ otherwise) [S3, unofficial].
*   `SHDegree`: `uint` (Clamped to active SH bands) [S1, S3, unofficial].

#### GsplatRigWeight (Skinning)
Used for animated splats [S1, unofficial]. Layout inferred from buffer creation requirements:
*   `BoneIndex0`: `uint`
*   `Weight0`: `float`
*   Padding: To align with `sizeof(uint) * 4 + sizeof(float) * 4` total size [S1, unofficial].

### Steps
1.  **Check Validity**: Return early if `m_globalBuffersDirty` is true, `m_globalOrderBuffer` is null, or `m_totalRemainingCount` is $0$ [S3, unofficial].
2.  **Initialize Property Block**: Clear or initialize `m_globalPropertyBlock` [S3, unofficial].
3.  **Bind Core Buffers**: Bind `k_globalOrderBuffer`, `k_globalPackedBuffer`, `k_rendererOffsetsProp`, `k_rendererTransformsProp`, and `k_rendererParamsProp` to their respective shader properties [S3, unofficial].
4.  **Bind Integer Parameters**: Set `k_totalSplatCount` to `m_totalRemainingCount` and `k_splatInstanceSize` to `GsplatSettings.Instance.SplatInstanceSize` [S3, unofficial].
5.  **Bind Spherical Harmonics (SH) Buffers**: Determine `m_globalSHBands` [S3, unofficial]. Conditionally bind `k_globalSH1Buffer` through `k_globalSH4Buffer` if `m_globalSHBands` is $\ge 1$ through $\ge 4$ respectively [S3, unofficial].
6.  **Configure Renderer Parameters**: For each active renderer $k$, if it is a `GsplatRenderer`, set `Brightness` = `gs.Brightness`, `ScaleFactor` = $1.0 - \text{gs.SplatDownscaleFactor}$, `GammaToLinear` = `gs.GammaToLinear ? 1 : 0`, and `SHDegree` = $\min(\text{gs.SHDegree}, \text{activeGsplats}[k].\text{SHBands})$ [S3, unofficial]. If it is not a `GsplatRenderer`, set `Brightness` = $1.0$, `ScaleFactor` = $1.0$, `GammaToLinear` = $0$, and `SHDegree` = $\text{activeGsplats}[k].\text{SHBands}$ [S3, unofficial]. Update the cached `RendererParams` array into `m_rendererParamsBuffer` [S3, unofficial].
7.  **Bind Additional Textures (WebGL Variant)**: Bind `sourceSplats` and `sourceSplats2` [S2, unofficial]. Bind SH block textures `shBlock0Texture` through `shBlock5Texture`, using `Splats.emptyTexture` for missing blocks [S2, unofficial]. Set layer configuration uniforms: `sourceLayerBits` to `layerBits`, `sourceLayerMask` to $2^{\text{layerBits}} - 1$, and `sourceBlockBits` to `SPLAT_BLOCKS_DISABLED` [S2, unofficial]. Disable advanced features by setting `sourcePackedGroups` to false, `sourceBlocks`/`sourceOpacities`/`sourceIndices` to `Splats.emptyTexture`, and `sourceIndexed` to false [S2, unofficial].
8.  **Finalize**: Mark the splats object as needing no further updates (`needsUpdate` = false) [S2, unofficial]. Construct `RenderParams` with the appropriate global material, a world bounds covering $10^6$ units in all directions, and the populated property block [S3, unofficial]. Compute instance count by dividing remaining splat count by `SplatInstanceSize`, rounded up, and issue the draw call using `RenderMeshPrimitives` [S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def configure_splat_shader_properties(renderer, material_property_block, global_state):
    """
    Configures the GPU render state for Gaussian Splatting.
    
    Args:
        renderer: The GsplatRenderer instance.
        material_property_block: The object to bind properties to.
        global_state: Object containing m_globalBuffersDirty, m_totalRemainingCount, 
                      m_globalSHBands, etc.
    """
    # Step 1: Check Validity
    if global_state.m_globalBuffersDirty:
        return
    if global_state.m_globalOrderBuffer is None:
        return
    if global_state.m_totalRemainingCount == 0:
        return

    # Step 2: Initialize Property Block
    material_property_block.clear()

    # Step 3: Bind Core Buffers
    material_property_block.set_buffer("k_globalOrderBuffer", global_state.k_globalOrderBuffer)
    material_property_block.set_buffer("k_globalPackedBuffer", global_state.k_globalPackedBuffer)
    material_property_block.set_buffer("k_rendererOffsetsProp", global_state.k_rendererOffsetsProp)
    material_property_block.set_buffer("k_rendererTransformsProp", global_state.k_rendererTransformsProp)

    # Step 4: Bind Integer Parameters
    material_property_block.set_int("k_totalSplatCount", global_state.m_totalRemainingCount)
    material_property_block.set_int("k_splatInstanceSize", global_state.splat_instance_size)

    # Step 5: Bind SH Buffers
    if global_state.m_globalSHBands >= 1:
        material_property_block.set_buffer("k_globalSH1Buffer", global_state.k_globalSH1Buffer)
    if global_state.m_globalSHBands >= 2:
        material_property_block.set_buffer("k_globalSH2Buffer", global_state.k_globalSH2Buffer)
    if global_state.m_globalSHBands >= 3:
        material_property_block.set_buffer("k_globalSH3Buffer", global_state.k_globalSH3Buffer)
    if global_state.m_globalSHBands >= 4:
        material_property_block.set_buffer("k_globalSH4Buffer", global_state.k_globalSH4Buffer)

    # Step 6: Configure Renderer Parameters
    renderer_params = {
        "Brightness": 1.0,
        "ScaleFactor": 1.0,
        "GammaToLinear": 0,
        "SHDegree": 0
    }

    if renderer.is_gsplat_renderer():
        gs = renderer.get_gsplat_settings()
        active_bands = renderer.active_gsplats.sh_bands
        renderer_params["Brightness"] = gs.brightness
        renderer_params["ScaleFactor"] = 1.0 - gs.splat_downscale_factor
        renderer_params["GammaToLinear"] = 1 if gs.gamma_to_linear else 0
        renderer_params["SHDegree"] = min(gs.sh_degree, active_bands)
    else:
        renderer_params["Brightness"] = 1.0
        renderer_params["ScaleFactor"] = 1.0
        renderer_params["GammaToLinear"] = 0
        renderer_params["SHDegree"] = renderer.active_gsplats.sh_bands

    global_state.m_renderer_params_buffer.update(renderer_params)
    material_property_block.set_buffer("k_rendererParamsProp", global_state.m_renderer_params_buffer)

    # Step 7: WebGL Specific Bindings (if applicable)
    # material_property_block.set_texture("sourceSplats", splats_texture)
    # material_property_block.set_texture("sourceSplats2", splats_texture_2)
    # Bind SH blocks 0-5, using empty texture if missing
    # Set layer bits and disable advanced features

    # Step 8: Finalize
    # renderer.needs_update = False
```

### Parameters and Typical Values
*   `shDegree`: Order of SH coefficients. Capped by the asset's `SHBands` property [S1, unofficial]. Typical values: $0, 1, 2, 3, 4$.
*   `brightness`: Brightness color scaling factor. Default: $1.0$ [S1, unofficial].
*   `scaleFactor`: Splat UV scaling factor. Default: $1.0$ [S1, unofficial].
*   `gammaToLinear`: Boolean flag ($0$ or $1$). Default: $0$ (disabled) [S3, unofficial].

### Edge Cases and Pitfalls
*   **Stale Data**: Drawing is skipped if `m_globalBuffersDirty` is true, which is critical in URP where `DrawAllIfEnabled` may fire before `DispatchSort` completes [S3, unofficial].
*   **SH Buffer Overread**: `SHDegree` must be clamped to the renderer's `SHBands` to prevent `EvalSH` from reading uninitialized data, as global buffers are sized to the maximum bands across all renderers [S3, unofficial].
*   **Portable Shader Variants**: Valid sampled resources must still be bound when skinning is disabled, because binding null or uninitialized buffers causes shader compilation or runtime errors [S1, unofficial].
*   **Missing SH Blocks**: In WebGL implementations, missing SH block textures must be bound to a dummy `emptyTexture` to prevent errors [S2, unofficial].

## Subtopics

### Renderer Uniform Configuration
A variant focusing on WebGL-specific uniform binding [S2, unofficial]. It binds main splat textures (`sourceSplats`, `sourceSplats2`) and SH block textures (`shBlock0Texture` to `shBlock5Texture`), using `Splats.emptyTexture` for missing blocks [S2, unofficial]. It also sets layer configuration uniforms (`sourceLayerBits`, `sourceLayerMask` = $2^{\text{layerBits}} - 1$, `sourceBlockBits` = `SPLAT_BLOCKS_DISABLED`) and disables advanced features like packed groups and indexed rendering by binding empty textures [S2, unofficial]. It marks the splats object as needing no further updates (`needsUpdate` = false) [S2, unofficial].

### Global Draw Call with MaterialPropertyBlock
A variant using Unity's `MaterialPropertyBlock` to bind global splat data buffers [S3, unofficial]. It supports multi-camera setups by avoiding state overwrites [S3, unofficial]. It returns early if `m_globalBuffersDirty`, `m_globalOrderBuffer` is null, or `m_totalRemainingCount` is $0$ [S3, unofficial]. It binds core buffers (`k_globalOrderBuffer`, `k_globalPackedBuffer`, `k_rendererOffsetsProp`, `k_rendererTransformsProp`, `k_rendererParamsProp`) and integer parameters (`k_totalSplatCount`, `k_splatInstanceSize`) [S3, unofficial]. It conditionally binds SH buffers based on `m_globalSHBands` [S3, unofficial].

### Renderer Parameter Caching and SH Degree Clamping
A variant that updates per-renderer parameters (brightness, scale, gamma, SH degree) and caches them in a `RendererParams` array [S3, unofficial]. It ensures `SHDegree` is clamped to `activeGsplats[k].SHBands` to prevent out-of-bounds reads [S3, unofficial]. For `GsplatRenderer` instances, it sets `Brightness` = `gs.Brightness`, `ScaleFactor` = $1.0 - \text{gs.SplatDownscaleFactor}$, `GammaToLinear` = `gs.GammaToLinear ? 1 : 0`, and `SHDegree` = $\min(\text{gs.SHDegree}, \text{activeGsplats}[k].\text{SHBands})$ [S3, unofficial]. For other renderers, it uses defaults (`Brightness` = $1.0$, `ScaleFactor` = $1.0$, `SHDegree` = `SHBands`) [S3, unofficial]. It sets the cached `RendererParams` array into `m_rendererParamsBuffer` [S3, unofficial].

### Global Splat Rendering Setup
A variant that constructs the final `RenderParams` object, including the global material, world bounds ($10^6$ units), and the populated property block [S3, unofficial]. It binds renderer parameters buffer containing total splat count and instance size to `k_rendererParamsProp` [S3, unofficial]. It conditionally binds spherical harmonic buffers for bands $1$ through $4$ based on `m_globalSHBands` [S3, unofficial]. It computes instance count by dividing remaining splat count by `SplatInstanceSize` (rounded up) and issues the draw call using `RenderMeshPrimitives` [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Performance**: Not given in the sources.
*   **Strengths**: Not given in the sources.
*   **Limitations**: Not given in the sources.
*   **When to Use**: Not given in the sources.
*   **When Not to Use**: Not given in the sources.

## Background
This method is part of the rendering implementation for 3D Gaussian Splatting in real-time engines, specifically adapted for Unity and WebGL [S3, S2, unofficial]. It builds on the screen-space quad rendering approach [2075-gaussian-splat-rendering-screen-space-quad.md] and handles the data transfer from CPU to GPU [S3, unofficial]. The use of `MaterialPropertyBlock` is a Unity-specific optimization for multi-camera scenarios [S3, unofficial]. The WebGL variant adapts this configuration for browser-based rendering [S2, unofficial].

## Sources
[S1] arloopa/UnitySplats, https://github.com/arloopa/UnitySplats, fetched 2026-10-27 (unofficial)
[S2] WilliamLiu-1997/Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
[S3] wuyize25/gsplat-unity, https://github.com/wuyize25/gsplat-unity, fetched 2026-10-27 (unofficial)
