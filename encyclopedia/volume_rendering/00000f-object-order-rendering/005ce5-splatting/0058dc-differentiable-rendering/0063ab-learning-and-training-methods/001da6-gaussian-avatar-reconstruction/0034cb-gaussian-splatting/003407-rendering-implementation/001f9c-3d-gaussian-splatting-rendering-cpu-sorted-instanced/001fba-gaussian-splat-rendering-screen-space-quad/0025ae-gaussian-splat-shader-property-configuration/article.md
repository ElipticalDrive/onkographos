# Gaussian Splat Shader Property Configuration

Also called: Render property block setup, Splat instance property binding

## Overview
This method configures the GPU render pass by populating a material property block (or uniform buffer) with splat count, spherical harmonic (SH) degree, brightness, scale, antialiasing settings, transform matrices, and optional skinning or volume lighting data. It solves the problem of efficiently passing per-renderer and global state to the fragment shader without causing state overwrites in multi-camera setups or shader compilation errors when optional features (like skinning) are disabled. It differs from standard mesh rendering by explicitly handling the decoupling of jittered projections and ensuring valid sampled resources even for unused features [S1, S3, unofficial].

## How it works
### Inputs and Outputs
- **Inputs**: 
  - Splat count (`m_totalRemainingCount` or `SplatCount`).
  - SH degree (`shDegree` or `SHBands`).
  - Brightness scaling factor (`brightness`, default 1.0f).
  - Scale factor (`scaleFactor`, default 1.0f).
  - Renderer-specific parameters (gamma, downscale).
  - Global buffers: Order buffer, Packed buffer, SH buffers, Transform buffers.
  - Optional: Skinning weights/bone indices if rigging is enabled.
- **Outputs**: 
  - A populated `MaterialPropertyBlock` (Unity) or uniform buffer structure containing all necessary shader constants and texture bindings.

### Data Structures
**RendererParams** [S3, unofficial]:
- `Brightness`: `float`
- `ScaleFactor`: `float`
- `GammaToLinear`: `uint` (1 if true, 0 if false)
- `SHDegree`: `uint`

**GsplatRigWeight** (for skinning support) [S1, unofficial]:
- `BoneIndex0`: `uint`
- `Weight0`: `float`
- Padding: Aligned to `sizeof(uint) * 4 + sizeof(float) * 4` total size.

### Steps
1. **Check Global State**: If `m_globalBuffersDirty` is true, `m_globalOrderBuffer` is null, or `m_totalRemainingCount` is 0, return early to prevent stale buffer usage [S3, unofficial].
2. **Initialize Property Block**: Clear or initialize `m_globalPropertyBlock` [S3, unofficial].
3. **Bind Core Buffers**: Bind the following to their respective shader properties:
   - `k_globalOrderBuffer`
   - `k_globalPackedBuffer`
   - `k_rendererOffsetsProp`
   - `k_rendererTransformsProp`
   - `k_rendererParamsProp`
4. **Bind Integer Parameters**:
   - `k_totalSplatCount` = `m_totalRemainingCount`
   - `k_splatInstanceSize` = `GsplatSettings.Instance.SplatInstanceSize`
5. **Bind SH Buffers**: Conditionally bind SH buffers based on `m_globalSHBands`:
   - If `m_globalSHBands >= 1`, bind `k_globalSH1Buffer`.
   - If `m_globalSHBands >= 2`, bind `k_globalSH2Buffer`.
   - If `m_globalSHBands >= 3`, bind `k_globalSH3Buffer`.
   - If `m_globalSHBands >= 4`, bind `k_globalSH4Buffer`.
6. **Update Per-Renderer Parameters** [S3, unofficial]:
   - For each active renderer `k`:
     - If `k` is a `GsplatRenderer`:
       - `Brightness` = `gs.Brightness`
       - `ScaleFactor` = `1.0f - gs.SplatDownscaleFactor`
       - `GammaToLinear` = `gs.GammaToLinear ? 1u : 0u`
       - `SHDegree` = `min(gs.SHDegree, activeGsplats[k].SHBands)`
     - Else:
       - `Brightness` = `1.0f`
       - `ScaleFactor` = `1.0f`
       - `GammaToLinear` = `0u`
       - `SHDegree` = `activeGsplats[k].SHBands`
   - Set the cached `RendererParams` array into `m_rendererParamsBuffer`.
