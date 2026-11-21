# GPU Texture Loading

Also called: LoadScalarField, Raw Volumetric Data Loading (8/16-bit)

## Overview
GPU Texture Loading transfers volumetric scalar data from host memory (binary files or VTK data structures) to GPU 3D textures for subsequent raycasting or sampling [S1, S2, S3, unofficial]. It solves the problem of preparing raw voxel data for hardware-accelerated interpolation by handling format conversion, memory allocation, and texture binding [S2, S3, unofficial]. It differs from [Tiled 2D Texture Storage for 3D Fields](2740-tiled-2d-texture-storage-for-3d-fields.md) by using native 3D texture objects rather than packing 3D data into 2D tile arrays, and from [CUDA-accelerated Vector Field Reading with Magnitude](2581-cuda-accelerated-vector-field-reading-with-magnitude.md) by focusing on scalar field loading rather than precomputing vector magnitudes [S1, S2, S3, unofficial].

## How it works
**Inputs:**
1.  Source data: Either a binary file path and bit-depth flag (8/16-bit) [S2, unofficial], or a `vtkImageData` pointer with scalar extraction parameters [S1, unofficial].
2.  Extent: A 6-integer array `textureExtent[6]` defining the requested subextent in dataset coordinates `[minX, maxX, minY, maxY, minZ, maxZ]` [S1, unofficial].
3.  Configuration: `is16Bits` flag (boolean) [S2, unofficial], `type` flag (integer: 0 for transformed input, 1 for raw input) [S1, unofficial], and `CellFlag` (boolean indicating point vs. cell data) [S1, unofficial].

**Outputs:**
A bound 3D texture object (`dataTex` or OpenGL texture ID) containing the scalar field data, ready for linear sampling [S1, S2, S3, unofficial].

**Data Structures:**
*   **Host Buffer:** `std::vector<uint8_t>` or `std::vector<uint16_t>` sized to the number of voxels in the extent [S1, S2, unofficial].
*   **GPU Texture (OpenGL):** Internal format `GL_RED`, swizzle `(R, R, R, R)`, type `GL_UNSIGNED_BYTE` or `GL_UNSIGNED_SHORT`, filter `GL_LINEAR`, wrap `GL_CLAMP_TO_BORDER` [S2, unofficial].
*   **GPU Texture (CUDA):** `cudaArray` with channel format `cudaCreateChannelDesc<ushort>` (or `uchar`), bound to `cudaTextureObject_t` [S3, unofficial].

**Steps:**

1.  **Cache Check (VTK Path Only):** If the input pointer has not changed, the modification time is unchanged, the cell flag is unchanged, and the requested `textureExtent` is enclosed within the previously loaded `LoadedExtent`, return success immediately without reloading [S1, unofficial].
2.  **Data Extraction:**
    *   *Binary Path:* Open file in binary mode. Determine size via `file.tellg()`. Allocate buffer of exact size. Read entire file into buffer [S2, unofficial].
    *   *VTK Path:* Update `TransformedInput` (or `AdditionalTransformedInput` based on `type`) [S1, unofficial]. Extract scalars using `GetScalars`. Validate that scalars are point or cell data (not field data) [S1, unofficial].
3.  **Validation:** Check extent bounds. If `CellFlag` is true, use `<=` comparison for bounds; if false, use strict `<` comparison [S1, unofficial]. If GPU memory allocation fails, return false [S1, unofficial].
4.  **Texture Creation & Upload:**
    *   *OpenGL:* Create 3D texture with `GL_RED` internal format. Set swizzle mask to `(R, R, R, R)`. Set filtering to `GL_LINEAR` and wrapping to `GL_CLAMP_TO_BORDER` for S, T, R axes [S2, unofficial]. Upload buffer data using `glTexSubImage3D` or `glTexImage3D` [S2, unofficial].
    *   *CUDA:* Create 3D array with `cudaMalloc3DArray` using the appropriate channel descriptor (`ushort` or `uchar`) [S3, unofficial]. Copy data from host to device array using `cudaMemcpy3D` with `cudaMemcpyHostToDevice` [S3, unofficial].
5.  **Binding & Configuration:**
    *   *OpenGL:* Bind texture to target `GL_TEXTURE_3D` [S2, unofficial].
    *   *CUDA:* Set texture parameters: `filterMode = cudaFilterModeLinear`, `addressMode = cudaAddressModeClamp` for all axes [S3, unofficial]. Bind array to texture object using `cudaBindTextureToArray` [S3, unofficial].
