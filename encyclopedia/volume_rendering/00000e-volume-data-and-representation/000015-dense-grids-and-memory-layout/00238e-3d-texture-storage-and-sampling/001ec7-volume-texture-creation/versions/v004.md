# Volume Texture Creation

Also called: 3D Volume Texture Creation

## Overview
Volume Texture Creation allocates GPU memory and loads volumetric data, transfer functions, or auxiliary fields into GPU-accessible texture resources for subsequent rendering [S1, unofficial][S2, unofficial][S3, unofficial]. It solves the problem of translating abstract host-side volumetric representations (dense grids, scalar fields, label maps) into hardware-ready texture objects with correct memory layouts, filtering modes, and resource flags [S1, unofficial][S2, unofficial][S3, unofficial][S4, unofficial][S5, unofficial]. Unlike [Volume Texture Setup](4055-volume-texture-setup.md), which configures sampling states and format flags, this method handles the high-level allocation, data transfer, and object lifecycle [S1, unofficial][S2, unofficial][S3, unofficial]. It differs from [Sparse Volume Texture (SVT)](7011-sparse-volume-texture-svt.md) by operating on dense grids [S7154-memory-organization-and-layout.md].

## How it works
The method varies significantly by API and data type. The following sections detail the primary implementation patterns found in the sources: CUDA, WebGPU, OpenGL (Standard), OpenGL (GPU Volume Rendering Data Setup), VolPack, and WebGL (Texture Array).

### Inputs and Outputs
*   **Inputs**: Host-side memory buffers containing volume data (e.g., `hostScalar`), transfer function tables, or image URLs. Dimensions $(dimX, dimY, dimZ)$ or width/height/depth.
*   **Outputs**: GPU texture objects (e.g., `cudaTextureObject_t`, `GPUTexture`, `GLuint`), bound to memory arrays or buffers.

### Data Structures

#### CUDA Volume Data
*   **3D CUDA Array**: Allocated with extent $(dim.x, dim.y, dim.z)$ [S2, unofficial].
*   **Channel Format**: Float data, typically 32-bit or 16-bit depending on precision requirements [S2, unofficial].
*   **Resource Descriptor (`cudaResourceDesc`)**:
    *   `resType`: `cudaResourceTypeArray` [S2, unofficial].
    *   `res.array.array`: Pointer to `cudaArray_t` [S2, unofficial].
*   **Texture Descriptor (`cudaTextureDesc`)**:
    *   `normalizedCoords`: `true` [S2, unofficial].
    *   `filterMode`: `cudaFilterModeLinear` [S2, unofficial].
    *   `addressMode[0-2]`: `cudaAddressModeClamp` [S2, unofficial].
    *   `readMode`: `cudaReadModeElementType` [S2, unofficial].

#### WebGPU Volume Data
*   **Texture Descriptor**:
    *   `size`: `{width, height, depthOrArrayLayers: slices}` [S3, unofficial].
    *   `dimension`: `'2d'` [S3, unofficial].
    *   `format`: `'r16float'` [S3, unofficial].
    *   `usage`: `GPUTextureUsage.TEXTURE_BINDING | GPUTextureUsage.COPY_DST` [S3, unofficial].
*   **Write Parameters**:
    *   `origin`: `{x:0, y:0, z:i}` [S3, unofficial].
    *   `bytesPerRow`: `width * 2` [S3, unofficial].
    *   `rowsPerImage`: `height` [S3, unofficial].
    *   `extent`: `{width, height, depthOrArrayLayers: 1}` [S3, unofficial].

#### OpenGL Standard (GPU Volume Rendering Data Setup)
*   **Attenuation Texture**:
    *   Format: `GL_R16` [S4, unofficial].
    *   Type: `GL_UNSIGNED_SHORT` [S4, unofficial].
    *   Data: 1D array of `dimX * dimY * dimZ` values in $[0,1]$ [S4, unofficial].
*   **Gradient Texture**:
    *   Format: `GL_RGB16F` [S4, unofficial].
    *   Type: `GL_FLOAT` [S4, unofficial].
    *   Data: 1D array of `dimX * dimY * dimZ` 3-tuples, components in $[-1, +1]$ [S4, unofficial].
*   **Transfer Function Texture**:
    *   Format: `GL_RGBA16` [S4, unofficial].
    *   Type: `GL_UNSIGNED_SHORT` [S4, unofficial].
    *   Size: Typically 256 entries [S4, unofficial].
    *   Data: 1D array of 4-tuples; RGB stores $k_d$, Alpha stores $\tau'$ [S4, unofficial].

