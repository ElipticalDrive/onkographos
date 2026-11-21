# Single-Stage Splat Vertex Shader

Also called: MetalGaussianSplatRelighting Vertex Shader

## Overview
This method implements a single-stage vertex shader for splat rasterization that integrates deferred indirect lighting (IBL) and depth-based occlusion directly into the vertex processing stage, rather than deferring these calculations to the fragment shader. It solves the problem of efficiently rendering large-scale Gaussian splats with physically-based relighting and AR scene occlusion on Metal GPUs. It differs from standard splatting by performing lighting and occlusion checks per-vertex (or per-primitive) to modulate coverage and color before fragment generation, reducing fragment shader workload and enabling early discard of occluded primitives.

## How it works
**Inputs and Outputs:**
- **Inputs:** Instance ID, vertex ID, splat metadata (position, covariance, color, opacity), chunk metadata, uniform data (view parameters, lighting environment), scene depth map.
- **Outputs:** Clip-space position, coverage, color (with lighting applied), depth.

**Data Structures:**
- `UniformsArray`: A constant buffer containing an array of `Uniforms` structs. Indexed by `amplificationID`. Fields: `indexedSplatCount`, `splatCount`, `chunkCount` [S1, unofficial].
- `splatIndexArray`: A constant buffer (`ChunkedSplatIndex`) mapping each splat ID to a `ChunkInfo` reference via `chunkIndex` [S1, unofficial].
- `chunks`: A device buffer storing per-chunk metadata (`ChunkInfo`), accessed via `idx.chunkIndex` [S1, unofficial].
- `prefilteredEnv`: A 3D texture (texturecube) storing prefiltered environment maps for specular relighting [S1, unofficial].
- `brdfLUT`: A 2D texture storing the pre-integrated BRDF lookup table for importance sampling [S1, unofficial].
- `irradianceEnv`: A 3D texture (texturecube) storing the irradiance environment map for diffuse relighting [S1, unofficial].

**Parameters:**
- `kMaxViewCount`: The maximum number of simultaneous views supported in the `UniformsArray` [S1, unofficial].

