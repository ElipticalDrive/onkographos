# GPU Memory and Buffer Construction


## Overview
GPU Memory and Buffer Construction organizes volumetric data, transfer functions, and rendering state into GPU-accessible memory structures (textures, framebuffers, and unified buffers) to enable efficient raycasting and volume visualization [G]. It solves the problem of translating host-side volumetric representations (dense grids, hexahedral meshes) into GPU-friendly formats that minimize memory bandwidth bottlenecks and support specific rendering techniques like MIP, speckle reduction, and hybrid rasterization-compositing [G]. It sets itself apart from [Volume Texture Creation](1857-volume-texture-creation.md) by focusing on the final memory layout, buffer construction, and resource management (framebuffers, unified buffers) rather than the interpolation or sampling logic itself [G].

## Shared principles
The family maps logical volumetric data structures to specific GPU memory resources with strict constraints on data types, dimensions, and formats to ensure compatibility with shader pipelines [G].

1. **Resource Typing**: Data is mapped to specific OpenGL/CUDA resource types [G].
   * 3D Textures are used for volumetric scalar data [G].
   * 1D Textures are used for transfer functions (color and opacity) [G].
   * 2D Textures are used for depth buffers, color buffers, noise textures, and specialized topology data (ClearView) [G].
   * Framebuffer Objects (FBOs) are used for offscreen rendering and ping-pong accumulation [G].
   * Unified Buffers/Structs are used for passing metadata and handles to kernels (CUDA) [S1, unofficial].

2. **Validation and Constraints**: Structures must be initialized and validated before being passed to shaders or render passes [G]. Texture formats must match shader expectations (e.g., `GL_RGBA32F` for compositing) [S2, unofficial].

3. **Initialization**: Host-side structures are populated with validated data types and bounds before kernel execution [S1, unofficial].

## Subtopics

### [GPU Data Layout](2469-gpu-data-layout.md)
GPU Data Layout defines the host-side memory structures and OpenGL resource IDs required to manage volumetric rendering state on the GPU. It organizes heterogeneous resources—3D scalar textures, 1D transfer function tables, and offscreen framebuffers—into a unified configuration object for raycasting mappers. It is chosen when using a VTK-based raycasting mapper that requires explicit management of texture IDs, framebuffers, and bounding boxes.

### [ClearView Unified Buffer Construction](2475-clearview-unified-buffer-construction.md)
ClearView Unified Buffer Construction is a preprocessing technique that converts a hexahedral mesh (positions, connectivity, scalars) into four GPU-friendly texture buffers (vertices, edges, faces, cellScalar) for the ClearView volume rendering pipeline. It sets itself apart from siblings like [GPU Data Layout](2469-gpu-data-layout.md) by explicitly organizing topology into a "unified" set of textures optimized for the specific gather/resolve passes of the ClearView renderer, rather than generic storage.

### Device Volume Snapshot
#### definition
Creates a compact, immutable snapshot of volume metadata and texture handles for passing to CUDA kernels [S1, unofficial].
#### step
Initialize `DeviceVolume` struct with values from `Volume::Description` and texture handles [S1, unofficial].
#### layout
`DeviceVolume` struct contains: `field_tex` (`cudaTextureObject_t`), `grad_tex` (`cudaTextureObject_t`, can be 0), `dim` (`int3`), `voxel_size` (`float3`), `origin` (`float3`), `value_range` (`float2`), `density_scale` (`float`) [S1, unofficial].

### GPU Texture Upload and Composition
#### definition
Draws the voxel map to a GPU texture, optionally compositing with rasterized scene objects [S2, unofficial].
#### step
Dispatch compute work groups based on the resolution of `outputTexture` to render the voxel map into it [S2, unofficial].
#### parameter
`outputTexture`: `GLuint` handle to the target texture. Must have internal format `GL_RGBA32F`. Its resolution determines the number of compute work groups dispatched [S2, unofficial].
`rasterColorTexture`: `int` handle to the color buffer for rasterized objects. Use `-1` to disable composition [S2, unofficial].
`rasterDepthTexture`: `int` handle to the depth buffer for rasterized objects. Use `-1` to disable composition [S2, unofficial].
#### pitfall
`outputTexture` must be `GL_RGBA32F`; using a different format will cause undefined behavior or rendering failure [S2, unofficial].

## Background
The concepts in this family originate from CUDA-accelerated volume rendering engines and hybrid rendering pipelines [G]. The "GPU Data Layout" and "Device Volume Snapshot" structures are derived from the `cuVRender` engine, which uses a `Scene` class to manage camera, volume, transfer function, and light state, committing snapshots to device memory for kernel execution [S1, unofficial]. The "GPU Texture Upload and Composition" technique is associated with the DoonEngine, which integrates voxel maps with rasterized scenes using compute shaders [S2, unofficial]. These methods are used in scientific visualization, medical imaging, and real-time volume rendering applications where performance and memory efficiency are critical [G].

## Sources
[S1] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S2] frozein/DoonEngine, https://github.com/frozein/DoonEngine, fetched 2026-10-26 (unofficial)
