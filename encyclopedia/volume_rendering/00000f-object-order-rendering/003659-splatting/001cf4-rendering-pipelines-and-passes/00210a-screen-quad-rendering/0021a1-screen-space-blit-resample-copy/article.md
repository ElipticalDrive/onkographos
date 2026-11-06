# Screen-space blit / Resample copy

Also called: Fullscreen Blit with Crosshair, Resample copy, Screen-space blit, blit.wgsl

## Overview
This method performs a screen-space copy (blit) of a source texture, typically a compute-shader output, into a target render target [S2, unofficial]. It simultaneously overlays a debug crosshair and forces the output alpha to 1.0 to prevent compositing artifacts on certain windowing systems like Wayland [S1, unofficial]. It differs from a standard blit by injecting pixel-level crosshair logic and alpha forcing directly in the fragment shader [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Input:** A source texture accessed via a Shader Resource View (SRV) bound at slot `uSRVSlot` [S2, unofficial].
*   **Output:** A target render target view (RTV) identified by slice `uSlice` and mip level `uMip`, cleared to transparent before drawing [S2, unofficial].
*   **Parameters:** `pSRVSrc` (source SRV handle), `pShader` (shader object containing vertex and pixel shaders), `uSRVSlot`, `uSlice`, and `uMip` [S2, unofficial].

### Data Structures
The implementation relies on Direct3D 11 API objects for state management, including render target views, depth stencil views, and viewport structures [S2, unofficial].

### Numbered Steps
1.  **Context Acquisition:** Get the immediate device context (`pDXContext`) from the device [S2, unofficial].
2.  **Save State:** Retrieve the current render target view (`pRTVBack`) and depth stencil view (`pDSVBack`) using `OMGetRenderTargets` [S2, unofficial].
3.  **Set Target:** Bind the target render target `m_vvpRTVs[uSlice][uMip]` using `OMSetRenderTargets` [S2, unofficial].
4.  **Clear:** Clear the target render target to transparent using `ClearRenderTargetView` [S2, unofficial].
5.  **Set Viewport:** Create a viewport matching the texture dimensions (`VpDepth`) and set it using `RSSetViewports` [S2, unofficial]. Save the previous viewport (`VpBack`) for restoration [S2, unofficial].
6.  **Bind Resources:** Bind the source SRV (`pSRVSrc`) at slot `uSRVSlot` using `PSSetShaderResources` [S2, unofficial].
7.  **Set Topology:** Set the primitive topology to `D3D11_PRIMITIVE_TOPOLOGY_TRIANGLESTRIP` [S2, unofficial].
8.  **Bind Shaders:** Set the vertex shader `g_uVSScreenQuad` using `VSSetShader` and the pixel shader `g_uPSResample` using `PSSetShader` [S2, unofficial].
9.  **Draw:** Issue a draw call for 3 vertices using `Draw(3, 0)` [S2, unofficial].
10. **Unbind:** Unbind the source SRV, vertex shader, and pixel shader by setting them to `nullptr` (or null SRV) at their respective slots [S2, unofficial].
11. **Restore State:** Restore the original viewport using `RSSetViewports` with `VpBack` [S2, unofficial].
12. **Restore Targets:** Restore the original render and depth-stencil targets using `OMSetRenderTargets` [S2, unofficial].

### Vertex Shader
The vertex shader renders a single oversized triangle using vertex indices 0, 1, 2 [S1, unofficial]. Positions are `(-1.0, -1.0)`, `(3.0, -1.0)`, `(-1.0, 3.0)` [S1, unofficial]. UVs are `(0.0, 1.0)`, `(2.0, 1.0)`, `(0.0, -1.0)` [S1, unofficial]. This covers the entire screen and extends beyond to ensure full coverage [S1, unofficial].

### Fragment Shader
The fragment shader samples the source texture at the interpolated UV coordinate [S1, unofficial]. It calculates screen dimensions and the absolute offset in pixels from the screen center `(0.5, 0.5)` [S1, unofficial]. It determines if the pixel is inside the crosshair arms: Horizontal arm checks `offset_px.x <= 12.0 AND offset_px.y <= 2.0` (outer) or `<= 1.0` (inner); Vertical arm checks `offset_px.y <= 12.0 AND offset_px.x <= 2.0` (outer) or `<= 1.0` (inner) [S1, unofficial]. It returns white `(1.0, 1.0, 1.0, 1.0)` for inner crosshair, black `(0.0, 0.0, 0.0, 1.0)` for outer crosshair, and the sampled color with alpha forced to `1.0` otherwise [S1, unofficial]. The alpha forcing prevents alpha-compositing issues on Wayland when TAA resolve smuggles grass-blade markers in the alpha channel [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_target_populate(pSRVSrc, pShader, uSRVSlot, uSlice, uMip):
    # 1. Get immediate device context
    pDXContext = get_immediate_device_context()

    # 2. Save current render targets
    pRTVBack = CPDXRenderTargetView()
    pDSVBack = CPDXDepthStencilView()
    pDXContext.OMGetRenderTargets(1, pRTVBack, pDSVBack)

    # 3. Set target render target
    target_rtv = m_vvpRTVs[uSlice][uMip]
    pDXContext.OMSetRenderTargets(1, target_rtv, None)

    # 4. Clear target to transparent
    pDXContext.ClearRenderTargetView(target_rtv, Colors.Transparent)

    # 5. Set viewport to texture dimensions
    uNumViewports = 1
    VpBack = D3D11_VIEWPORT()
    pDXContext.RSGetViewports(uNumViewports, VpBack)
    VpDepth = CD3D11_VIEWPORT(m_pTexture, target_rtv)
    pDXContext.RSSetViewports(uNumViewports, VpDepth)

    # 6. Bind source SRV
    pDXContext.PSSetShaderResources(uSRVSlot, 1, pSRVSrc)

    # 7. Set primitive topology
    pDXContext.IASetPrimitiveTopology(D3D11_PRIMITIVE_TOPOLOGY_TRIANGLESTRIP)

    # 8. Bind shaders
    vs = pShader.GetVertexShader(g_uVSScreenQuad)
    ps = pShader.GetPixelShader(g_uPSResample)
    pDXContext.VSSetShader(vs, None, 0)
    pDXContext.GSSetShader(None, None, 0)
    pDXContext.PSSetShader(ps, None, 0)

    # 9. Draw 3 vertices
    pDXContext.Draw(3, 0)

    # 10. Unbind resources
    pNullSRV = LPDXShaderResourceView(None)
    pDXContext.PSSetShaderResources(uSRVSlot, 1, pNullSRV)
    pDXContext.VSSetShader(None, None, 0)
    pDXContext.PSSetShader(None, None, 0)

    # 11. Restore viewport
    pDXContext.RSSetViewports(uNumViewports, VpBack)

    # 12. Restore render targets
    pDXContext.OMSetRenderTargets(1, pRTVBack, pDSVBack)
```

### Fragment Shader Logic Detail
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fragment_shader(uv: vec2<f32>, screen_dims: vec2<f32>, source_texture: Texture2D) -> vec4<f32>:
    # Sample source
    color = source_texture.sample(uv)
    
    # Calculate pixel offset from center
    center = vec2<f32>(0.5, 0.5)
    pixel_pos = uv * screen_dims
    offset_px = abs(pixel_pos - center * screen_dims)
    
    # Check crosshair arms
    # Horizontal arm
    h_outer = offset_px.x <= 12.0 and offset_px.y <= 2.0
    h_inner = offset_px.x <= 12.0 and offset_px.y <= 1.0
    
    # Vertical arm
    v_outer = offset_px.y <= 12.0 and offset_px.x <= 2.0
    v_inner = offset_px.y <= 12.0 and offset_px.x <= 1.0
    
    is_inner = h_inner or v_inner
    is_outer = (h_outer or v_outer) and not is_inner
    
    if is_inner:
        return vec4<f32>(1.0, 1.0, 1.0, 1.0)
    elif is_outer:
        return vec4<f32>(0.0, 0.0, 0.0, 1.0)
    else:
        # Force alpha to 1.0 to prevent TAA resolve issues on Wayland
        return vec4<f32>(color.rgb, 1.0)
```

### Edge Cases and Pitfalls
*   **Wayland TAA Artifacts:** Failing to force alpha to 1.0 can result in grass-blade markers from the TAA resolve process leaking into the alpha channel, causing compositing issues [S1, unofficial].
*   **Viewport Mismatch:** The viewport must be set to the texture dimensions, not the window dimensions, to ensure correct sampling coordinates [S2, unofficial].
*   **State Restoration:** Failing to restore the original render targets and viewport can corrupt subsequent rendering passes [S2, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Time:** O(1) relative to scene complexity; O(N) relative to screen resolution (N = number of pixels). The cost is dominated by the fragment shader execution over the screen area [G].
*   **Memory:** O(1) additional memory for state storage (backups of RTV/DSV/Viewport). The source and target textures occupy their respective memory [G].

### Measured Performance
*   **Hardware:** Tested on Direct3D 11 hardware [S2, unofficial].
*   **Resolution:** Performance scales linearly with screen resolution [G].
*   **Scene:** Independent of scene geometry or voxel count, as it operates on screen-space textures [G].

### Strengths and Limitations
*   **Strengths:**
    *   Efficient screen-space data transfer [G].
    *   Built-in debug visualization (crosshair) [S1, unofficial].
    *   Avoids CPU-GPU synchronization overhead by using immediate context commands [S2, unofficial].
*   **Limitations:**
    *   Specific to Direct3D 11 API in the provided implementation [S2, unofficial].
    *   Crosshair logic is hardcoded and not configurable via parameters in the base method [S1, unofficial].
    *   Alpha forcing may be undesirable for non-debug blits [S1, unofficial].

### When to Use
*   Use when transferring compute shader output to a render target for display or further processing [S2, unofficial].
*   Use when a debug crosshair is needed for screen-space alignment [S1, unofficial].
*   Use when targeting Wayland compositors to avoid TAA-related alpha artifacts [S1, unofficial].

### When Not to Use
*   Do not use for general-purpose texture copying where alpha preservation is critical and the Wayland TAA issue is not present [S1, unofficial].
*   Do not use if the crosshair overlay is not desired [S1, unofficial].

## Background

### Origin
The method originates from the `voxelG` and `VoxelizerX` projects, which are unofficial implementations of voxel rendering techniques [S1, unofficial][S2, unofficial]. The specific blit implementation with crosshair overlay is documented in the `RenderTarget::Populate` function of `VoxelizerX` [S2, unofficial] and the associated shader code in `voxelG` [S1, unofficial].

### History and Development
The technique is part of the rendering pipeline for screen-space blitting in voxel renderers [S2, unofficial]. The crosshair overlay was added to assist with debugging screen-space coordinates and alignment [S1, unofficial]. The alpha forcing was introduced to address specific artifacts observed on Wayland compositors with TAA [S1, unofficial].

### Usage
Used in voxel rendering pipelines for transferring data between render targets and providing debug visualization [S1, unofficial][S2, unofficial].

## Sources

[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S2] StarsX/VoxelizerX, https://github.com/StarsX/VoxelizerX, fetched 2026-11-01 (unofficial)
