# Gaussian Splat Shader Property Configuration

Also called: Render property block setup, Splat instance property binding

## Overview
This method configures the GPU render state by binding splat data buffers, spherical harmonic (SH) coefficients, and per-renderer parameters (brightness, scale, SH degree) to a material's property block. It solves the problem of efficiently passing large, variable-sized splat datasets to the fragment shader without redundant state changes, enabling multi-camera setups and dynamic updates. It differs from sibling methods by focusing on the **CPU-side setup** of the `MaterialPropertyBlock` and buffer bindings, whereas [Gaussian Splat Fragment Shader](3258-gaussian-splat-fragment-shader.md) handles the per-pixel evaluation and [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md) handles the draw call issuance.

## How it works
The configuration process involves initializing buffers, binding textures to shader uniforms, and setting scalar parameters that control the appearance and evaluation of the Gaussian splats.

### Inputs and Outputs
*   **Inputs**:
    *   Splat data buffers (positions, rotations, scales, opacities, SH coefficients).
    *   Renderer-specific parameters (brightness, scale factor, SH degree, gamma).
    *   Global state flags (dirty buffers, total splat count, active SH bands).
*   **Outputs**:
    *   A populated `MaterialPropertyBlock` (or equivalent uniform state) ready for the draw call.
    *   Bound GPU buffers accessible by the fragment shader.

### Data Structures

#### RendererParams
A struct passed to the shader to control per-renderer appearance.
*   `Brightness`: `float` (Default: 1.0) [S1, S3, unofficial].
*   `ScaleFactor`: `float` (Default: 1.0) [S1, S3, unofficial].
*   `GammaToLinear`: `uint` (1 if gamma correction is enabled, 0 otherwise) [S3, unofficial].
*   `SHDegree`: `uint` (Clamped to active SH bands) [S1, S3, unofficial].

#### GsplatRigWeight (Skinning)
Used for animated splats. Layout inferred from buffer creation requirements:
*   `BoneIndex0`: `uint`
*   `Weight0`: `float`
*   Padding: To align with `sizeof(uint) * 4 + sizeof(float) * 4` total size [S1, unofficial].

### Steps

1.  **Check Validity**: Return early if global buffers are dirty (`m_globalBuffersDirty`), the order buffer is null, or the total remaining splat count is 0. This prevents rendering with stale data, especially critical in pipelines like URP where draw calls may fire before sorting completes [S3, unofficial].
2.  **Initialize Property Block**: Clear or initialize the `MaterialPropertyBlock` (`m_globalPropertyBlock`) [S3, unofficial].
3.  **Bind Core Buffers**: Bind the following buffers to their respective shader properties:
    *   `k_globalOrderBuffer`: Pre-sorted splat indices [S3, unofficial].
    *   `k_globalPackedBuffer`: Packed splat attributes (position, rotation, scale, opacity) [S3, unofficial].
    *   `k_rendererOffsetsProp`: Renderer offsets [S3, unofficial].
    *   `k_rendererTransformsProp`: World-to-view matrices [S3, unofficial].
    *   `k_rendererParamsProp`: The `RendererParams` struct [S3, unofficial].
4.  **Bind Integer Parameters**:
    *   `k_totalSplatCount`: Set to `m_totalRemainingCount` [S3, unofficial].
    *   `k_splatInstanceSize`: Set to `GsplatSettings.Instance.SplatInstanceSize` [S3, unofficial].
5.  **Bind Spherical Harmonics (SH) Buffers**:
    *   Determine `m_globalSHBands` (maximum SH bands across all renderers) [S3, unofficial].
    *   Conditionally bind SH coefficient textures/buffers based on `m_globalSHBands`:
        *   If `m_globalSHBands >= 1`: Bind `k_globalSH1Buffer` [S3, unofficial].
        *   If `m_globalSHBands >= 2`: Bind `k_globalSH2Buffer` [S3, unofficial].
        *   If `m_globalSHBands >= 3`: Bind `k_globalSH3Buffer` [S3, unofficial].
        *   If `m_globalSHBands >= 4`: Bind `k_globalSH4Buffer` [S3, unofficial].
    *   *Note*: In WebGL implementations, missing SH block textures are bound to a dummy `Splats.emptyTexture` [S2, unofficial].
