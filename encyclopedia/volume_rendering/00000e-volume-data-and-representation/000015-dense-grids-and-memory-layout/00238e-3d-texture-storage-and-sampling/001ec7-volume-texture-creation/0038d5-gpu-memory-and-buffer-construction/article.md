# GPU Memory and Buffer Construction
## Overview
GPU Memory and Buffer Construction defines the mechanisms for allocating, initializing, and managing GPU-accessible memory resources (textures, buffers, framebuffers) required to store and render volumetric data [G]. It solves the problem of transferring host-side volumetric representations into device-local memory structures that shaders can efficiently sample [G]. It differs from [Volume Texture Creation](1857-volume-texture-creation.md) by focusing on the low-level resource allocation and memory management (staging buffers, texture objects, framebuffers) rather than the high-level API calls for texture generation, and from [Trilinear Volume Sampling for Texture Generation](2726-trilinear-volume-sampling-for-texture-generation.md) by handling the data layout and binding rather than the interpolation logic [G].

## Shared principles
The family shares the principle of mapping host-side data structures to GPU memory layouts that optimize for specific access patterns (e.g., coalesced memory access, texture filtering) [G]. Key shared concepts include:
- **Memory Hierarchy**: Data is often transferred via staging buffers (Host → Staging → Device) to handle cases where direct host-to-device mapping is unsupported or when data preparation is required [15388-volume-upload-via-staging-buffer.md].
- **Resource Binding**: Resources (textures, buffers) must be bound to specific texture units or shader stages with correct usage flags (e.g., `eSampled`, `eTransferDst`) to ensure correct access permissions and performance [15388-volume-upload-via-staging-buffer.md].
- **Immutability and Safety**: In some contexts, snapshots of volume data are made immutable to ensure thread-safe sharing across multiple kernels or frames [2246-device-volume-snapshot.md].

## Subtopics

### [Device Volume Snapshot](2246-device-volume-snapshot.md)
Creates a compact, immutable snapshot of volume metadata and GPU texture handles, optimized for passing to CUDA kernels. It differs from mutable buffer constructions by enforcing immutability once built, ensuring thread-safe sharing of internal resources across multiple snapshots.

### [GPU Data Layout](2469-gpu-data-layout.md)
Defines the specific arrangement of OpenGL resources (textures, framebuffers, and auxiliary buffers) required to execute multi-volume raycasting on the GPU. It solves the problem of managing multiple heterogeneous data sources (primary volume, color maps, depth buffers, and noise textures) within a single rendering pass. It differs from [Device Volume Snapshot](2246-device-volume-snapshot.md) by focusing on the active resource binding for raycasting rather than post-processing snapshots, and from [ClearView Unified Buffer Construction](2475-clearview-unified-buffer-construction.md) by using distinct texture objects for color/opacity tables rather than a unified buffer structure.

### [ClearView Unified Buffer Construction](2475-clearview-unified-buffer-construction.md)
A preprocessing technique that converts a hexahedral mesh (positions, connectivity, and scalar values) into four GPU-friendly texture buffers for the ClearView volume rendering pipeline. It differs from standard volume texture creation by explicitly encoding topological connectivity (faces and edges) alongside scalar data to support a specific fragment-based rendering pass, rather than relying on implicit grid traversal or sparse voxel structures.

### [Volume Upload via Staging Buffer](15388-volume-upload-via-staging-buffer.md)
Transfers dense 3D volume data from host memory to device-local GPU memory using an intermediate staging buffer. It solves the problem of moving large datasets to the GPU when direct host-to-device mapping is not supported or when the data must be prepared on the CPU before transfer. It differs from [Device Volume Snapshot](2246-device-volume-snapshot.md) by initiating the transfer from the host side rather than capturing existing device state, and from [GPU Data Layout](2469-gpu-data-layout.md) by explicitly managing the two-stage copy process (Host → Staging → Device) rather than defining the final memory layout alone.

### [Texture Buffer Object for Block Types](15685-texture-buffer-object-for-block-types.md)
Texture Buffer Object (TBO) is a technique to reinterpret a buffer object as a texture using `glTexBuffer`, allowing direct memory access via texture sampling functions. It is primarily used to store block type colors or voxel data in a TBO and bind it as a texture to allow shaders to look up values based on indices stored in vertex or voxel data. This method avoids the padding waste (e.g., 3 bytes per `uint8_t` element) or manual packing required when using Shader Storage Buffer Objects (SSBO) or Uniform Buffer Objects (UBO) directly with small integer types.

### GPU Texture Upload and Composition
- **Definition**: Draws the voxel map to a GPU texture, optionally compositing with rasterized scene objects [S1, unofficial].
- **Step**: Dispatch compute work groups based on the resolution of `outputTexture` to render the voxel map into it [S1, unofficial].
- **Parameter**: `outputTexture`: `GLuint` handle to the target texture. Must have internal format `GL_RGBA32F`. Its resolution determines the number of compute work groups dispatched [S1, unofficial]. `rasterColorTexture`: `int` handle to the color buffer for rasterized objects. Use `-1` to disable composition [S1, unofficial]. `rasterDepthTexture`: `int` handle to the depth buffer for rasterized objects. Use `-1` to disable composition [S1, unofficial].
- **Pitfall**: `outputTexture` must be `GL_RGBA32F`; using a different format will cause undefined behavior or rendering failure [S1, unofficial].

### GPU Data Structure Memory Management
- **Parameter**: The new buffer allocates double the size of the existing data structure to handle the worst case where each segment is split, as the number of splits is unknown beforehand [S2]. The theoretical minimum storage is four floats per segment and two integer values per pixel, but the implementation uses double the size for safety [S2].
- **Layout**: Data structures for all pixels are stored in one memory pool in global GPU memory. Two arrays maintain the size and offset of the respective data structure in the pool for per-pixel access [S2].

## Background
The methods described in this family are derived from various implementations of volume rendering pipelines, including OpenGL-based raycasting [S1, unofficial] and specialized mesh-based rendering techniques. The use of staging buffers for volume upload is a common pattern in Vulkan and OpenGL to handle large data transfers efficiently [15388-volume-upload-via-staging-buffer.md]. The ClearView technique represents a specific approach to rendering hexahedral meshes by encoding topology into textures [2475-clearview-unified-buffer-construction.md].

## Sources
[S1] frozein/DoonEngine, https://github.com/frozein/DoonEngine, fetched 2026-10-26 (unofficial)
[S2] View‐Dependent Visibility Optimization for Monte Carlo Volume Visualization, https://doi.org/10.1111/cgf.70064, fetched 2026-11-05