6.  **State Update (VTK Path Only):** Update `LoadedExtent`, `LoadedBounds`, `LoadedCellFlag`, and `LoadedTime` with current values [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def load_gpu_texture(source, extent, is_16bit=False, cell_flag=False):
    """
    Loads volumetric data into a GPU 3D texture.
    source: dict with keys 'file_path' (str) or 'vtk_data' (object)
    extent: tuple (min_x, max_x, min_y, max_y, min_z, max_z)
    is_16bit: bool, True for uint16, False for uint8
    cell_flag: bool, True for cell data, False for point data
    """
    # 1. Cache Check (VTK specific)
    if source.get('vtk_data') is not None:
        if is_cached_and_valid(source['vtk_data'], extent, cell_flag):
            return get_cached_texture()

    # 2. Data Extraction
    if 'file_path' in source:
        # Binary loading
        file_size = get_file_size(source['file_path'])
        buffer = read_binary_file(source['file_path'], file_size)
        data_type = GL_UNSIGNED_SHORT if is_16bit else GL_UNSIGNED_BYTE
    else:
        # VTK loading
        vtk_data = source['vtk_data']
        scalars = extract_scalars(vtk_data, cell_flag)
        if scalars is None:
            raise ValueError("No scalars found")
        buffer = scalars_to_buffer(scalars, is_16bit)
        data_type = GL_UNSIGNED_SHORT if is_16bit else GL_UNSIGNED_BYTE

    # 3. Validation
    if not check_extent_bounds(extent, cell_flag):
        raise ValueError("Invalid extent bounds")

    # 4. Texture Creation & Upload
    if use_opengl():
        tex_id = create_opengl_3d_texture(data_type, extent)
        set_opengl_params(tex_id, is_16bit)
        upload_opengl_data(tex_id, buffer, extent)
        texture = bind_opengl_texture(tex_id)
    else:
        # CUDA path
        channel_desc = cuda_create_channel_desc_ushort() if is_16bit else cuda_create_channel_desc_uchar()
        array = cuda_malloc_3d_array(channel_desc, extent)
        copy_params = make_cuda_memcpy3d_params(buffer, array, extent)
        cuda_memcpy_3d(copy_params)
        texture = bind_cuda_texture(array, channel_desc)

    # 5. State Update (VTK specific)
    if source.get('vtk_data') is not None:
        update_cache_state(source['vtk_data'], extent, cell_flag)

    return texture
```

## Subtopics

### [CUDA-accelerated Vector Field Reading with Magnitude](2581-cuda-accelerated-vector-field-reading-with-magnitude.md)
Enables fast, linearly interpolated reading of 3D vector field values at arbitrary positions using CUDA 3D textures. It differs from standard scalar texture loading by storing precomputed vector magnitudes in the fourth component of a `float4` structure to avoid redundant calculations during rendering. Choose this when rendering vector fields where magnitude is frequently accessed. Link: [CUDA-accelerated Vector Field Reading with Magnitude](2581-cuda-accelerated-vector-field-reading-with-magnitude.md).

### [Hardware-Filtered Data Textures](13812-hardware-filtered-data-textures.md)
Enables efficient single-fetch filtered lookups by mapping volume data to specific GPU texture formats that match the sampling kernel, avoiding the high cost of manual interpolation on integer textures. This method differs from general texture loading by prioritizing the internal format's compatibility with hardware filtering units over raw transfer bandwidth, whereas sibling methods like CUDA-accelerated Vector Field Reading focus on vector magnitude computation rather than scalar field sampling optimization. Choose this when manual interpolation is a bottleneck. Link: [Hardware-Filtered Data Textures](13812-hardware-filtered-data-textures.md).

### Data Loading and Texture Binding
Loads 3D volume data into a CUDA 3D array and binds it to a texture object for GPU access [S3, unofficial]. It copies data dimensions to a device symbol, creates a 3D array with ushort channel format, copies data using `cudaMemcpy3D`, sets linear filter and clamp address mode, and binds the array to the texture object [S3, unofficial]. Choose this for simple CUDA-based volume rendering pipelines. Link: [CUDA-accelerated Vector Field Reading with Magnitude](2581-cuda-accelerated-vector-field-reading-with-magnitude.md) (related CUDA context).

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the loaded extent, dominated by memory copy operations [S1, S2, S3, unofficial].
*   **Memory Complexity:** $O(N)$ bytes, where $N$ is the number of voxels, plus overhead for texture object metadata [S1, S2, S3, unofficial].
*   **Performance:** Loading time depends on PCIe bandwidth and GPU VRAM availability. Insufficient VRAM causes allocation failure [S1, unofficial].
*   **Strengths:** Simple API, hardware-accelerated interpolation support, caching reduces redundant loads [S1, S2, unofficial].
*   **Limitations:** Limited to scalar fields (unless using variants), fixed extent loading may waste memory if subextents are small [S1, S2, unofficial].
*   **When to use:** When raw volumetric data needs to be rendered on GPU with linear interpolation.
*   **When not to use:** When data is already in GPU memory, or when tiled storage is required for very large datasets exceeding 3D texture size limits.

## Background
Originated in VTK-based volume rendering pipelines (bozorgi/VTKMultiVolumeRayCaster) and standalone OpenGL/CUDA volume rendering projects (jose-villegas/VolumeRendering, ojaisnielsen/dtu-2010-cuda-volume-rendering) [S1, S2, S3, unofficial]. It is used in medical imaging and scientific visualization where raw DICOM or binary volume data must be transferred to the GPU for raycasting [S2, S3, unofficial].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] jose-villegas/VolumeRendering, https://github.com/jose-villegas/VolumeRendering, fetched 2026-10-27 (unofficial)
[S3] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
