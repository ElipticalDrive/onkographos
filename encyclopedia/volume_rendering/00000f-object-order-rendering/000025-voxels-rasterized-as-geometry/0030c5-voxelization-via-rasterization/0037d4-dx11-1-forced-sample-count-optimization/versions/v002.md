# DX11.1 Forced Sample Count Optimization

Also called: Forced Sample Count MSAA Voxelization

## Overview
This method leverages the DirectX 11.1 "Forced Sample Count" feature to perform Multi-Sample Anti-Aliasing (MSAA) rasterization without requiring a bound MSAA render target [S1, S2, unofficial]. It serves as a memory-efficient alternative to standard MSAA voxelization, which necessitates allocating an MSAA render target in video memory (vidmem) [S2, unofficial]. By decoupling the rasterization sample count from the render target format, it reduces vidmem overhead while providing coverage-based voxelization [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of voxel primitives (typically cubes or quads representing occupied voxels) and a target voxel grid resolution.
*   **Output:** A voxel grid (scalar field) populated with occupancy data.

**Data Structures:**
*   **Render Target:** A standard single-sampled (non-MSAA) render target (e.g., `DXGI_FORMAT_R8G8B8A8_UNORM` or similar format suitable for the shader output).
*   **Voxel Grid:** A 3D texture or structured buffer representing the voxel space, updated via the pixel shader.

**Steps:**
1.  **Configure Rasterizer State:** Create a `D3D11_RASTERIZER_DESC` structure. Set `SampleDesc.Count` to the desired number of samples (e.g., 4 or 8) and `SampleDesc.Quality` to 0. Crucially, enable the `ForcedSampleCount` flag (or equivalent driver-specific extension/feature) if available, or rely on the DX11.1 capability to override sample counts during draw calls [S1, S2, unofficial].
2.  **Bind Render Target:** Bind a standard, non-MSAA render target view (RTV) to the output merger stage [S1, S2, unofficial].
3.  **Draw Primitives:** Issue draw calls for the voxel primitives. The rasterizer will generate fragments for each sample location defined by the MSAA sample mask, even though the final render target is single-sampled [S1, S2, unofficial].
4.  **Pixel Shader Execution:** The pixel shader executes for each covered sample. Inside the shader, determine which voxel grid cell each sample falls into.
5.  **Atomic Update:** Use atomic operations to update the voxel grid. For each sample, calculate the voxel index $(i, j, k)$ and perform an atomic OR or SET operation to mark the voxel as occupied [G].
6.  **Resolve:** Since no MSAA render target is bound, no resolve pass is needed. The single-sampled target contains the final aggregated result (or the shader handles aggregation per sample directly into the voxel grid) [S1, S2, unofficial].

**Formulas:**
*   **Sample Coverage:** A primitive covers a sample if the sample point lies within the primitive's bounds. The rasterizer determines this coverage automatically [S1, S2, unofficial].
*   **Voxel Index:** For a sample at screen position $(x, y)$, the corresponding voxel index is derived from the fragment's world-space position transformed into voxel space [G].

**Parameters:**
*   **Sample Count:** Typically 4 or 8, depending on the desired anti-aliasing quality and performance trade-off [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Holes:** This method is described as "relatively aggressive" and is not "genuine conservative rasterization" [S1, S2, unofficial]. Primitives that lie in-between subsamples may not be voxelized properly, leading to "holes" in the voxel grid [S1, S2, unofficial].
*   **Mitigation:** To avoid holes, ensure the voxel grid resolution is sufficiently high (fine-grained) relative to the size of the voxelized primitives [S1, S2, unofficial].
*   **Precision:** Standard rasterization precision applies; sub-pixel precision is limited by the sample count [S1, S2, unofficial].

## Operating characteristics
*   **Memory:** Reduces vidmem cost compared to standard MSAA voxelization by avoiding the allocation of an MSAA render target [S2, unofficial].
*   **Performance:** Performance depends on the number of samples and the complexity of the pixel shader. The overhead of MSAA rasterization is incurred, but the memory bandwidth cost of resolving an MSAA target is saved [S1, S2, unofficial].
*   **Quality:** Provides better coverage than conventional rasterization but is less robust than true conservative rasterization [S1, S2, unofficial].
*   **When to use:** When vidmem is constrained and a moderate level of coverage accuracy is acceptable, provided the voxel grid is high-resolution enough to mitigate holes [S1, S2, unofficial].
*   **When not to use:** When strict conservative rasterization is required to prevent holes, or when the voxel grid resolution is too low to handle the aggressive nature of this method [S1, S2, unofficial].

## Background
This technique was discussed in the context of GPU voxelization basics, highlighting the trade-offs between different rasterization methods [S1, S2, unofficial]. It was presented as a practical optimization for DirectX 11.1 hardware, allowing developers to utilize MSAA coverage benefits without the associated memory penalty [S1, S2, unofficial].

## Sources
[S1] 【转】The Basics of GPU Voxelization - 竹庭, https://www.cnblogs.com/hustztz/p/7575729.html, fetched 2026-10-27 (unofficial)
[S2] https://developer.nvidia.com/content/basics-gpu-voxelization, https://developer.nvidia.com/content/basics-gpu-voxelization, fetched 2026-10-28 (unofficial)
