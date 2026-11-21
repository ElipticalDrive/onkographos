# Cascaded Voxel Cone Tracing Voxelization
Also called: Geometry Volume (GV) Voxelization

## Overview
Cascaded Voxel Cone Tracing Voxelization is a technique that renders scene geometry into a 3D texture (Geometry Volume, GV) using an orthographic camera aligned with the light direction, organized into cascades to handle varying distances and resolutions [S1, unofficial]. It differs from sibling methods by explicitly managing spatial coherence through a cascade hierarchy where each level covers a volume twice as large as the previous one, enabling efficient indirect lighting computation via cone tracing [S1, unofficial].

## How it works
**Inputs:**
- Scene geometry (meshes).
- Directional light transform (position and rotation) [S1, unofficial].
- Parameters: `gridResolution` (integer, e.g., 64 or 128), `gridPixelSize` (world units per voxel), `gridCascades` (integer count) [S1, unofficial].
- Culling mask for the projection camera [S1, unofficial].

**Outputs:**
- `_tex3d_gv`: A 3D RenderTexture representing the Geometry Volume.
- `_world_to_light_local_matrix`: A transformation matrix mapping world coordinates to light-local space.

**Data Structures:**
- `CVCT_gv`: A `RWStructuredBuffer<int>` of size `gridResolution^3`. Used as an intermediate buffer during rasterization before packing into the 3D texture [S1, unofficial].
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
   Create or reuse an orthographic camera (`_projectCam`) with `aspect = 1`, `clearFlags = Nothing`, and `stereoTargetEye = None` [S1, unofficial]. Align its position and rotation with the directional light's transform [S1, unofficial]. Set its culling mask [S1, unofficial].

2. **Allocate Buffers:**
   Allocate `cb_gv` as a `ComputeBuffer` of type `RWStructuredBuffer<int>` with size `gridResolution * gridResolution * gridResolution` [S1, unofficial].
   Allocate `dummy_target` as a temporary `RenderTexture` with dimensions `gridResolution x gridResolution`, format `R8`, and no depth buffer [S1, unofficial].

3. **Configure Compute Shaders:**
   Identify kernels: `ClearKernel`, `RepackLvl0Kernel`, `RepackAndUpscaleKernel` [S1, unofficial].
   Set global shader integer `_CVCT_GridResolutionI` to `gridResolution` [S1, unofficial].
   Set compute shader integers: `GridResolution` and `GridHalfResolution` (`gridResolution / 2`) [S1, unofficial].
   Bind `cb_gv` to `CVCT_gv` in all kernels [S1, unofficial].
   Bind `_tex3d_gv` to `CVCT_tex3d` in repack kernels [S1, unofficial].

4. **Iterate Over Cascades (i from 0 to gridCascades-1):**
   
   a. **Calculate Cascade Bounds:**
      Compute `half_size`:
      $$ \text{half\_size} = 0.5 \times \text{gridResolution} \times \text{gridPixelSize} \times 2^i $$
      
   b. **Clear Intermediate Buffer:**
      Calculate thread group count for clearing:
      $$ \text{thread\_groups} = \left\lfloor \frac{\text{gridResolution}^3 \times 63}{64} \right\rfloor $$
      Dispatch `ClearKernel` with `(thread_groups, 1, 1)` [S1, unofficial]. This zeros out `cb_gv`.

   c. **Render Geometry:**
      Configure the projection camera:
      - `orthographicSize = half_size`
      - `nearClipPlane = -half_size`
      - `farClipPlane = half_size`
      - `targetTexture = dummy_target` (used to drive the render pipeline, but data is written to the buffer) [S1, unofficial].
      
      Set random write target:
      ```python
      set_random_write_target(1, cb_gv)
      ```
      
      Render the scene using `gvShader`. The fragment shader writes voxel occupancy data into `cb_gv` [S1, unofficial].

   d. **Repack into 3D Texture:**
      Calculate thread group count for repacking:
      $$ \text{thread\_groups} = \left\lceil \frac{\text{gridResolution} + 3}{4} \right\rceil $$
      
      If `i > 0`:
      Set compute shader integer array `GridCascadeBase` to `(i * gridResolution, (i - 1) * gridResolution)` [S1, unofficial].
      
      Dispatch the appropriate repack kernel:
      - If `i == 0`: Dispatch `RepackLvl0Kernel` with `(thread_groups, thread_groups, thread_groups)` [S1, unofficial].
      - If `i > 0`: Dispatch `RepackAndUpscaleKernel` with `(thread_groups, thread_groups, thread_groups)` [S1, unofficial].

