# Cascaded Voxel Cone Tracing Voxelization

Also called: Geometry Volume (GV) Voxelization

## Overview
Cascaded Voxel Cone Tracing Voxelization is a technique that renders scene geometry into a 3D texture (Geometry Volume, GV) using an orthographic camera aligned with the light direction, organized into cascades to handle varying distances and resolutions [S1, unofficial]. It differs from sibling methods by explicitly managing spatial coherence through a cascade hierarchy where each level covers a volume twice as large as the previous one, enabling efficient indirect lighting computation via cone tracing [S1, unofficial].

## How it works
**Inputs:**
- Scene geometry (meshes).
- Directional light transform (position and rotation).
- Parameters: `gridResolution` (integer, e.g., 64 or 128), `gridPixelSize` (world units per voxel), `gridCascades` (integer count).
- Culling mask for the projection camera.

**Outputs:**
- `_tex3d_gv`: A 3D RenderTexture representing the Geometry Volume.
- `_world_to_light_local_matrix`: A transformation matrix mapping world coordinates to light-local space.

**Data Structures:**
- `CVCT_gv`: A `RWStructuredBuffer<int>` of size `gridResolution^3`. Used as an intermediate buffer during rasterization before packing into the 3D texture.
- `_tex3d_gv`: A 3D RenderTexture (`Tex3D`) with dimensions:
  - Width: `gridResolution`
  - Height: `gridResolution * gridCascades`
  - Depth: `gridResolution`
  - Format: `R8`
  - WrapMode: `Clamp`
  - FilterMode: `Bilinear`
  - No mipmaps.

**Steps:**

1. **Initialize Projection Camera:**
   Create or reuse an orthographic camera (`_projectCam`) with `aspect = 1`, `clearFlags = Nothing`, and `stereoTargetEye = None`. Align its position and rotation with the directional light's transform. Set its culling mask.

2. **Allocate Buffers:**
   Allocate `cb_gv` as a `ComputeBuffer` of type `RWStructuredBuffer<int>` with size `gridResolution * gridResolution * gridResolution`.
   Allocate `dummy_target` as a temporary `RenderTexture` with dimensions `gridResolution x gridResolution`, format `R8`, and no depth buffer.

3. **Configure Compute Shaders:**
   Identify kernels: `ClearKernel`, `RepackLvl0Kernel`, `RepackAndUpscaleKernel`.
   Set global shader integer `_CVCT_GridResolutionI` to `gridResolution`.
   Set compute shader integers: `GridResolution` and `GridHalfResolution` (`gridResolution / 2`).
   Bind `cb_gv` to `CVCT_gv` in all kernels.
   Bind `_tex3d_gv` to `CVCT_tex3d` in repack kernels.

4. **Iterate Over Cascades (i from 0 to gridCascades-1):**
   
   a. **Calculate Cascade Bounds:**
      Compute `half_size`:
      $$ \text{half\_size} = 0.5 \times \text{gridResolution} \times \text{gridPixelSize} \times 2^i $$
      
   b. **Clear Intermediate Buffer:**
      Calculate thread group count for clearing:
      $$ \text{thread\_groups} = \left\lfloor \frac{\text{gridResolution}^3 \times 63}{64} \right\rfloor $$
      Dispatch `ClearKernel` with `(thread_groups, 1, 1)`. This zeros out `cb_gv`.

   c. **Render Geometry:**
      Configure the projection camera:
      - `orthographicSize = half_size`
      - `nearClipPlane = -half_size`
      - `farClipPlane = half_size`
      - `targetTexture = dummy_target` (used to drive the render pipeline, but data is written to the buffer).
      
      Set random write target:
      ```python
      Graphics.SetRandomWriteTarget(1, cb_gv)
      ```
      
      Render the scene using `gvShader`. The fragment shader writes voxel occupancy data into `cb_gv`.

   d. **Repack into 3D Texture:**
      Calculate thread group count for repacking:
      $$ \text{thread\_groups} = \left\lceil \frac{\text{gridResolution} + 3}{4} \right\rceil $$
      
      If `i > 0`:
      Set compute shader integer array `GridCascadeBase` to `(i * gridResolution, (i - 1) * gridResolution)`.
      
      Dispatch the appropriate repack kernel:
      - If `i == 0`: Dispatch `RepackLvl0Kernel` with `(thread_groups, thread_groups, thread_groups)`.
      - If `i > 0`: Dispatch `RepackAndUpscaleKernel` with `(thread_groups, thread_groups, thread_groups)`.