#### VolPack Context
*   **Voxel Structure**: Defined by `bytes_per_voxel`, `num_voxel_fields`, `num_shade_fields`, `num_classify_fields` [S7].
*   **Field Properties**: Declared via `vpSetVoxelField` with `field_num`, `field_size`, `field_offset`, `field_max` [S7].

### Numbered Steps

#### CUDA Implementation [S2, unofficial]
1.  Create channel format description for float data.
2.  Allocate 3D CUDA array with extent $(dim.x, dim.y, dim.z)$.
3.  Configure `cudaMemcpy3DParms`:
    *   `srcPtr`: Pointer to `hostScalar`.
    *   `srcPitch`: $dim.x \times sizeof(float)$.
    *   `dstArray`: Target `cudaArray_t`.
    *   `extent`: Volume dimensions.
    *   `kind`: `cudaMemcpyHostToDevice`.
4.  Execute the 3D memory copy.
5.  Initialize `cudaResourceDesc` with `resType = cudaResourceTypeArray` and `res.array.array` pointing to the array.
6.  Initialize `cudaTextureDesc` with `normalizedCoords = true`, `filterMode = cudaFilterModeLinear`, `addressMode[0-2] = cudaAddressModeClamp`, `readMode = cudaReadModeElementType`.
7.  Create texture object using `cudaCreateTextureObject` with the resource and texture descriptors.

#### WebGPU Implementation [S3, unofficial]
1.  Create a GPU texture with size `{width, height, depthOrArrayLayers: slices}`, dimension `'2d'`, format `'r16float'`, and usage `GPUTextureUsage.TEXTURE_BINDING | GPUTextureUsage.COPY_DST`.
2.  For each slice $i$ from $0$ to $depth-1$:
    *   Write raw data to the volume texture using `device.queue.writeTexture`.
    *   Specify `texture`, `origin {x:0, y:0, z:i}`, `source buffer` (`rawData.buffer`), `bytesPerRow` (`width * 2`), `rowsPerImage` (`height`), and `extent {width, height, depthOrArrayLayers: 1}`.

#### OpenGL Standard Implementation [S4, unofficial]
1.  **Attenuation**: Call `glTexImage3D(GL_TEXTURE_3D, 0, GL_R16, dimX, dimY, dimZ, 0, GL_RED, GL_UNSIGNED_SHORT, volumeTexMap)`.
2.  **Gradient**: Call `glTexImage3D(GL_TEXTURE_3D, 0, GL_RGB16F, dimX, dimY, dimZ, 0, GL_RGB, GL_FLOAT, gradientTexMap)`.
3.  **Transfer Function**: Call `glTexImage1D(GL_TEXTURE_1D, 0, GL_RGBA16, 256, 0, GL_RGBA, GL_UNSIGNED_SHORT, transferTex)`.

#### VolPack Implementation [S7]
1.  Create a new context using `vpCreateContext()`.
2.  Set volume dimensions using `vpSetVolumeSize(vpc, xlen, ylen, zlen)`.
3.  Set voxel size and field counts using `vpSetVoxelSize(vpc, bytes_per_voxel, num_voxel_fields, num_shade_fields, num_classify_fields)`.
4.  Declare voxel field properties using `vpSetVoxelField(vpc, field_num, field_size, field_offset, field_max)` and `vpFieldOffset(voxel_ptr, field_name)` for each field.

#### WebGL Texture Array Implementation [S6, unofficial]
1.  Create a WebGL texture object. Bind it as `gl.TEXTURE_2D_ARRAY`.
2.  Initialize with a $1 \times 1 \times 1$ RGBA texture (magenta) to allocate storage.
3.  For each URL in `urls`:
    *   Load an image.
    *   Create a canvas of size `width` $\times$ `height`.
    *   Draw the image onto the canvas.
    *   Extract pixel data using `ctx.getImageData`.
    *   Flatten the data into a `Uint8Array` and append to `pixels` array.
4.  Upload the complete `pixels` array to the texture using `texImage3D` with dimensions `width`, `height`, and `depth` equal to `urls.length`.
5.  Generate mipmaps.
6.  Set mag filter to `NEAREST` and min filter to `NEAREST_MIPMAP_LINEAR`.

#### Splat Texture Management [S8, unofficial]
1.  Create a texture from a `Uint32Array` by instantiating `THREE.DataArrayTexture` with `data`, `width`, `height`, `depth`.
2.  Set format to `THREE.RGBAIntegerFormat`, type to `THREE.UnsignedIntType`, magFilter and minFilter to `THREE.NearestFilter`, and `generateMipmaps` to `false`.
3.  When ensuring an SH texture:
    *   Calculate padded size as `maxSplats * 4`.
    *   If input data length is less than padded size, allocate a new `Uint32Array` of that size, copy data into it, and replace the original reference.