7. **Bind Textures (WebGL Variant)** [S2, unofficial]:
   - Bind `sourceSplats` and `sourceSplats2`.
   - Bind SH block textures `shBlock0Texture` through `shBlock5Texture`. If a block is missing, bind `Splats.emptyTexture`.
   - Set layer configuration: `sourceLayerBits` = `layerBits`, `sourceLayerMask` = `2 ** layerBits - 1`, `sourceBlockBits` = `SPLAT_BLOCKS_DISABLED`.
   - Disable advanced features: `sourcePackedGroups` = `false`, `sourceBlocks`/`sourceOpacities`/`sourceIndices` = `Splats.emptyTexture`, `sourceIndexed` = `false`.
   - Mark splats object as needing no further updates (`needsUpdate` = `false`).
8. **Construct Render Params**: Create `RenderParams` with the global material, world bounds (1e6 units), and the populated property block [S3, unofficial].
9. **Compute Instance Count**: `instanceCount` = `ceil(m_totalRemainingCount / SplatInstanceSize)` [S3, unofficial].
10. **Issue Draw Call**: Use `RenderMeshPrimitives` with the instance mesh, `instanceCount`, and `RenderParams` [S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def configure_splat_shader_properties(renderer, global_state, splat_asset):
    """
    Configures the material property block for Gaussian Splat rendering.
    """
    # Step 1: Check global state
    if global_state.m_global_buffers_dirty or \
       global_state.m_global_order_buffer is None or \
       global_state.m_total_remaining_count == 0:
        return

    # Step 2: Initialize Property Block
    prop_block = MaterialPropertyBlock()

    # Step 3: Bind Core Buffers
    prop_block.set_buffer("k_globalOrderBuffer", global_state.k_global_order_buffer)
    prop_block.set_buffer("k_globalPackedBuffer", global_state.k_global_packed_buffer)
    prop_block.set_buffer("k_rendererOffsetsProp", global_state.k_renderer_offsets_prop)
    prop_block.set_buffer("k_rendererTransformsProp", global_state.k_renderer_transforms_prop)
    prop_block.set_buffer("k_rendererParamsProp", global_state.k_renderer_params_prop)

    # Step 4: Bind Integer Parameters
    prop_block.set_int("k_totalSplatCount", global_state.m_total_remaining_count)
    prop_block.set_int("k_splatInstanceSize", GsplatSettings.instance.splat_instance_size)

    # Step 5: Bind SH Buffers
    global_sh_bands = global_state.m_global_sh_bands
    if global_sh_bands >= 1:
        prop_block.set_texture("k_globalSH1Buffer", global_state.k_global_sh1_buffer)
    if global_sh_bands >= 2:
        prop_block.set_texture("k_globalSH2Buffer", global_state.k_global_sh2_buffer)
    if global_sh_bands >= 3:
        prop_block.set_texture("k_globalSH3Buffer", global_state.k_global_sh3_buffer)
    if global_sh_bands >= 4:
        prop_block.set_texture("k_globalSH4Buffer", global_state.k_global_sh4_buffer)

    # Step 6: Update Per-Renderer Parameters
    renderer_params_array = []
    for k in range(renderer.active_renderer_count):
        gs = renderer.active_gsplats[k]
        if renderer.is_gsplat_renderer(k):
            brightness = gs.brightness
            scale_factor = 1.0f - gs.splat_downscale_factor
            gamma_to_linear = 1 if gs.gamma_to_linear else 0
            sh_degree = min(gs.sh_degree, gs.sh_bands)
        else:
            brightness = 1.0f
            scale_factor = 1.0f
            gamma_to_linear = 0
            sh_degree = gs.sh_bands
        
        renderer_params_array.append({
            "Brightness": brightness,
            "ScaleFactor": scale_factor,
            "GammaToLinear": gamma_to_linear,
            "SHDegree": sh_degree
        })
    
    prop_block.set_buffer("m_rendererParamsBuffer", renderer_params_array)

    # Step 7: WebGL Variant Specifics (if applicable)
    # Note: This section is specific to WebGL implementations [S2, unofficial]
    # prop_block.set_texture("sourceSplats", source_splats)
    # prop_block.set_texture("sourceSplats2", source_splats2)
    # for i in range(6):
    #     tex = sh_block_textures[i] if i < len(sh_block_textures) else empty_texture
    #     prop_block.set_texture(f"shBlock{i}Texture", tex)
    # prop_block.set_int("sourceLayerBits", layer_bits)
    # prop_block.set_int("sourceLayerMask", 2 ** layer_bits - 1)
    # prop_block.set_int("sourceBlockBits", SPLAT_BLOCKS_DISABLED)
    # prop_block.set_bool("sourcePackedGroups", False)
    # prop_block.set_texture("sourceBlocks", empty_texture)
    # prop_block.set_texture("sourceOpacities", empty_texture)
    # prop_block.set_texture("sourceIndices", empty_texture)
    # prop_block.set_bool("sourceIndexed", False)
    # splats_object.needs_update = False

    # Step 8: Construct Render Params
    render_params = RenderParams(
        material=splat_asset.gsplat_material,
        world_bounds=Bounds(center=(0,0,0), extents=(1e6, 1e6, 1e6)),
        property_block=prop_block
    )

    # Step 9: Compute Instance Count
    instance_count = ceil(global_state.m_total_remaining_count / GsplatSettings.instance.splat_instance_size)

    # Step 10: Issue Draw Call
    render_mesh_primitives(
        instance_mesh=splat_asset.instance_mesh,
        instance_count=instance_count,
        render_params=render_params
    )
```

### Parameters
- `shDegree`: Order of SH coefficients used for rendering, capped by the asset's `SHBands` property [S1, unofficial].
- `brightness`: Brightness color scaling factor, default 1.0f [S1, unofficial].
- `scaleFactor`: Splat UV scaling factor to reduce splat size while keeping visual fidelity, default 1.0f [S1, unofficial].

### Edge Cases and Pitfalls
- **Portable Shader Variants**: If skinning is disabled, valid sampled resources must still be bound. Binding null or uninitialized buffers causes shader compilation/runtime errors [S1, unofficial].
- **Stale Buffers**: Drawing is skipped if `m_globalBuffersDirty` is true. This is critical in URP where `DrawAllIfEnabled` may fire before `DispatchSort` [S3, unofficial].
- **SH Degree Clamping**: `SHDegree` must be clamped to the renderer's `SHBands` to prevent `EvalSH` from reading uninitialized data, as global buffers are sized to the max bands across all renderers [S3, unofficial].

## Subtopics

### [Gaussian Splat TAA Jitter Integration](13883-gaussian-splat-taa-jitter-integration.md)
Integrates temporal anti-aliasing (TAA) jitter into the Gaussian Splatting rendering pipeline to prevent splats from appearing "swimming" or bit-stable while surrounding meshes are correctly jittered. It solves the problem where splats, previously rendered with an unjittered projection matrix, failed to accumulate temporal history correctly under TAA and temporal upscaling (e.g., FSR2), causing residual displacement and visual artifacts. It differs from standard mesh rendering by explicitly excluding the engine's depth correction from the jittered projection, as the Gaussian pipeline relies on linear view-space depth [13883-gaussian-splat-taa-jitter-integration.md].

## Operating characteristics
- **Time Complexity**: O(1) for property configuration per frame, assuming buffer updates are handled separately. The draw call is O(N) where N is the number of splats, but the configuration step itself is constant relative to splat count.
- **Memory Complexity**: O(1) for the property block itself, but depends on the size of the global buffers (Order, Packed, SH) which are O(N).
- **Strengths**: Efficiently handles multi-camera setups without state overwrites [S3, unofficial]. Prevents shader errors by ensuring valid resources for disabled features [S1, unofficial].
- **Limitations**: Requires careful management of buffer dirty flags to avoid stale data [S3, unofficial]. SH degree clamping is necessary to prevent out-of-bounds reads [S3, unofficial].
- **When to Use**: Always required for rendering 3D Gaussian Splatting in Unity or WebGL environments.
- **When Not to Use**: Not applicable; this is a core rendering step.

## Background
Originated from the implementation of 3D Gaussian Splatting in real-time engines like Unity and WebGL. The method was developed to address the specific challenges of passing large amounts of per-splat and global state to the GPU efficiently. Key implementations include `UnitySplats` [S1, S3, unofficial] and `Gaussian-Splat-Lite` [S2, unofficial]. The approach evolved to support multi-camera setups, TAA jitter integration, and efficient SH evaluation.

## Sources
[S1] arloopa/UnitySplats, https://github.com/arloopa/UnitySplats, fetched 2026-10-27 (unofficial)
[S2] WilliamLiu-1997/Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
[S3] wuyize25/gsplat-unity, https://github.com/wuyize25/gsplat-unity, fetched 2026-10-27 (unofficial)
