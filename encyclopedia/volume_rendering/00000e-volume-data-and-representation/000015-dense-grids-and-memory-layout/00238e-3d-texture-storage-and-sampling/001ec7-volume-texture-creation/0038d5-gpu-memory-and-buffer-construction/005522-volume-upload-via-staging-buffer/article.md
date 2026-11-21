# Volume Upload via Staging Buffer

Also called: GPU Memory Transfer via Staging, Staging Buffer Upload

## Overview
This method transfers dense 3D volume data from host memory to GPU device-local memory using an intermediate staging buffer. It solves the problem of populating a GPU-resident 3D texture when the underlying hardware does not support direct host-to-device memory mapping or when the data requires preparation (e.g., byte-swapping or format conversion) before upload. It is distinct from [Device Volume Snapshot](2246-device-volume-snapshot.md), which captures existing device data, and [ClearView Unified Buffer Construction](2475-clearview-unified-buffer-construction.md), which may use unified memory architectures.

## How it works
**Inputs:**
- `dims`: A structure containing volume dimensions `nx`, `ny`, `nz`.
- `field`: Host-side array of volume data (typically `float`).
- `kChannels`: Number of channels per voxel (e.g., 1 for scalar, 4 for RGBA).
- `kVolumeFormat`: The GPU texture format (e.g., `R32F`, `RGBA32F`).

**Outputs:**
- A GPU 3D image resource bound to a 3D image view, ready for sampling.

**Data Structures:**
1.  **Volume Dimensions (`dims`)**:
    -   `nx`: Width (uint32)
    -   `ny`: Height (uint32)
    -   `nz`: Depth (uint32)
    -   `count()`: Returns `nx * ny * nz` [S1, unofficial].
2.  **Staging Buffer**:
    -   Host-visible, host-coherent memory allocation.
    -   Size: `byteSize` bytes.
3.  **Device Image**:
    -   Type: `e3D`
    -   Format: `kVolumeFormat`
    -   Extent: `(dims.nx, dims.ny, dims.nz)`
    -   Mip Levels: 1
    -   Array Layers: 1
    -   Samples: `e1`
    -   Tiling: `eOptimal`
    -   Usage Flags: `eSampled | eTransferDst | eTransferSrc` [S1, unofficial].
    -   Initial Layout: `eUndefined` [S1, unofficial].
4.  **Image View**:
    -   View Type: `e3D`
    -   Format: `kVolumeFormat`
    -   Subresource Range: Aspect `eColor`, Base Mip Level 0, Level Count 1, Base Array Layer 0, Layer Count 1 [S1, unofficial].

**Steps:**
1.  **Validate Dimensions**: Check if `dims.nx`, `dims.ny`, or `dims.nz` exceed the device's `limits.maxImageDimension3D`. If any dimension exceeds the limit, return error `Errc::unsupported_configuration` [S1, unofficial].
2.  **Calculate Size**: Compute the required byte size for the staging buffer:
    $$ \text{byteSize} = \text{dims.count()} \times \text{kChannels} \times \text{sizeof(float)} $$
    [S1, unofficial]
3.  **Allocate Resources**:
    -   Allocate device-local memory for the 3D image.
    -   Allocate host-visible, host-coherent memory for the staging buffer [S1, unofficial].
4.  **Populate Staging Buffer**:
    -   Map the staging buffer memory.
    -   Create a `std::span<float>` of size `dims.count() * kChannels`.
    -   Populate the span with the `field` data. This step may involve calling a helper like `deriveField<T>` to handle data transformation and return the `autoRange` [S1, unofficial].
    -   Unmap the staging memory.
5.  **Create GPU Resources**:
    -   Create the 3D image using the creation info defined in **Data Structures**.
    -   Create the 3D image view using the creation info defined in **Data Structures** [S1, unofficial].
6.  **Upload Data**:
    -   Record a command buffer command to copy the data from the staging buffer (`eTransferSrc`) to the 3D image (`eTransferDst`).
    -   Execute the command buffer.
    -   Wait for completion.