#### Volume Texture Atlas Export [S9, unofficial]
1.  Create an RGBA uint8 atlas array of shape `(rows * height, cols * width, 4)` initialized to zero.
2.  Set Alpha channel to 255: `atlas[:, :, 3] = 255`.
3.  Iterate `z_index` from $0$ to `depth-1`:
    *   Calculate `row, column = divmod(z_index, cols)`.
    *   Calculate `y0, x0` offsets: `y0 = row * height`, `x0 = column * width`.
    *   Copy `result.density[z_index]` to `atlas[y0:y0+height, x0:x0+width, 0]`.

### Parameters
*   **CUDA**: Volume dimensions $(dim.x, dim.y, dim.z)$ [S2, unofficial].
*   **WebGPU**: Width, height, depth/slices [S3, unofficial].
*   **OpenGL (Standard)**: `dimX, dimY, dimZ` for 3D textures; 256 for transfer function [S4, unofficial].
*   **VolPack**: `xlen, ylen, zlen` for volume size; `bytes_per_voxel`, `num_voxel_fields`, `num_shade_fields`, `num_classify_fields` for voxel structure [S7].
*   **WebGL Array**: `width`, `height`, `urls.length` (depth) [S6, unofficial].
*   **Splat**: `maxSplats` (maximum number of splats supported) [S8, unofficial].
*   **Atlas**: `tile_columns` (typically equal to volume width), `tile_rows` (typically 1) [S9, unofficial].

### Edge Cases and Pitfalls
*   **VTK Scalar Components**: Fails or is invalid if `numberOfScalarComponents` is not 1 or 4 [S1, unofficial].
*   **Splat Data Length**: If `data.length` is zero or undefined, `getTextureSize` receives `Math.max(1, 0) = 1` to prevent degenerate texture dimensions [S8, unofficial].
*   **Precision**: Larger transfer function arrays provide more precision, but 256 is usually sufficient [S4, unofficial].
*   **Range Conversion**: OpenGL converts CPU-side `GL_UNSIGNED_SHORT` values to $[0,1]$ in textures automatically, regardless of whether they are stored as floats or shorts on the CPU [S4, unofficial].

## Subtopics

### [Trilinear Volume Sampling for Texture Generation](2726-trilinear-volume-sampling-for-texture-generation.md)
Retrieves scalar or vector attributes from a 3D voxel grid at continuous surface positions by interpolating values from the surrounding 2x2x2 neighborhood of active grid points. It enables spatially coherent attribute coding and direct surface queries without UV mapping, serving as the fundamental data retrieval primitive for direct volume rendering, isosurface extraction, or maximum intensity projection. Unlike its parent method, which handles coordinate transformation and buffer construction, this method focuses strictly on the interpolation operation itself.

### [Volume Texture Setup](4055-volume-texture-setup.md)
Configures GPU-accessible 3D texture objects and their associated sampling states to serve as the data source for real-time volumetric rendering. It solves the problem of translating abstract volumetric data representations (density fields, label maps, noise functions) into hardware-ready texture resources with correct memory layouts, filtering modes, and resource flags. It is distinct from [Volume Texture Creation](1857-volume-texture-creation.md), which handles high-level allocation and object lifecycle, by focusing on the specific configuration of format, filtering, and platform-specific flags required for correct sampling behavior.

### [Texture Slabs](7262-texture-slabs.md)
A volume rendering technique that loads 3D density data into the graphics card's texture map memory, treating the volume as a series of slabs between slices. It differs from standard ray casting by enabling the pre-integration of the volume rendering integral, allowing for efficient accumulation of color and opacity along rays.

### [GPU Memory and Buffer Construction](8345-gpu-memory-and-buffer-construction.md)
Defines the mechanisms for organizing volumetric data, transfer functions, and rendering state into GPU-accessible memory structures (textures, framebuffers, and unified buffers) to enable efficient raycasting and volume visualization. It solves the problem of translating host-side volumetric representations (dense grids, hexahedral meshes) into GPU-friendly formats that minimize memory bandwidth bottlenecks and support specific rendering techniques like MIP, speckle reduction, and hybrid rasterization-compositing. It sets itself apart from [Volume Texture Creation](1857-volume-texture-creation.md) by focusing on the final memory layout, buffer construction, and resource management (framebuffers, unified buffers) rather than the interpolation or sampling logic itself.