6.  **Configure Renderer Parameters**:
    *   For each active renderer `k`:
        *   If it is a `GsplatRenderer`:
            *   `Brightness` = `gs.Brightness` [S3, unofficial].
            *   `ScaleFactor` = `1.0f - gs.SplatDownscaleFactor` [S3, unofficial].
            *   `GammaToLinear` = `gs.GammaToLinear ? 1u : 0u` [S3, unofficial].
            *   `SHDegree` = `min(gs.SHDegree, activeGsplats[k].SHBands)` [S3, unofficial].
        *   If it is not a `GsplatRenderer`:
            *   `Brightness` = 1.0f [S3, unofficial].
            *   `ScaleFactor` = 1.0f [S3, unofficial].
            *   `GammaToLinear` = 0u [S3, unofficial].
            *   `SHDegree` = `activeGsplats[k].SHBands` [S3, unofficial].
    *   Update the cached `RendererParams` array in `m_rendererParamsBuffer` [S3, unofficial].
7.  **Bind Additional Textures (WebGL Variant)**:
    *   Bind `sourceSplats` and `sourceSplats2` [S2, unofficial].
    *   Bind SH block textures (`shBlock0Texture` to `shBlock5Texture`), using `Splats.emptyTexture` if a block is missing [S2, unofficial].
    *   Set layer configuration uniforms: `sourceLayerBits`, `sourceLayerMask` (`2 ** layerBits - 1`), `sourceBlockBits` (`SPLAT_BLOCKS_DISABLED`) [S2, unofficial].
    *   Disable advanced features: `sourcePackedGroups` = false, `sourceBlocks`/`sourceOpacities`/`sourceIndices` = `Splats.emptyTexture`, `sourceIndexed` = false [S2, unofficial].
8.  **Finalize**: Mark the splats object as needing no further updates (`needsUpdate = false`) if applicable [S2, unofficial].

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
        return # Skip rendering to avoid stale data [S3, unofficial]
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
    # Conditionally bind based on global max SH bands [S3, unofficial]
    if global_state.m_globalSHBands >= 1:
        material_property_block.set_buffer("k_globalSH1Buffer", global_state.k_globalSH1Buffer)
    if global_state.m_globalSHBands >= 2:
        material_property_block.set_buffer("k_globalSH2Buffer", global_state.k_globalSH2Buffer)
    if global_state.m_globalSHBands >= 3:
        material_property_block.set_buffer("k_globalSH3Buffer", global_state.k_globalSH3Buffer)
    if global_state.m_globalSHBands >= 4:
        material_property_block.set_buffer("k_globalSH4Buffer", global_state.k_globalSH4Buffer)

    # Step 6: Configure Renderer Parameters
    # Create RendererParams struct
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
        renderer_params["ScaleFactor"] = 1.0f - gs.splat_downscale_factor
        renderer_params["GammaToLinear"] = 1 if gs.gamma_to_linear else 0
        # Clamp SHDegree to prevent reading uninitialized data [S3, unofficial]
        renderer_params["SHDegree"] = min(gs.sh_degree, active_bands)
    else:
        renderer_params["Brightness"] = 1.0f
        renderer_params["ScaleFactor"] = 1.0f
        renderer_params["GammaToLinear"] = 0
        renderer_params["SHDegree"] = renderer.active_gsplats.sh_bands

    # Update buffer with new params
    global_state.m_renderer_params_buffer.update(renderer_params)
    material_property_block.set_buffer("k_rendererParamsProp", global_state.m_renderer_params_buffer)

    # Step 7: WebGL Specific Bindings (if applicable) [S2, unofficial]
    # material_property_block.set_texture("sourceSplats", splats_texture)
    # material_property_block.set_texture("sourceSplats2", splats_texture_2)
    # Bind SH blocks 0-5, using empty texture if missing
    # Set layer bits and disable advanced features

    # Step 8: Finalize
    # renderer.needs_update = False