**Steps:**
1. Fetch uniforms from the `UniformsArray` buffer using the `amplificationID` clamped to `kMaxViewCount` [S1, unofficial].
2. Compute the global splat ID by multiplying the instance ID by the indexed splat count per view and adding the vertex ID divided by 4 [S1, unofficial].
3. Discard the primitive if the computed splat ID exceeds the total splat count by outputting a clip-space position of $(1, 1, 0, 1)$ [S1, unofficial].
4. Fetch the chunk index mapping from the `splatIndexArray` using the computed splat ID [S1, unofficial].
5. Discard the primitive if the chunk index exceeds the total chunk count [S1, unofficial].
6. Fetch the chunk metadata from the `chunks` buffer using the validated chunk index [S1, unofficial].
7. Extract normal, view direction, roughness, and reflection strength from vertex attributes (e.g., `v.normalRough`, `v.viewRefl`) [S1, unofficial].
8. Normalize normal $N$ and view vector $V$ [S1, unofficial].
9. Compute roughness as $r = \text{float}(v.\text{normalRough}.a) \times \text{invMW}$ and reflection strength as $s = \text{float}(v.\text{viewRefl}.a) \times \text{invMW}$, where $\text{invMW}$ is the inverse material weight [S1, unofficial].
10. Compute indirect blended color as $\text{indirectBlended} = \text{half3}(\text{float3}(v.\text{indirect}) \times \text{invMW})$ [S1, unofficial].
11. Apply tint if `relight.tint.w > 0.0f` by preserving value using luminance weights $L = [0.2126, 0.7152, 0.0722]$ [S1, unofficial].
12. Shade using IBL deferred function: $\text{shaded} = \text{shadeIBLDeferred}(\text{baseColor}, \text{oriColor}, \text{indirectBlended}, N, V, r, s, \text{relight}, \text{prefilteredEnv}, \text{irradianceEnv}, \text{brdfLUT}, \text{iblSampler})$ [S1, unofficial].
13. If debug mode is enabled, return $\text{half4}(\text{shaded} \times \text{coverage}, \text{coverage})$ [S1, unofficial].
14. Apply depth occlusion: if `relight.occlusionEnabled != 0u` and scene depth $D_{scene} > 0.01$, linearize splat NDC depth to meters and compare with scene depth. If splat is behind scene geometry, suppress coverage [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_stage_splat_vertex_shader(instance_id, vertex_id, amplification_id, v_normal_rough, v_view_refl, v_indirect, v_depth, coverage, scene_depth_m, relight, uniforms_array, splat_index_array, chunks, k_max_view_count):
    # Step 1: Fetch uniforms
    view_idx = min(amplification_id, k_max_view_count)
    uniforms = uniforms_array[view_idx]
    
    # Step 2: Compute global splat ID
    indexed_splat_count = uniforms.indexedSplatCount
    global_splat_id = instance_id * indexed_splat_count + (vertex_id // 4)
    
    # Step 3: Check splat count
    if global_splat_id >= uniforms.splatCount:
        return clip_space_position(1, 1, 0, 1)
    
    # Step 4: Fetch chunk index
    chunk_info_ref = splat_index_array[global_splat_id]
    chunk_index = chunk_info_ref.chunkIndex
    
    # Step 5: Check chunk count
    if chunk_index >= uniforms.chunkCount:
        return clip_space_position(1, 1, 0, 1)
    
    # Step 6: Fetch chunk metadata
    chunk_metadata = chunks[chunk_index]
    
    # Step 7-8: Extract and normalize vectors
    N = normalize(v_normal_rough.rgb)
    V = normalize(v_view_refl.rgb)
    
    # Step 9: Compute roughness and reflection strength
    inv_mw = 1.0 / chunk_metadata.material_weight # Assuming material weight is in chunk metadata or uniforms
    roughness = float(v_normal_rough.a) * inv_mw
    refl_strength = float(v_view_refl.a) * inv_mw
    
    # Step 10: Compute indirect blended color
    indirect_blended = half3(float3(v_indirect) * inv_mw)
    
    # Step 11: Apply tint if enabled
    base_color = v_base_color
    ori_color = v_ori_color
    if relight.tint.w > 0.0f:
        lw = float3(0.2126f, 0.7152f, 0.0722f)
        base_color = half3(recolor_preserving_value(float3(base_color), relight.tint.xyz, lw, relight.tint.w))
        ori_color = half3(recolor_preserving_value(float3(ori_color), relight.tint.xyz, lw, relight.tint.w))
        
    # Step 12: Shade using IBL
    shaded = shade_ibl_deferred(base_color, ori_color, indirect_blended, N, V, roughness, refl_strength, relight, prefiltered_env, irradiance_env, brdf_lut, ibl_sampler)
    
    # Step 13: Debug mode
    if relight.debug_mode != 0:
        return half4(shaded * coverage, coverage)
    
    # Step 14: Depth occlusion
    effective_coverage = coverage
    if relight.occlusion_enabled != 0u and scene_depth_m > 0.01f:
        splat_ndc = float(v_depth) / float(coverage)
        splat_depth_m = relight.depth_linearize.y / max(relight.depth_linearize.x, 1e-6) # Simplified linearization based on source snippet
        if splat_depth_m > scene_depth_m:
            effective_coverage = 0.0
            shaded = half3(0.0)
            
    return clip_space_position(v_position, v_depth), effective_coverage, shaded
```

## Operating characteristics
- **Time Complexity:** $O(1)$ per vertex, but includes texture lookups for IBL and depth.
- **Memory Complexity:** Depends on chunk metadata and environment maps.
- **Strengths:** Reduces fragment shader workload by performing lighting and occlusion early. Enables efficient AR occlusion.
- **Limitations:** Lighting is per-vertex, which may cause artifacts if splats are large and cover varying depths. Precision issues may arise in depth comparison.

## Background
This method is part of the MetalGaussianSplatRelighting project, an unofficial implementation for rendering Gaussian splats with relighting on Metal GPUs. It was developed to integrate physically-based lighting and AR occlusion into the splatting pipeline.

## Sources
[S1] john-rocky/MetalGaussianSplatRelighting, https://github.com/john-rocky/MetalGaussianSplatRelighting, fetched 2026-10-27 (unofficial)