### GPU Volume Rendering Data Setup
Stores attenuation values $\tau[x,y,z]$ in a 3D texture of scalar values (typically 16-bit in $[0,1]$), pre-computed gradients $\nabla\tau[x,y,z]$ in a 3D texture of 3-tuples (16-bit float in $[-1,+1]$), and transfer function $f: \tau \rightarrow k_d, \tau'$ in a 1D texture of 4-tuples (16-bit values in $[0,1]$) [S4, unofficial]. The transfer function texture size is typically 256 [S4, unofficial].

### Texture Generation
Creates an empty 2D RGBA texture in OpenGL for storing rendered image data [S5, unofficial]. It generates a texture ID, binds to `GL_TEXTURE_2D`, sets min/mag filters to `GL_LINEAR`, wrap modes to `GL_CLAMP_TO_EDGE`, and allocates memory with `glTexImage2D` using `GL_RGBA` format and `GL_UNSIGNED_BYTE` type [S5, unofficial].

### Texture Array Loading
Loads multiple 2D images into a 3D texture array (WebGL) [S6, unofficial]. It creates a `TEXTURE_2D_ARRAY`, initializes storage, loads images from URLs, flattens pixel data into a `Uint8Array`, and uploads it via `texImage3D` [S6, unofficial].

### Volume Context and Configuration
Manages VolPack rendering state [S7]. It creates/destroys contexts (`vpCreateContext`, `vpDestroyContext`), sets volume dimensions (`vpSetVolumeSize`), and declares voxel field properties (`vpSetVoxelSize`, `vpSetVoxelField`) [S7].

### Splat Texture Management
Manages textures for Gaussian Splatting [S8, unofficial]. It creates `DataArrayTexture` with specific formats and filters, and handles padding for spherical harmonic data based on `maxSplats` [S8, unofficial].

### Volume Texture Atlas Export
Exports volume slices into a 2D RGBA atlas for Unreal Engine 5 [S9, unofficial]. It creates an atlas array, sets alpha to 255, and copies density slices into tiled positions within the atlas [S9, unofficial].

## Operating characteristics
*   **Time Complexity**: Dominated by memory bandwidth during host-to-device transfer. CUDA `cudaMemcpy3D` and WebGPU `writeTexture` are optimized for 3D layouts. OpenGL `glTexImage3D` is typically a one-time upload.
*   **Memory Complexity**: Proportional to $dimX \times dimY \times dimZ \times bytes\_per\_voxel$. For VolPack, memory is also influenced by the number of fields per voxel [S7].
*   **Performance**: Linear filtering and normalized coordinates (CUDA) or standard OpenGL filtering provide smooth sampling but require texture cache hits [S2, unofficial][S4, unofficial]. Nearest filtering (WebGL Array) is faster but aliasing-prone [S6, unofficial].
*   **Strengths**: Hardware-accelerated sampling, support for mipmaps (WebGL), and efficient 3D memory layouts (CUDA/WebGPU).
*   **Limitations**: Fixed resolution (dense grids), memory consumption for large volumes, and potential precision loss with 16-bit formats if not carefully managed [S4, unofficial].
*   **When to use**: Real-time volume rendering, isosurface extraction, or when hardware sampling is required.
*   **When not to use**: Extremely large sparse volumes (use SVT), or when dynamic, per-frame voxel modification is needed without re-uploading (use compute buffers/SSBOs).

## Background
Volume texture creation is a fundamental technique in GPU-accelerated volume rendering, emerging with the advent of 3D texture support in OpenGL and CUDA. Early implementations used OpenGL's `glTexImage3D` for static volumes [S4, unofficial]. CUDA introduced `cudaArray` and texture objects for efficient, cached sampling in compute kernels [S2, unofficial]. WebGPU and WebGL 2.0+ extended these concepts to the web with `GPUTexture` and `TEXTURE_2D_ARRAY` [S3, unofficial][S6, unofficial]. VolPack provided an early high-level API for managing voxel fields and rendering contexts [S7]. The technique is widely used in medical imaging, scientific visualization, and real-time game engines (e.g., Unreal Engine's Volume Texture) [S9, unofficial].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S3] dongeronimo/webgpu_volume_renderer_demo, https://github.com/dongeronimo/webgpu_volume_renderer_demo, fetched 2026-10-29 (unofficial)
[S4] CISC/CMPE 454, GPU Volume Rendering, https://watkins.cs.queensu.ca/~jstewart/454/notes/44/44-gpu-volume-rendering.html, fetched 2026-10-30 (unofficial)
[S5] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S6] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S7] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S8] WilliamLiu-1997/Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
[S9] mingtu149411/Noise-Cloud, https://github.com/mingtu149411/Noise-Cloud, fetched 2026-10-27 (unofficial)