5. **Compute World-to-Light Matrix:**
   After processing all cascades, release `dummy_target` and `cb_gv` [S1, unofficial].
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
    cam.aspect = 1.0
    cam.clear_flags = "Nothing"
    cam.stereo_target_eye = "None"
    
    # 2. Allocate Buffers
    cb_gv = allocate_compute_buffer(gridResolution ** 3, element_size=4)
    dummy_target = get_temporary_render_texture(gridResolution, gridResolution, format="R8")
    
    # 3. Configure Compute Shaders
    clear_kernel = gvCompute.find_kernel("ClearKernel")
    repack_lvl0_kernel = gvCompute.find_kernel("RepackLvl0Kernel")
    repack_upscale_kernel = gvCompute.find_kernel("RepackAndUpscaleKernel")
    
    gvCompute.set_int("GridResolution", gridResolution)
    gvCompute.set_int("GridHalfResolution", gridResolution // 2)
    
    gvCompute.set_buffer(clear_kernel, "CVCT_gv", cb_gv)
    gvCompute.set_buffer(repack_lvl0_kernel, "CVCT_gv", cb_gv)
    gvCompute.set_texture(repack_lvl0_kernel, "CVCT_tex3d", _tex3d_gv)
    gvCompute.set_buffer(repack_upscale_kernel, "CVCT_gv", cb_gv)
    gvCompute.set_texture(repack_upscale_kernel, "CVCT_tex3d", _tex3d_gv)
    
    set_global_int("_CVCT_GridResolutionI", gridResolution)
    
    # 4. Iterate Cascades
    for i in range(gridCascades):
        half_size = 0.5 * gridResolution * gridPixelSize * (2.0 ** i)
        
        # Clear
        thread_groups_clear = (gridResolution ** 3 * 63) // 64
        gvCompute.dispatch(clear_kernel, thread_groups_clear, 1, 1)
        
        # Render
        cam.orthographic_size = half_size
        cam.near_clip_plane = -half_size
        cam.far_clip_plane = half_size
        cam.target_texture = dummy_target
        set_random_write_target(1, cb_gv)
        render_scene_with_shader(cam, gvShader, "RenderType")
        
        # Repack
        thread_groups_repack = (gridResolution + 3) // 4
        if i > 0:
            gvCompute.set_ints("GridCascadeBase", i * gridResolution, (i - 1) * gridResolution)
        
        kernel = repack_lvl0_kernel if i == 0 else repack_upscale_kernel
        gvCompute.dispatch(kernel, thread_groups_repack, thread_groups_repack, thread_groups_repack)
        
    # 5. Cleanup and Matrix
    release_temporary_render_texture(dummy_target)
    cb_gv.release()
    
    sz = 0.5 * gridResolution * gridPixelSize
    scale_matrix = create_scale_matrix(1.0 / sz)
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
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Depends heavily on `gridResolution` and `gridCascades`. Higher resolutions increase memory bandwidth pressure during repacking [G].
- **Strengths:** Handles large scenes efficiently by varying resolution with distance. Orthographic projection ensures no perspective distortion in the light space [G].
- **Limitations:** Requires multiple render passes (one per cascade). The intermediate buffer allocation and deallocation can be costly if not pooled [G].
- **When to Use:** Real-time global illumination approximations where performance is critical and scene geometry is static or changes slowly [G].
- **When Not to Use:** Dynamic scenes with high-frequency geometry changes may suffer from aliasing or require frequent re-voxelization [G].

## Background
Originated in the "Cascaded Voxel Cone Tracing" implementation by arigo, shared as an unofficial GitHub repository [S1, unofficial]. It is part of the broader family of voxel-based global illumination techniques, specifically those using rasterization to build the initial geometry volume.

## Sources
[S1] arigo/CascatedVoxelConeTracing, https://github.com/arigo/CascatedVoxelConeTracing, fetched 2026-10-26 (unofficial)