```

### Parameters and Typical Values
*   `shDegree`: Order of SH coefficients. Capped by the asset's `SHBands` property [S1, unofficial]. Typical values: 0, 1, 2, 3, 4.
*   `brightness`: Brightness color scaling factor. Default: 1.0f [S1, unofficial].
*   `scaleFactor`: Splat UV scaling factor. Default: 1.0f [S1, unofficial].
*   `gammaToLinear`: Boolean flag (0 or 1). Default: 0 (disabled) [S3, unofficial].

### Edge Cases and Pitfalls
*   **Stale Data**: Drawing is skipped if `m_globalBuffersDirty` is true. This is critical in URP where `DrawAllIfEnabled` may fire before `DispatchSort` completes [S3, unofficial].
*   **SH Buffer Overread**: `SHDegree` must be clamped to the renderer's `SHBands`. Global buffers are sized to the max bands across all renderers; failing to clamp causes `EvalSH` to read uninitialized data [S3, unofficial].
*   **Portable Shader Variants**: When skinning is disabled, valid sampled resources must still be bound. Binding null or uninitialized buffers causes shader compilation/runtime errors [S1, unofficial].
*   **Missing SH Blocks**: In WebGL, missing SH block textures must be bound to a dummy `emptyTexture` to prevent errors [S2, unofficial].

## Subtopics

### Renderer Uniform Configuration
A variant focusing on WebGL-specific uniform binding. It binds main splat textures (`sourceSplats`, `sourceSplats2`) and SH block textures (`shBlock0Texture` to `shBlock5Texture`), using `Splats.emptyTexture` for missing blocks. It also sets layer configuration uniforms (`sourceLayerBits`, `sourceLayerMask`, `sourceBlockBits`) and disables advanced features like packed groups and indexed rendering by binding empty textures. This ensures compatibility with simpler WebGL renderers that do not support advanced splat features [S2, unofficial].

### Global Draw Call with MaterialPropertyBlock
A variant using Unity's `MaterialPropertyBlock` to bind global splat data buffers. It supports multi-camera setups by avoiding state overwrites. It binds core buffers (`k_globalOrderBuffer`, `k_globalPackedBuffer`, etc.) and integer parameters (`k_totalSplatCount`, `k_splatInstanceSize`). SH buffers are bound conditionally based on `m_globalSHBands`. This method is robust against stale data by checking `m_globalBuffersDirty` before binding [S3, unofficial].

### Renderer Parameter Caching and SH Degree Clamping
A variant that updates per-renderer parameters (brightness, scale, gamma, SH degree) and caches them in a `RendererParams` array. It ensures `SHDegree` is clamped to `activeGsplats[k].SHBands` to prevent out-of-bounds reads. If the renderer is not a `GsplatRenderer`, default values (Brightness=1.0, ScaleFactor=1.0, SHDegree=SHBands) are used. This allows for efficient updates without rebinding all buffers [S3, unofficial].

### Global Splat Rendering Setup
A variant that constructs the final `RenderParams` object, including the global material, world bounds (1e6 units), and the populated property block. It computes the instance count by dividing the remaining splat count by `SplatInstanceSize` (rounded up) and issues the draw call using `RenderMeshPrimitives`. This is the final step before the GPU executes the render pass [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: O(N) where N is the number of active SH bands (typically small constant, 1-5) for binding SH buffers. The buffer updates are O(1) for scalar parameters and O(B) for buffer bindings, where B is the number of buffers.
*   **Memory Complexity**: O(1) additional memory for the property block and parameter structs. The bulk of memory is in the GPU buffers, which are managed separately.
*   **Performance**: Efficient for multi-camera setups as it avoids redundant state changes. The use of `MaterialPropertyBlock` allows sharing the same material across multiple renderers with different parameters.
*   **Strengths**: Supports dynamic updates, multi-camera rendering, and variable SH degrees. Prevents rendering artifacts from stale data.
*   **Limitations**: Requires careful management of buffer dirty flags. SH buffer binding logic can be complex due to variable band counts.
*   **When to Use**: When rendering 3D Gaussian Splatting in a real-time engine (Unity, WebGL) with multiple cameras or dynamic updates.
*   **When Not to Use**: For static, single-camera renders where pre-baked state might be simpler.

## Background
This method is part of the rendering implementation for 3D Gaussian Splatting in real-time engines, specifically adapted for Unity and WebGL. It builds on the screen-space quad rendering approach [2075-gaussian-splat-rendering-screen-space-quad.md] and handles the data transfer from CPU to GPU. The use of `MaterialPropertyBlock` is a Unity-specific optimization for multi-camera scenarios [S3, unofficial]. The WebGL variant [S2, unofficial] adapts this for browser-based rendering. The method addresses the challenge of efficiently passing large, variable-sized splat datasets to the fragment shader.

## Sources
[S1] arloopa/UnitySplats, https://github.com/arloopa/UnitySplats, fetched 2026-10-27 (unofficial)
[S2] WilliamLiu-1997/Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
[S3] wuyize25/gsplat-unity, https://github.com/wuyize25/gsplat-unity, fetched 2026-10-27 (unofficial)