5. **Compute World-to-Light Matrix:**
   After processing all cascades, release `dummy_target` and `cb_gv`.
   Calculate the scale factor:
   $$ \text{sz} = 0.5 \times \text{gridResolution} \times \text{gridPixelSize} $$
   
   Construct the matrix `_world_to_light_local_matrix`:
   $$ \text{mat} = \text{Scale}\left(\frac{1}{\text{sz}}\right) \times \text{cam.transform.worldToLocalMatrix} $$
   
   This matrix maps world coordinates such that cascade 0 spans the box from $(-1, -1, -1)$ to $(+1, +1, +1)$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_voxelization(gridResolution, gridPixelSize, gridCascades, directionalLight, gvShader, gvCompute, _tex3d_gv):
    # 1. Initialize Camera
    cam = get_or_create_orthographic_camera()
    cam.transform.position = directionalLight.transform.position
    cam.transform.rotation = directionalLight.transform.rotation
    cam.orthographic = True
    cam.aspect = 1
    cam.clearFlags = CameraClearFlags.Nothing
    cam.stereoTargetEye = StereoTargetEyeMask.None
    
    # 2. Allocate Buffers
    cb_gv = ComputeBuffer(gridResolution**3, 4) # RWStructuredBuffer<int>
    dummy_target = RenderTexture.GetTemporary(gridResolution, gridResolution, 0, RenderTextureFormat.R8)
    
    # 3. Configure Compute Shaders
    clear_kernel = gvCompute.FindKernel("ClearKernel")
    repack_lvl0_kernel = gvCompute.FindKernel("RepackLvl0Kernel")
    repack_upscale_kernel = gvCompute.FindKernel("RepackAndUpscaleKernel")
    
    gvCompute.SetInt("GridResolution", gridResolution)
    gvCompute.SetInt("GridHalfResolution", gridResolution // 2)
    
    gvCompute.SetBuffer(clear_kernel, "CVCT_gv", cb_gv)
    gvCompute.SetBuffer(repack_lvl0_kernel, "CVCT_gv", cb_gv)
    gvCompute.SetTexture(repack_lvl0_kernel, "CVCT_tex3d", _tex3d_gv)
    gvCompute.SetBuffer(repack_upscale_kernel, "CVCT_gv", cb_gv)
    gvCompute.SetTexture(repack_upscale_kernel, "CVCT_tex3d", _tex3d_gv)
    
    Shader.SetGlobalInt("_CVCT_GridResolutionI", gridResolution)
    
    # 4. Iterate Cascades
    for i in range(gridCascades):
        half_size = 0.5 * gridResolution * gridPixelSize * (2.0 ** i)
        
        # Clear
        thread_groups_clear = (gridResolution**3 * 63) // 64
        gvCompute.Dispatch(clear_kernel, thread_groups_clear, 1, 1)
        
        # Render
        cam.orthographicSize = half_size
        cam.nearClipPlane = -half_size
        cam.farClipPlane = half_size
        cam.targetTexture = dummy_target
        Graphics.SetRandomWriteTarget(1, cb_gv)
        cam.RenderWithShader(gvShader, "RenderType")
        
        # Repack
        thread_groups_repack = (gridResolution + 3) // 4
        if i > 0:
            gvCompute.SetInts("GridCascadeBase", i * gridResolution, (i - 1) * gridResolution)
        
        kernel = repack_lvl0_kernel if i == 0 else repack_upscale_kernel
        gvCompute.Dispatch(kernel, thread_groups_repack, thread_groups_repack, thread_groups_repack)
        
    # 5. Cleanup and Matrix
    RenderTexture.ReleaseTemporary(dummy_target)
    cb_gv.Release()
    
    sz = 0.5 * gridResolution * gridPixelSize
    scale_matrix = Matrix4x4.Scale(Vector3.one / sz)
    _world_to_light_local_matrix = scale_matrix * cam.transform.worldToLocalMatrix
    
    return _world_to_light_local_matrix
```

**Edge Cases and Pitfalls:**
- **Stereo Rendering:** If the main camera is stereo, the projection camera must explicitly disable stereo (`stereoTargetEye = None`) to avoid confusion in `SetTargetBuffers` mode [S1, unofficial].
- **Precision:** The use of `R8` format for the geometry volume limits occupancy data to 8-bit integers. Ensure the fragment shader writes appropriate values.
- **Boundary Conditions:** The `Clamp` wrap mode on `_tex3d_gv` ensures out-of-bounds accesses during tracing do not wrap around, which is critical for cascade boundaries.

## Subtopics

### Cascaded Voxel Cone Tracing (CVCT) Tracing
A tracing technique that samples the Light Volume (LightTex3d) to compute indirect lighting for each voxel in the scene [S1, unofficial]. It operates on the same 3D texture layout as the Geometry Volume but reads from `LightTex3d` instead. For each cascade `i`, it dispatches `TracingKernel` with thread groups `((gridResolution+3)/4, (gridResolution+3)/4, (gridResolution+3)/4)` and sets `GridCascadeBase` to `(i * gridResolution, (gridCascades - 1) * gridResolution)` [S1, unofficial]. It is chosen when indirect lighting approximation via cone tracing is required, building on the voxelized geometry [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Voxelization is $O(N \cdot C)$, where $N$ is the number of voxels per cascade ($gridResolution^3$) and $C$ is the number of cascades. The repacking step is also $O(N \cdot C)$.
- **Memory Complexity:** $O(N \cdot C)$ for the 3D texture `_tex3d_gv` and $O(N)$ for the intermediate buffer `cb_gv`.
- **Performance:** Depends heavily on `gridResolution` and `gridCascades`. Higher resolutions increase memory bandwidth pressure during repacking.
- **Strengths:** Handles large scenes efficiently by varying resolution with distance. Orthographic projection ensures no perspective distortion in the light space.
- **Limitations:** Requires multiple render passes (one per cascade). The intermediate buffer allocation and deallocation can be costly if not pooled.
- **When to Use:** Real-time global illumination approximations where performance is critical and scene geometry is static or changes slowly.
- **When Not to Use:** Dynamic scenes with high-frequency geometry changes may suffer from aliasing or require frequent re-voxelization.

## Background
Originated in the "Cascaded Voxel Cone Tracing" implementation by arigo, shared as an unofficial GitHub repository [S1, unofficial]. It is part of the broader family of voxel-based global illumination techniques, specifically those using rasterization to build the initial geometry volume.

## Sources
[S1] arigo/CascatedVoxelConeTracing, https://github.com/arigo/CascatedVoxelConeTracing, fetched 2026-10-26 (unofficial)