7.  **Cleanup**:
    -   Destroy the staging buffer and its memory.
    -   Destroy the image view, image, and their respective memory allocations in the correct order (see Pitfalls) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def upload_volume_to_gpu(field: list[float], dims: Dimensions, k_channels: int, k_volume_format: str):
    # 1. Validate dimensions
    max_dim = get_device_limit("maxImageDimension3D")
    if dims.nx > max_dim or dims.ny > max_dim or dims.nz > max_dim:
        raise Error(Errc::unsupported_configuration, "Volume dimensions exceed device limits")

    # 2. Calculate size
    voxel_count = dims.nx * dims.ny * dims.nz
    byte_size = voxel_count * k_channels * sizeof(float)

    # 3. Allocate staging buffer (Host-visible, Host-coherent)
    staging_buffer = allocate_buffer(size=byte_size, usage=Usage::eTransferSrc, sharing_mode=SharingMode::eExclusive)
    staging_memory = allocate_memory(buffer=staging_buffer, properties=MemoryProperty::eHostVisible | MemoryProperty::eHostCoherent)
    bind_memory(buffer=staging_buffer, memory=staging_memory)

    # 4. Populate staging buffer
    mapped_data = map_memory(staging_memory)
    # Create span and populate data. deriveField handles transformation if needed.
    span = std_span_float(mapped_data, voxel_count * k_channels)
    auto_range = derive_field(span, field) # [S1, unofficial]
    unmap_memory(staging_memory)

    # 5. Allocate device image
    image_info = ImageCreateInfo(
        type=ImageType::e3D,
        format=k_volume_format,
        extent=Extent3D(dims.nx, dims.ny, dims.nz),
        mip_levels=1,
        array_layers=1,
        samples=SampleCount::e1,
        tiling=Tiling::eOptimal,
        usage=Usage::eSampled | Usage::eTransferDst | Usage::eTransferSrc,
        initial_layout=ImageLayout::eUndefined
    )
    gpu_image = create_image(image_info)
    image_memory = allocate_memory(image=gpu_image, properties=MemoryProperty::eDeviceLocal)
    bind_memory(image=gpu_image, memory=image_memory)

    # 6. Create image view
    view_info = ImageViewCreateInfo(
        image=gpu_image,
        view_type=ImageViewType::e3D,
        format=k_volume_format,
        subresource_range=SubresourceRange(
            aspect_mask=ImageAspectFlag::eColor,
            base_mip_level=0,
            level_count=1,
            base_array_layer=0,
            layer_count=1
        )
    )
    image_view = create_image_view(view_info)

    # 7. Upload via command buffer
    command_buffer = begin_command_buffer()
    transition_layout(command_buffer, gpu_image, ImageLayout::eUndefined, ImageLayout::eTransferDstOptimal)
    copy_buffer_to_image(command_buffer, staging_buffer, gpu_image, dims)
    transition_layout(command_buffer, gpu_image, ImageLayout::eTransferDstOptimal, ImageLayout::eShaderReadOnlyOptimal)
    end_command_buffer(command_buffer)
    submit_and_wait(command_buffer)

    # 8. Cleanup staging
    destroy_buffer(staging_buffer)
    free_memory(staging_memory)

    return VolumeResource(image=gpu_image, view=image_view, dims=dims)
```

**Edge Cases and Pitfalls:**
-   **Memory Destruction Order**: Memory must be destroyed in reverse order of binding to avoid freeing memory bound to destroyed objects. The required order is: `imageMem`, `image`, `view`, `stagingMem`, `staging` [S1, unofficial].
-   **Dimension Limits**: Exceeding `maxImageDimension3D` results in an unsupported configuration error [S1, unofficial].
-   **Coherency**: The staging buffer must be host-coherent to ensure written data is visible to the GPU without explicit cache flushes, or explicit flushes must be performed [S1, unofficial].

## Subtopics

### Volume Readback via Staging Buffer
This variant transfers data from the GPU device to the host. It allocates a host-visible, host-coherent staging buffer with usage `eTransferDst` [S1, unofficial]. After rendering or processing, the GPU image is copied to the staging buffer. The host then maps the staging memory, copies `byteSize` bytes into a `std::vector<float>`, unmaps the memory, and returns a `VolumeReadback` struct containing the dimensions and data [S1, unofficial]. The byte size calculation is identical: `byteSize = dims.count() * kChannels * sizeof(float)` [S1, unofficial].

## Operating characteristics
-   **Time Complexity**: Dominated by the PCIe transfer time between host and device. The copy operation is linear with respect to the number of voxels.
-   **Memory Complexity**: Requires additional host memory equal to `byteSize` for the staging buffer.
-   **Performance**: Slower than direct mapping if the hardware supports it, due to the extra copy step. However, it is often necessary for optimal performance because it allows the GPU to use device-local memory (optimized for sampling) while using host-coherent memory for efficient CPU writes.
-   **Strengths**: Works on hardware without direct host-to-device mapping support; ensures optimal GPU memory layout (optimal tiling).
-   **Limitations**: Requires double the memory footprint during upload (host staging + device image); introduces latency due to the copy operation.

## Background
This method is a standard technique in GPU programming, particularly in Vulkan and OpenGL, for managing volume data. It is implemented in libraries such as `imaginaryVolumes` [S1, unofficial]. The approach leverages the Vulkan memory model's separation of host-visible and device-local memory to balance CPU write efficiency and GPU read performance.

## Sources
[S1] steubengineer/imaginaryVolumes, https://github.com/steubengineer/imaginaryVolumes, fetched 2026-10-26 (unofficial)
