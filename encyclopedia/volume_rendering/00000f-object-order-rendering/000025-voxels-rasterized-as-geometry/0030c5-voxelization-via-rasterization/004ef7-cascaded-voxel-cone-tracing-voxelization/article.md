# Cascaded Voxel Cone Tracing Voxelization

Also called: Geometry Volume (GV) Voxelization

## Overview
Cascaded Voxel Cone Tracing Voxelization is a technique that renders scene geometry into a 3D texture (Geometry Volume, GV) using an orthographic camera aligned with the light direction, organized into cascades to handle varying distances and resolutions [S1, unofficial]. It differs from sibling methods by explicitly separating the voxelization into multiple cascaded volumes, where each cascade covers a volume twice as large as the previous one, allowing for high-resolution detail near the light source and coarser resolution at a distance [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene geometry, a directional light (defining view direction), `gridResolution` (integer, e.g., 64 or 128), `gridPixelSize` (world units), `gridCascades` (integer), and a culling mask [S1, unofficial].
*   **Outputs:** A 3D RenderTexture `_tex3d_gv` containing the voxelized geometry data [S1, unofficial].

**Data Structures:**
*   `CVCT_gv`: A `RWStructuredBuffer<int>` of size $N^3$ (where $N = \text{gridResolution}$). Used as an intermediate buffer during rasterization to accumulate voxel data before packing [S1, unofficial].
*   `_tex3d_gv`: A 3D RenderTexture with dimensions $W=N$, $H=N \times C$, $D=N$ (where $C = \text{gridCascades}$). Format: R8. Dimension: Tex3D. WrapMode: Clamp. FilterMode: Bilinear. No mipmaps [S1, unofficial].
*   `dummy_target`: A temporary R8 RenderTexture of size $N \times N$, used as the camera's target texture to drive the render pipeline while data is written to `CVCT_gv` via random write targets [S1, unofficial].

**Steps:**

1.  **Initialize Camera:** Create or reuse an orthographic camera aligned with the directional light's transform. Set `orthographic = true`, `aspect = 1`, `clearFlags = Nothing`, and `stereoTargetEye = None` [S1, unofficial].
2.  **Allocate Buffers:** Allocate `CVCT_gv` with size $N^3$ integers. Allocate `_tex3d_gv` with dimensions $(N, N \times C, N)$ in R8 format [S1, unofficial].
3.  **Compute Light-Space Matrix:** Calculate `_world_to_light_local_matrix`. This matrix maps world coordinates to light-local space such that cascade 0 spans $(-1, -1, -1)$ to $(+1, +1, +1)$ [S1, unofficial].
    $$
    sz = 0.5 \times N \times \text{gridPixelSize}
    $$
    $$
    \text{Matrix} = \text{Scale}\left(\frac{1}{sz}\right) \times \text{cam.transform.worldToLocalMatrix}
    $$
4.  **Loop over Cascades ($i = 0$ to $C-1$):**
    a.  **Calculate Cascade Bounds:** Determine the half-size for the current cascade.
        $$
        \text{half\_size} = 0.5 \times N \times \text{gridPixelSize} \times 2^i
        $$
    b.  **Configure Camera:** Set `cam.orthographicSize = half_size`, `cam.nearClipPlane = -half_size`, `cam.farClipPlane = half_size` [S1, unofficial].
    c.  **Clear Buffer:** Dispatch `ClearKernel` to zero out `CVCT_gv`. Thread group count: $((N^3 \times 63) / 64, 1, 1)$ [S1, unofficial].
    d.  **Render Geometry:** Set `cam.targetTexture = dummy_target`. Set random write target 1 to `CVCT_gv`. Render scene using `gvShader` [S1, unofficial].
    e.  **Repack Data:** Dispatch repacking kernel.
        *   If $i=0$: Dispatch `RepackLvl0Kernel`.
        *   If $i>0$: Dispatch `RepackAndUpscaleKernel`. Set `GridCascadeBase` to $(i \times N, (i-1) \times N)$ for upscale kernels [S1, unofficial].
        *   Thread group count: $((N+3)/4, (N+3)/4, (N+3)/4)$ [S1, unofficial].
5.  **Cleanup:** Release `dummy_target` and `CVCT_gv` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cvct_voxelize(grid_resolution, grid_pixel_size, grid_cascades, directional_light_transform, gv_shader, gv_compute):
    N = grid_resolution
    C = grid_cascades
    
    # Allocate intermediate buffer and final texture
    cb_gv = ComputeBuffer(N * N * N, 4) # RWStructuredBuffer<int>
    tex3d_gv = RenderTexture(N, N * C, N, RenderTextureFormat.R8)
    tex3d_gv.wrapMode = TextureWrapMode.Clamp
    tex3d_gv.filterMode = TextureFilterMode.Bilinear
    
    # Create orthographic camera aligned with light
    cam = CreateOrthographicCamera()
    cam.transform.position = directional_light_transform.position
    cam.transform.rotation = directional_light_transform.rotation
    cam.orthographic = True
    cam.aspect = 1.0
    cam.clearFlags = CameraClearFlags.Nothing
    cam.stereoTargetEye = StereoTargetEyeMask.None
    
    # Pre-calculate light-space matrix for cascade 0 bounds (-1,-1,-1) to (1,1,1)
    sz = 0.5 * N * grid_pixel_size
    world_to_light_local_matrix = Matrix4x4.Scale(Vector3.one / sz) * cam.transform.worldToLocalMatrix
    
    dummy_target = RenderTexture.GetTemporary(N, N, 0, RenderTextureFormat.R8)
    
    clear_kernel = gv_compute.FindKernel("ClearKernel")
    repack_lvl0_kernel = gv_compute.FindKernel("RepackLvl0Kernel")
    repack_upscale_kernel = gv_compute.FindKernel("RepackAndUpscaleKernel")
    
    gv_compute.SetInt("GridResolution", N)
    gv_compute.SetInt("GridHalfResolution", N // 2)
    
    for i in range(C):
        # Calculate cascade volume size
        half_size = 0.5 * N * grid_pixel_size * (2 ** i)
        
        # Configure camera frustum
        cam.orthographicSize = half_size
        cam.nearClipPlane = -half_size
        cam.farClipPlane = half_size
        
        # Step 1: Clear intermediate buffer
        thread_groups_clear = (N * N * N * 63) // 64
        gv_compute.SetBuffer(clear_kernel, "CVCT_gv", cb_gv)
        gv_compute.Dispatch(clear_kernel, thread_groups_clear, 1, 1)
        
        # Step 2: Render scene into intermediate buffer
        cam.targetTexture = dummy_target
        Graphics.SetRandomWriteTarget(1, cb_gv)
        cam.RenderWithShader(gv_shader, "RenderType")
        
        # Step 3: Repack from buffer to 3D texture
        thread_groups = (N + 3) // 4
        
        if i == 0:
            gv_compute.SetTexture(repack_lvl0_kernel, "CVCT_tex3d", tex3d_gv)
            gv_compute.SetBuffer(repack_lvl0_kernel, "CVCT_gv", cb_gv)
            gv_compute.Dispatch(repack_lvl0_kernel, thread_groups, thread_groups, thread_groups)
        else:
            gv_compute.SetInts("GridCascadeBase", i * N, (i - 1) * N)
            gv_compute.SetTexture(repack_upscale_kernel, "CVCT_tex3d", tex3d_gv)
            gv_compute.SetBuffer(repack_upscale_kernel, "CVCT_gv", cb_gv)
            gv_compute.Dispatch(repack_upscale_kernel, thread_groups, thread_groups, thread_groups)
            
    # Cleanup
    RenderTexture.ReleaseTemporary(dummy_target)
    cb_gv.Release()
    
    return world_to_light_local_matrix, tex3d_gv
```

## Subtopics

### Cascaded Voxel Cone Tracing (CVCT) Tracing
A tracing technique that samples the Light Volume (LightTex3d) to compute indirect lighting for each voxel in the scene [S1, unofficial]. It processes each cascade $i$ by dispatching `TracingKernel` with thread group count $((N+3)/4, (N+3)/4, (N+3)/4)$ and setting `GridCascadeBase` to $(i \times N, (C-1) \times N)$ [S1, unofficial]. It uses a `LightTex3d` 3D RenderTexture with the same dimensions and format as the Geometry Volume [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels per cascade and the number of cascades. The voxelization step involves one render pass per cascade and one compute dispatch per cascade.
*   **Memory Complexity:** $O(N^3 \times C)$ for the 3D texture storage. The intermediate buffer is $O(N^3)$ per cascade iteration (reused).
*   **Strengths:** Handles large scenes efficiently by varying resolution per cascade. Orthographic projection ensures uniform voxel size within each cascade.
*   **Limitations:** Requires multiple render passes (one per cascade). The repacking step adds compute overhead. Precision issues may arise at cascade boundaries if not handled correctly by the upscale kernel [S1, unofficial].
*   **When to use:** When indirect lighting via voxel cone tracing is required for large scenes with varying detail requirements.
*   **When not to use:** For small scenes where a single resolution grid suffices, or when hardware does not support compute shaders or 3D textures.

## Background
Originated in the "Cascaded Voxel Cone Tracing" implementation by arigo [S1, unofficial]. The technique is part of the broader family of voxel-based global illumination methods, building on the principles of voxelization via rasterization [G]. It was developed to address the performance and memory constraints of single-resolution voxel grids in large-scale real-time rendering.

## Sources
[S1] arigo/CascatedVoxelConeTracing, https://github.com/arigo/CascatedVoxelConeTracing, fetched 2026-10-26 (unofficial)
