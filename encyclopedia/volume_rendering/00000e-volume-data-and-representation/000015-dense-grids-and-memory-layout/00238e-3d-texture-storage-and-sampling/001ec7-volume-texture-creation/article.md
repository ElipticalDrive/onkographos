# Volume Texture Creation

Also called: 3D Volume Texture Creation

## Overview
Volume Texture Creation allocates GPU memory and loads volumetric data (such as scalar fields, gradients, or transfer functions) into GPU-accessible texture objects for subsequent rendering [S1, S2, S4, S5, S7, S8, S9, unofficial]. It differs from [Volume Texture Setup](4055-volume-texture-setup.md) by focusing on the high-level creation logic and data injection rather than just API parameterization, and from [3D Texture Upload](4419-3d-texture-upload.md) by encompassing the entire lifecycle from allocation to binding [S1, S3, S5, unofficial]. It solves the problem of mapping host-side volumetric fields to device-local memory with appropriate access patterns [S1, S2, unofficial][S3, unofficial].

## How it works
The method varies significantly by API and data type. The following sections detail the specific implementations found in the sources.

### CUDA Texture Object Creation
This approach creates a CUDA texture object bound to a 3D CUDA array for efficient volume sampling with linear filtering and normalized coordinates [S2, unofficial].

**Inputs and Outputs:**
- Input: Host-side scalar volume data (`hostScalar`), volume dimensions (`dim.x`, `dim.y`, `dim.z`).
- Output: CUDA texture object (`cudaTextureObject_t`).

**Data Structures:**
- `cudaResourceDesc`: Resource descriptor.
- `cudaTextureDesc`: Texture descriptor.
- `cudaMemcpy3DParms`: 3D memory copy parameters.
- `cudaChannelFormatDesc`: Channel format description.

**Steps:**
1. Create channel format description for float data [S2, unofficial].
2. Allocate 3D array with extent defined by volume dimensions [S2, unofficial].
3. Configure `cudaMemcpy3DParms` for host-to-device copy:
   - Source pointer: `hostScalar`.
   - Source pitch: `dim.x * sizeof(float)`.
   - Destination array: `m_arrayField`.
   - Extent: Matches volume dimensions.
   - Kind: `cudaMemcpyHostToDevice`.
4. Execute the 3D memory copy [S2, unofficial].
5. Initialize `cudaResourceDesc` with `resType` set to `cudaResourceTypeArray` and `res.array.array` pointing to the target `cudaArray_t` [S2, unofficial].
6. Initialize `cudaTextureDesc`:
   - `normalizedCoords`: `true`.
   - `filterMode`: `cudaFilterModeLinear`.
   - `addressMode[0-2]`: `cudaAddressModeClamp`.
   - `readMode`: `cudaReadModeElementType`.
