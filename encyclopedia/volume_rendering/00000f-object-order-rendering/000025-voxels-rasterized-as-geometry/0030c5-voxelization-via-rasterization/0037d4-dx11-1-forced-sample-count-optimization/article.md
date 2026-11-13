# DX11.1 Forced Sample Count Optimization

Also called: Forced Sample Count MSAA Voxelization

## Overview
This method enables Multi-Sample Anti-Aliasing (MSAA) rasterization for voxelization without requiring an MSAA render target to be bound to the output merger stage. It solves the video memory (vidmem) overhead associated with standard MSAA voxelization by leveraging the DirectX 11.1 Forced Sample Count feature [S1, S2, unofficial]. It is distinguished from standard MSAA voxelization by its reduced memory footprint, while remaining an "aggressive" approximation that may produce holes compared to genuine conservative rasterization [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A mesh or set of primitives to be voxelized.
*   **Output:** A voxel grid (typically stored in a structured buffer or texture) populated with primitive IDs or coverage data.

**Data Structures:**
*   **Render Target:** A standard, non-MSAA render target (e.g., 1x MSAA) is used.
*   **Rasterizer State:** Configured with the `ForcedSampleCount` flag set to the desired MSAA level (e.g., 4x or 8x) [S1, unofficial].

**Steps:**
1.  Create a standard render target view (RTV) for the voxelization output buffer. This buffer does **not** need to be allocated as an MSAA texture [S2, unofficial].
2.  Create a rasterizer state object.
3.  Set the `ForcedSampleCount` member of the rasterizer state to the target MSAA sample count (e.g., 4) [S1, unofficial].
4.  Bind the rasterizer state to the pipeline.
5.  Bind the standard (non-MSAA) render target to the Output Merger stage.
6.  Draw the primitives. The GPU rasterizer will invoke the pixel shader once per sample location within each pixel, effectively performing MSAA rasterization logic, but discarding the sample coverage data into the single-sampled render target or using it for conservative rasterization heuristics depending on the specific implementation details of the voxelization shader [S1, S2, unofficial].

**Formulas:**
None specified in sources.

**Parameters:**
*   **MSAA Sample Count:** Typically 4 or 8. Higher counts improve coverage accuracy but increase pixel shader invocation count [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Holes:** This method is not "genuine" conservative rasterization. Primitives that lie in-between subsamples may not be voxelized properly, leading to "holes" in the resulting voxel grid [S1, S2, unofficial].
*   **Mitigation:** Holes are avoidable by ensuring the voxel grid resolution is sufficiently high (fine-grained) relative to the size of the voxelized primitives [S1, S2, unofficial].
*   **Precision:** The method relies on the rasterizer's internal sample coverage calculation. If the primitive is too small relative to the pixel grid, it may be missed entirely if it does not overlap with any of the forced sample points [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Setup Rasterizer State
rasterizer_desc = RasterizerStateDesc()
rasterizer_desc.ForcedSampleCount = 4  # Enable 4x MSAA logic
rasterizer_desc.ScissorEnable = False
rasterizer_desc.MultisampleEnable = True # Note: MultisampleEnable might be ignored if no MSAA RT is bound, but ForcedSampleCount is the key

# Bind Standard (Non-MSAA) Render Target
# This is the critical difference from standard MSAA voxelization
rtv = CreateRenderTargetView(non_msaa_texture)
device_context.OutputMerger.SetTargets(rtv)

# Bind Rasterizer State
device_context.Rasterizer.SetState(rasterizer_desc)

# Draw Primitives
device_context.DrawIndexed(index_count, start_index, base_vertex)
```

## Subtopics

### Conservative Raster via Edge Extension
Uses primitive expansion to ensure coverage. More robust against holes but computationally more expensive per primitive. [Link](2480-3-axis-rasterization-voxelization.md)

### Chunked UV Rasterization with Face ID Storage
Optimizes memory access patterns for large voxel grids. [Link](2598-chunked-uv-rasterization-with-face-id-storage.md)

### Triangle Projection Voxelization
Projects triangles directly into voxel space. [Link](5432-triangle-projection-voxelization.md)

## Operating characteristics
*   **Memory Cost:** Reduces vidmem usage compared to standard MSAA voxelization, which requires allocating an MSAA render target [S2, unofficial].
*   **Performance:** Increases pixel shader invocations by the MSAA sample count factor (e.g., 4x for 4x MSAA) compared to single-sampled rasterization [S1, unofficial].
*   **Accuracy:** Aggressive approximation. It does not guarantee full conservative coverage, potentially leaving gaps ("holes") in the voxel grid [S1, S2, unofficial].
*   **When to use:** When vidmem is constrained and a high-resolution voxel grid can mitigate coverage gaps.
*   **When not to use:** When strict conservative rasterization is required for small primitives or low-resolution voxel grids, as holes are likely [S2, unofficial].

## Background
This technique was described in NVIDIA's "The Basics of GPU Voxelization" documentation, highlighting the use of DirectX 11.1 features to optimize the voxelization pipeline [S1, S2, unofficial]. It addresses the memory overhead of traditional MSAA-based voxelization methods.

## Sources
[S1] 【转】The Basics of GPU Voxelization - 竹庭, https://www.cnblogs.com/hustztz/p/7575729.html, fetched 2026-10-27 (unofficial)
[S2] https://developer.nvidia.com/content/basics-gpu-voxelization, https://developer.nvidia.com/content/basics-gpu-voxelization, fetched 2026-10-28 (unofficial)
