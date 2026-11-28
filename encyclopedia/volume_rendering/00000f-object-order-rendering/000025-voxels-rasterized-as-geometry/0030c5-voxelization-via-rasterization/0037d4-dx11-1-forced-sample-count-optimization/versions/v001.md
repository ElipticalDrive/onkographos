# DX11.1 Forced Sample Count Optimization

Also called: Forced Sample Count MSAA Voxelization

## Overview
This method enables Multi-Sample Anti-Aliasing (MSAA) rasterization behavior for voxelization without requiring the allocation and binding of an MSAA render target. By leveraging the DirectX 11.1 "Forced Sample Count" feature, it achieves the coverage benefits of MSAA rasterization while reducing video memory (vidmem) consumption compared to standard MSAA voxelization techniques [S1, S2, unofficial]. It serves as a memory-efficient alternative to binding an MSAA render target for conservative rasterization effects [S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A mesh or primitive set to be voxelized; a target voxel grid (texture or buffer).
- **Output:** Voxel occupancy data in the target grid.

**Data Structures:**
- **Render Target:** A standard single-sample render target (or depth/stencil buffer) is used. No MSAA render target is allocated [S1, S2, unofficial].
- **Rasterizer State:** Configured with the `ForcedSampleCount` flag set to the desired number of samples (e.g., 4 or 8) [S1, S2, unofficial].

**Steps:**
1. Configure the DirectX 11.1 rasterizer state to enable MSAA rasterization by setting the `ForcedSampleCount` parameter to $N$ samples, where $N > 1$ [S1, S2, unofficial].
2. Bind a standard (non-MSAA) render target to the output merger stage [S1, S2, unofficial].
3. Issue draw calls for the geometry. The GPU rasterizer evaluates coverage for each primitive against the $N$ subsamples per pixel [S1, S2, unofficial].
4. In the pixel shader, write voxel data based on the rasterization coverage. Since no MSAA render target is bound, the shader writes to a single-sample location, but the rasterizer's coverage mask determines which pixels/subsamples are active [S1, S2, unofficial].
5. Resolve or accumulate the results into the voxel grid [S1, S2, unofficial].

**Parameters:**
- **ForcedSampleCount:** The number of subsamples to use for rasterization coverage. Typical values are 4 or 8 [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
- **Holes:** This method is not "genuine" conservative rasterization. Primitives that lie in-between subsamples may not be voxelized properly, leading to "holes" in the voxelization [S1, S2, unofficial].
- **Mitigation:** To avoid holes, the voxel grid resolution must be sufficiently high (fine-grained) relative to the size of the voxelized primitives [S1, S2, unofficial].
- **Precision:** Standard rasterization precision applies; this method relies on MSAA coverage rather than explicit conservative rasterization algorithms [S1, S2, unofficial].

## Operating characteristics
- **Memory:** Reduces vidmem cost compared to standard MSAA voxelization, which requires an MSAA render target [S2, unofficial]. The memory savings come from not allocating the multi-sample buffer [S1, S2, unofficial].
- **Performance:** Performance impact depends on the forced sample count. MSAA rasterization is generally more expensive than single-sample rasterization but cheaper than full MSAA rendering with resolve [S1, S2, unofficial].
- **Accuracy:** Less accurate than genuine conservative rasterization. It is described as a "relatively aggressive method" due to the potential for holes [S1, S2, unofficial].
- **When to use:** When vidmem is constrained and a high-resolution voxel grid can mitigate coverage holes [S1, S2, unofficial].
- **When not to use:** When strict conservative rasterization is required and holes are unacceptable, or when the voxel grid resolution is too low to prevent gaps [S1, S2, unofficial].

## Background
This technique was introduced in the context of GPU voxelization optimization, specifically addressing the memory overhead of MSAA-based voxelization. It leverages features available in DirectX 11.1 [S1, S2, unofficial].

## Sources
[S1] 【转】The Basics of GPU Voxelization - 竹庭, https://www.cnblogs.com/hustztz/p/7575729.html, fetched 2026-10-27 (unofficial)
[S2] https://developer.nvidia.com/content/basics-gpu-voxelization, https://developer.nvidia.com/content/basics-gpu-voxelization, fetched 2026-10-28 (unofficial)