7. Create the texture object using `cudaCreateTextureObject` with the resource and texture descriptors [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Create channel format description for float data
channelDesc = cudaCreateChannelDesc(32, 0, 0, 0, cudaChannelFormatKindFloat)

# Step 2: Allocate 3D array with extent defined by volume dimensions
extent = cudaExtent(width=dim.x, height=dim.y, depth=dim.z)
cudaMalloc3DArray(&m_arrayField, &channelDesc, extent)

# Step 3 & 4: Configure and execute 3D memory copy
copyParams = cudaMemcpy3DParms()
copyParams.srcPtr = make_cudaPitchedPtr(hostScalar, dim.x * sizeof(float), dim.x, dim.y)
copyParams.dstArray = m_arrayField
copyParams.extent = extent
copyParams.kind = cudaMemcpyHostToDevice
cudaMemcpy3D(&copyParams)

# Step 5: Initialize resource descriptor
resourceDesc = cudaResourceDesc()
resourceDesc.resType = cudaResourceTypeArray
resourceDesc.res.array.array = m_arrayField

# Step 6: Initialize texture descriptor
textureDesc = cudaTextureDesc()
textureDesc.normalizedCoords = True
textureDesc.filterMode = cudaFilterModeLinear
textureDesc.addressMode[0] = cudaAddressModeClamp
textureDesc.addressMode[1] = cudaAddressModeClamp
textureDesc.addressMode[2] = cudaAddressModeClamp
textureDesc.readMode = cudaReadModeElementType

# Step 7: Create the texture object
cudaCreateTextureObject(&textureObject, &resourceDesc, &textureDesc, None)
```

### OpenGL Volume Texture Setup
This approach sets up 3D textures for attenuation values, gradients, and transfer functions using OpenGL [S4, unofficial].

**Inputs and Outputs:**
- Input: Raw data arrays (`volumeTexMap`, `gradientTexMap`, `transferTex`), dimensions (`dimX`, `dimY`, `dimZ`).
- Output: OpenGL texture IDs.

**Data Structures:**
- Attenuation: 3D texture of scalar values, 16-bit in range [0,1] [S4, unofficial].
- Gradients: 3D texture of 3-tuples, each component 16-bit float in range [-1,+1] [S4, unofficial].
- Transfer Function: 1D texture of 4-tuples, RGB stores $k_d$, Alpha stores $\tau'$, both 16-bit in [0,1] [S4, unofficial].

**Steps:**
1. Create attenuation texture:
   - Call `glTexImage3D(GL_TEXTURE_3D, 0, GL_R16, dimX, dimY, dimZ, 0, GL_RED, GL_UNSIGNED_SHORT, volumeTexMap)` [S4, unofficial].
2. Create gradient texture:
   - Call `glTexImage3D(GL_TEXTURE_3D, 0, GL_RGB16F, dimX, dimY, dimZ, 0, GL_RGB, GL_FLOAT, gradientTexMap)` [S4, unofficial].
3. Create transfer function texture:
   - Call `glTexImage1D(GL_TEXTURE_1D, 0, GL_RGBA16, 256, 0, GL_RGBA, GL_UNSIGNED_SHORT, transferTex)` [S4, unofficial].
   - Note: 256 is typically enough, but larger arrays provide more precision [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Attenuation Texture
glBindTexture(GL_TEXTURE_3D, volumeTexMap)
glTexImage3D(GL_TEXTURE_3D, 0, GL_R16, dimX, dimY, dimZ, 0, GL_RED, GL_UNSIGNED_SHORT, None)
glTexSubImage3D(GL_TEXTURE_3D, 0, 0, 0, 0, dimX, dimY, dimZ, GL_RED, GL_UNSIGNED_SHORT, volumeTexMap)

# Gradient Texture
glBindTexture(GL_TEXTURE_3D, gradientTexMap)
glTexImage3D(GL_TEXTURE_3D, 0, GL_RGB16F, dimX, dimY, dimZ, 0, GL_RGB, GL_FLOAT, None)
glTexSubImage3D(GL_TEXTURE_3D, 0, 0, 0, 0, dimX, dimY, dimZ, GL_RGB, GL_FLOAT, gradientTexMap)

# Transfer Function Texture
glBindTexture(GL_TEXTURE_1D, transferTex)
glTexImage1D(GL_TEXTURE_1D, 0, GL_RGBA16, 256, 0, GL_RGBA, GL_UNSIGNED_SHORT, None)
glTexSubImage1D(GL_TEXTURE_1D, 0, 0, 256, GL_RGBA, GL_UNSIGNED_SHORT, transferTex)
```

### WebGPU Volume Texture Upload
This approach creates a GPU texture and writes raw data to it using WebGPU [S3, unofficial].

**Inputs and Outputs:**
- Input: Raw data buffer, volume dimensions (`width`, `height`, `depthOrArrayLayers`).
- Output: WebGPU texture.

**Data Structures:**
- Format: `r16float` [S3, unofficial].
- Usage: `GPUTextureUsage.TEXTURE_BINDING | GPUTextureUsage.COPY_DST` [S3, unofficial].

**Steps:**
1. Create a GPU texture with size `{width, height, depthOrArrayLayers: slices}`, dimension `'2d'`, format `'r16float'`, and usage `GPUTextureUsage.TEXTURE_BINDING | GPUTextureUsage.COPY_DST` [S3, unofficial].
2. Write raw data to the volume texture using `device.queue.writeTexture`:
   - Texture: The created texture.
   - Origin: `{x:0, y:0, z:i}` for each slice `i`.
   - Source buffer: `rawData.buffer`.
   - BytesPerRow: `width * 2`.
   - RowsPerImage: `height`.
   - Extent: `{width, height, depthOrArrayLayers: 1}` [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Create texture
textureDescriptor = {
    size: {width: width, height: height, depthOrArrayLayers: slices},
    dimension: '2d',
    format: 'r16float',
    usage: GPUTextureUsage.TEXTURE_BINDING | GPUTextureUsage.COPY_DST
}
texture = device.createTexture(textureDescriptor)

# Step 2: Write data slice by slice
for i in range(slices):
    device.queue.writeTexture(
        texture=texture,
        data=rawData.buffer,
        textureOffset={x: 0, y: 0, z: i},
        bytesPerRow=width * 2,
        rowsPerImage=height,
        size={width: width, height: height, depthOrArrayLayers: 1}
    )
```

### VTK Volume Texture Creation
This approach allocates GPU memory and loads the color transfer function table for a `vtkVolume` [S1, unofficial].

**Inputs and Outputs:**
- Input: `vtkVolume` pointer (`vol`), `numberOfScalarComponents` (integer).
- Output: Loaded transfer function.

**Parameters:**
- `numberOfScalarComponents`: Must be exactly 1 or 4 [S1, unofficial].

**Pitfalls:**
- Fails or is invalid if `numberOfScalarComponents` is not 1 or 4 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Check scalar component count
if numberOfScalarComponents not in [1, 4]:
    raise ValueError("numberOfScalarComponents must be 1 or 4")

# Allocate GPU memory and load transfer function
if vol.transfer_function_changed:
    allocate_gpu_memory(vol)
    load_transfer_function_table(vol)
```

### Texture Array Loading (WebGL)
This approach loads multiple 2D images into a 3D texture array [S5, unofficial].

**Inputs and Outputs:**
- Input: List of URLs (`urls`), image dimensions (`width`, `height`).
- Output: WebGL texture object.

**Steps:**
1. Create a WebGL texture object. Bind it as `gl.TEXTURE_2D_ARRAY`. Initialize with a 1x1x1 RGBA texture (magenta) to allocate storage [S5, unofficial].
2. For each URL in `urls`:
   - Load an image.
   - Create a canvas of size `width` x `height`.
   - Draw the image onto the canvas.
   - Extract pixel data using `ctx.getImageData`.
   - Flatten the data into a `Uint8Array` and append to `pixels` array.
3. Upload the complete `pixels` array to the texture using `texImage3D` with dimensions `width`, `height`, and `depth` equal to `urls.length`.
4. Generate mipmaps.
5. Set mag filter to `NEAREST` and min filter to `NEAREST_MIPMAP_LINEAR` [S5, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Create and bind texture
texture = gl.createTexture()
gl.bindTexture(gl.TEXTURE_2D_ARRAY, texture)
gl.texImage3D(gl.TEXTURE_2D_ARRAY, 0, gl.RGBA, 1, 1, 1, 0, gl.RGBA, gl.UNSIGNED_BYTE, [255, 0, 255, 255])

# Step 2: Load images and collect pixels
pixels = []
for url in urls:
    img = load_image(url)
    canvas = create_canvas(width, height)
    ctx = canvas.get_context('2d')
    ctx.draw_image(img, 0, 0)
    data = ctx.get_image_data().data
    pixels.extend(data)

# Step 3: Upload data
gl.texImage3D(gl.TEXTURE_2D_ARRAY, 0, gl.RGBA, width, height, len(urls), 0, gl.RGBA, gl.UNSIGNED_BYTE, pixels)

# Step 4 & 5: Set filters
gl.generate_mipmap(gl.TEXTURE_2D_ARRAY)
gl.tex_parameteri(gl.TEXTURE_2D_ARRAY, gl.TEXTURE_MAG_FILTER, gl.NEAREST)
gl.tex_parameteri(gl.TEXTURE_2D_ARRAY, gl.TEXTURE_MIN_FILTER, gl.NEAREST_MIPMAP_LINEAR)
```

### VolPack Volume Context Configuration
This approach configures a VolPack context to define volume dimensions and voxel structure [S6].

**Inputs and Outputs:**
- Input: Context pointer (`vpc`), dimensions (`xlen`, `ylen`, `zlen`), voxel parameters.
- Output: Configured context.

**Steps:**
1. Create a new context using `vpCreateContext()`. Returns a pointer to `vpContext` with default values [S6].
2. Set volume dimensions using `vpSetVolumeSize(vpc, xlen, ylen, zlen)`. Returns `vpResult` (`VP_OK` on success) [S6].
3. Set voxel size and field counts using `vpSetVoxelSize(vpc, bytes_per_voxel, num_voxel_fields, num_shade_fields, num_classify_fields)` [S6].
4. Declare voxel field properties using `vpSetVoxelField(vpc, field_num, field_size, field_offset, field_max)` and `vpFieldOffset(void *voxel_ptr, field_name)` [S6].
5. Destroy context using `vpDestroyContext(vpc)` when done [S6].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Create context
vpc = vpCreateContext()

# Step 2: Set volume dimensions
result = vpSetVolumeSize(vpc, xlen, ylen, zlen)
if result != VP_OK:
    raise Error("Failed to set volume size")

# Step 3: Set voxel size
result = vpSetVoxelSize(vpc, bytes_per_voxel, num_voxel_fields, num_shade_fields, num_classify_fields)
if result != VP_OK:
    raise Error("Failed to set voxel size")

# Step 4: Declare fields
for field_num in range(num_voxel_fields):
    field_offset = vpFieldOffset(dummy_voxel, field_name)
    result = vpSetVoxelField(vpc, field_num, field_size, field_offset, field_max)
    if result != VP_OK:
        raise Error("Failed to set voxel field")

# Step 5: Destroy context
vpDestroyContext(vpc)
```

### Splat Texture Management
This approach creates and manages textures for Gaussian splats [S7, unofficial].

**Inputs and Outputs:**
- Input: `Uint32Array` data, `maxSplats`.
- Output: Texture object.

**Steps:**
1. Create a texture from a `Uint32Array` by instantiating `THREE.DataArrayTexture` with `data`, `width`, `height`, `depth`.
2. Set format to `THREE.RGBAIntegerFormat`, type to `THREE.UnsignedIntType`, magFilter and minFilter to `THREE.NearestFilter`, and generateMipmaps to `false` [S7, unofficial].
3. When ensuring an SH texture:
   - Calculate padded size as `maxSplats * 4`.
   - If input data length is less than padded size, allocate a new `Uint32Array` of that size, copy data into it, and replace the original reference [S7, unofficial].

**Pitfalls:**
- If `data.length` is zero or undefined, `getTextureSize` receives `Math.max(1, 0) = 1` to prevent degenerate texture dimensions [S7, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1 & 2: Create texture
texture = THREE.DataArrayTexture(data, width, height, depth)
texture.format = THREE.RGBAIntegerFormat
texture.type = THREE.UnsignedIntType
texture.magFilter = THREE.NearestFilter
texture.minFilter = THREE.NearestFilter
texture.generateMipmaps = False

# Step 3: Ensure SH texture padding
padded_size = maxSplats * 4
if len(data) < padded_size:
    new_data = Uint32Array(padded_size)
    new_data[0:len(data)] = data
    data = new_data
```

### Volume Texture Atlas Export
This approach exports a 3D volume to a 2D RGBA atlas for Unreal Engine 5 [S8, unofficial].

**Inputs and Outputs:**
- Input: Volume density data (`result.density`), dimensions (`width`, `height`, `depth`).
- Output: 2D RGBA atlas array.

**Parameters:**
- `tile_columns`: Number of columns in the atlas, typically equal to the volume size (`width`) [S8, unofficial].
- `tile_rows`: Number of rows in the atlas, typically 1 for single-row layout [S8, unofficial].

**Steps:**
1. Create an RGBA uint8 atlas array of shape `(rows * height, cols * width, 4)` initialized to zero.
2. Set Alpha channel to 255: `atlas[:, :, 3] = 255` [S8, unofficial].
3. Iterate `z_index` from 0 to `depth-1`:
   - Calculate row and column: `row, column = divmod(z_index, cols)`.
   - Calculate y0, x0 offsets: `y0, x0 = row * height, column * width`.
   - Copy `result.density[z_index]` to `atlas[y0:y0+height, x0:x0+width, 0]` [S8, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1 & 2: Create atlas
atlas = np.zeros((tile_rows * height, tile_columns * width, 4), dtype=np.uint8)
atlas[:, :, 3] = 255

# Step 3: Copy slices
for z_index in range(depth):
    row, column = divmod(z_index, tile_columns)
    y0, x0 = row * height, column * width
    atlas[y0:y0+height, x0:x0+width, 0] = result.density[z_index]
```

### Texture Creation from Palette (Unreal Engine)
This approach creates a texture resource in Unreal Engine [S9, unofficial].

**Inputs and Outputs:**
- Input: Palette data (`Palette.GetData()`).
- Output: `OutTexture`.

**Steps:**
1. Set `OutTexture->LODGroup` to `TEXTUREGROUP_World`, `CompressionSettings` to `TC_Default`, `MipGenSettings` to `TMGS_NoMipmaps`, `DeferCompression` to `true` [S9, unofficial].
2. Init `OutTexture->Source` with width 256, height 1, depth 1, layers 1, format `TSF_BGRA8`, data from `Palette.GetData()` [S9, unofficial].
3. Call `OutTexture->UpdateResource()` and `OutTexture->PostEditChange()` [S9, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Set properties
OutTexture.LODGroup = TEXTUREGROUP_World
OutTexture.CompressionSettings = TC_Default
OutTexture.MipGenSettings = TMGS_NoMipmaps
OutTexture.DeferCompression = True

# Step 2: Init source
OutTexture.Source.Init(256, 1, 1, 1, TSF_BGRA8, Palette.GetData())

# Step 3: Update
OutTexture.UpdateResource()
OutTexture.PostEditChange()
```

## Subtopics

### [Trilinear Volume Sampling for Texture Generation](2726-trilinear-volume-sampling-for-texture-generation.md)
Trilinear Volume Sampling retrieves scalar or vector attributes from a 3D voxel grid at continuous surface positions by interpolating values from the surrounding 2x2x2 neighborhood of active grid points. It enables spatially coherent attribute coding and direct surface queries without UV mapping.

### [Texture Generation](2769-texture-generation.md)
Texture Generation creates an empty 2D RGBA texture in OpenGL for storing rendered image data. It differs from [Trilinear Volume Sampling for Texture Generation](2726-trilinear-volume-sampling-for-texture-generation.md) by focusing on the allocation and configuration of the destination buffer rather than the interpolation logic, and from [Volume Texture Setup](4055-volume-texture-setup.md) by specifically targeting 2D image output rather than 3D volume input.

### [Volume Texture Setup](4055-volume-texture-setup.md)
Volume Texture Setup is the method for configuring GPU-side 3D texture objects to store volumetric data for rendering. It solves the problem of preparing the graphics API state (format, dimensions, filtering, wrapping) required for subsequent data injection or sampling. It is distinct from [Volume Texture Creation](1857-volume-texture-creation.md) in that it focuses on the specific parameterization and binding of the texture object rather than the high-level creation logic, and from [3D Texture Upload](4419-3d-texture-upload.md) which handles the actual data transfer.

### [Texture Slabs](7262-texture-slabs.md)
Texture Slabs is a volume rendering technique that loads 3D density data into the graphics card's texture map memory, treating the volume as a series of slabs between slices. It differs from standard ray casting by enabling the pre-integration of the volume rendering integral, allowing for efficient accumulation of color and opacity along rays.

### [GPU Memory and Buffer Construction](8345-gpu-memory-and-buffer-construction.md)
GPU Memory and Buffer Construction defines the mechanisms for organizing volumetric data, transfer functions, and rendering state into GPU-accessible memory structures (textures, buffers, and framebuffers) to enable efficient raycasting and rendering. It solves the problem of mapping logical host-side volumetric fields and topological meshes to device-local GPU memory with appropriate access patterns and formats. It differs from [Volume Texture Creation](1857-volume-texture-creation.md) by focusing on the final buffer allocation, memory binding, and layout construction rather than the generation of the texture content itself.

### GPU Volume Rendering Data Setup
Storage of attenuation values $\tau[x,y,z]$ in a 3D texture of scalar values, typically 16-bit in range [0,1], representing attenuation per distance [S4, unofficial]. Storage of pre-computed gradients $\nabla\tau[x,y,z]$ in a 3D texture of 3-tuples, where each component is a 16-bit float in range [-1,+1] [S4, unofficial]. Storage of transfer function $f: \tau \rightarrow k_d, \tau'$ in a 1D texture of 4-tuples. RGB components store $k_d$, Alpha component stores $\tau'$. Both are 16-bit values in [0,1] [S4, unofficial]. Transfer function texture size is typically 256, providing enough precision. Larger arrays provide more precision [S4, unofficial].

### Texture Array Loading
Create a WebGL texture object. Bind it as `gl.TEXTURE_2D_ARRAY`. Initialize with a 1x1x1 RGBA texture (magenta) to allocate storage [S5, unofficial]. For each URL in `urls`, load an image. Create a canvas of size `width` x `height`. Draw the image onto the canvas. Extract pixel data using `ctx.getImageData`. Flatten the data into a `Uint8Array` and append to `pixels` array [S5, unofficial]. Upload the complete `pixels` array to the texture using `texImage3D` with dimensions `width`, `height`, and `depth` equal to `urls.length`. Generate mipmaps. Set mag filter to `NEAREST` and min filter to `NEAREST_MIPMAP_LINEAR` [S5, unofficial]. `width` is the width of the individual texture layers, `height` is the height of the individual texture layers [S5, unofficial].

### Volume Context and Configuration
Create a new context using `vpCreateContext()`. Returns a pointer to `vpContext` with default values [S6]. Destroy a context using `vpDestroyContext(vpContext *vpc)` [S6]. Set volume dimensions using `vpSetVolumeSize(vpContext *vpc, int xlen, int ylen, int zlen)`. Returns `vpResult` (`VP_OK` on success) [S6]. Set voxel size and field counts using `vpSetVoxelSize(vpContext *vpc, int bytes_per_voxel, int num_voxel_fields, int num_shade_fields, int num_classify_fields)` [S6]. Declare voxel field properties using `vpSetVoxelField(vpContext *vpc, int field_num, int field_size, int field_offset, int field_max)` and `vpFieldOffset(void *voxel_ptr, field_name)` [S6].

### Splat Texture Management
Create a texture from a `Uint32Array` by instantiating `THREE.DataArrayTexture` with `data`, `width`, `height`, `depth`, then setting format to `THREE.RGBAIntegerFormat`, type to `THREE.UnsignedIntType`, magFilter and minFilter to `THREE.NearestFilter`, and generateMipmaps to `false` [S7, unofficial]. When ensuring an SH texture, calculate padded size as `maxSplats * 4`. If the input data length is less than padded size, allocate a new `Uint32Array` of that size, copy data into it, and replace the original reference [S7, unofficial]. `maxSplats` is the maximum number of splats supported by the texture buffer, used to compute padding dimensions and buffer capacity [S7, unofficial]. If `data.length` is zero or undefined, `getTextureSize` receives `Math.max(1, 0) = 1` to prevent degenerate texture dimensions [S7, unofficial].

### Volume Texture Atlas Export
Create an RGBA uint8 atlas array of shape `(rows * height, cols * width, 4)` initialized to zero. Set Alpha channel to 255: `atlas[:, :, 3] = 255` [S8, unofficial]. Iterate `z_index` from 0 to `depth-1`. Calculate row and column: `row, column = divmod(z_index, cols)`. Calculate y0, x0 offsets: `y0, x0 = row * height, column * width`. Copy `result.density[z_index]` to `atlas[y0:y0+height, x0:x0+width, 0]` [S8, unofficial]. `tile_columns` is the number of columns in the atlas, typically equal to the volume size (`width`) [S8, unofficial]. `tile_rows` is the number of rows in the atlas, typically 1 for single-row layout [S8, unofficial]. Designed for Unreal Engine 5, which requires creating a Volume Texture from a Texture2D atlas. The atlas is imported as a Texture2D (Masks/Linear), then 'Create Volume Texture' is used to generate the 3D resource [S8, unofficial].

### Texture Creation from Palette
Set `OutTexture->LODGroup` to `TEXTUREGROUP_World`, `CompressionSettings` to `TC_Default`, `MipGenSettings` to `TMGS_NoMipmaps`, `DeferCompression` to `true` [S9, unofficial]. Init `OutTexture->Source` with width 256, height 1, depth 1, layers 1, format `TSF_BGRA8`, data from `Palette.GetData()` [S9, unofficial]. Call `OutTexture->UpdateResource()` and `OutTexture->PostEditChange()` [S9, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S3] dongeronimo/webgpu_volume_renderer_demo, https://github.com/dongeronimo/webgpu_volume_renderer_demo, fetched 2026-10-29 (unofficial)
[S4] CISC/CMPE 454, GPU Volume Rendering, https://watkins.cs.queensu.ca/~jstewart/454/notes/44/44-gpu-volume-rendering.html, fetched 2026-10-30 (unofficial)
[S5] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S6] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S7] WilliamLiu-1997/Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
[S8] mingtu149411/Noise-Cloud, https://github.com/mingtu149411/Noise-Cloud, fetched 2026-10-27 (unofficial)
[S9] mik14a/VOX4U, https://github.com/mik14a/VOX4U, fetched 2026-10-26 (unofficial)
